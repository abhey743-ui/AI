# Semantic Caching in Spring AI — Concept, Trade-offs, and Two Full Implementations

## 1. What semantic caching actually is

A normal cache matches on **exact keys** — same input, same cache hit. Semantic caching matches on **meaning** instead: if a new question is *semantically similar enough* to a question you've already answered, the cached answer is returned immediately, without calling the model at all — even if the new question is worded completely differently from the original.

Mechanically, it works almost exactly like RAG's retrieval step (same embeddings, same similarity search, same vector store machinery), just pointed at a different problem: instead of retrieving *source documents* relevant to a question, you're retrieving *a previous question-and-answer pair* that's close enough to count as "the same question, worded differently." If a close-enough match is found, its stored answer is returned directly — the actual model is never called for that request.

**The one-sentence version:** semantic caching is "have we essentially already answered this?" checked by embedding similarity, instead of by exact string matching.

---

## 2. Why we use it

### 2.1 Cost reduction
Every model call costs money, billed by tokens. In any real application, a meaningful fraction of incoming questions are **effectively duplicates of each other**, just phrased differently — "How do I cancel my subscription?" and "What's the process for cancelling my plan?" are different strings but the same underlying question. An exact-match cache misses both as a pair entirely; a semantic cache correctly recognizes them as the same question and serves the cached answer for the second one at zero additional model cost.

### 2.2 Latency reduction
A cache hit returning a stored string is dramatically faster than a live round trip to a model provider. For high-traffic, FAQ-heavy applications (support bots, documentation assistants), this can be the difference between a response in milliseconds versus a response in seconds.

### 2.3 Reduced load on rate-limited or quota-constrained APIs
If you're operating under a request-per-minute or token-per-minute limit from a provider, every cache hit is a request that never counts against that limit — directly increasing how much real traffic your application can absorb on the same quota.

### 2.4 Consistency
A semantic cache hit returns the *exact same* previously-generated answer for a repeated question, rather than a freshly generated (and potentially slightly different, given temperature/sampling) response each time — useful anywhere consistent, predictable answers for recurring questions matter more than fresh generation.

---

## 3. Use cases

- **Customer support / FAQ bots** — a large share of real user questions cluster around a small set of actual underlying questions, just phrased differently every time; this is the textbook case semantic caching was built for.
- **Documentation assistants** — "how do I install X," "what's the setup process for X," "steps to get X running" are all the same question, and likely to recur constantly across different users.
- **High-traffic public-facing assistants** — anywhere the same handful of popular questions make up a disproportionate share of total traffic (a classic long-tail distribution), caching the head of that distribution yields outsized savings.
- **Cost-sensitive, budget-capped deployments** — applications where controlling total model spend is a hard requirement, not just a nice-to-have.
- **Internal tools with repetitive queries** — e.g. an internal assistant repeatedly asked variations of "what's our refund policy" by different support agents throughout the day.

---

## 4. Drawbacks and risks — read this before you turn it on everywhere

### 4.1 Stale answers
If the cached answer concerned something that has since changed (a policy update, a price change, a fixed bug), a semantically similar new question can return the **old, now-incorrect** cached answer instead of a fresh one reflecting the current reality. Unlike a vector store feeding fresh retrieval into every call, a semantic cache actively *skips* calling the model (and therefore skips any live, current grounding) the moment it decides there's a match.

### 4.2 False positives from an overly loose similarity threshold
Two questions can embed as "similar" while actually needing different answers — e.g. "How do I cancel my Professional plan?" and "How do I cancel my Enterprise plan?" might be close enough in vector space to false-positive match each other, despite the correct answer genuinely differing between them. A threshold set too low turns this from a rare edge case into a frequent, silently wrong source of answers.

### 4.3 Doesn't help with genuinely novel questions
A semantic cache only pays off for *recurring* questions. An application whose traffic is mostly unique, one-off questions gets little benefit from caching while still paying the infrastructure cost (a running vector store/Redis instance) of maintaining one.

### 4.4 Cache growth and maintenance
Every cache miss is a new entry — without some eviction/expiry strategy, the cache grows indefinitely, including entries that may become stale or simply irrelevant over time. This needs active management (TTLs, periodic invalidation when source data changes), not a "set it and forget it" deployment.

### 4.5 An extra moving part in your infrastructure
A semantic cache (whether Redis-backed or vector-store-backed) is another stateful service to run, monitor, and keep available — if it goes down, you need a defined fallback behavior (fail open and just call the model normally, ideally) rather than the whole application breaking.

### 4.6 Per-user/conversation leakage risk
If a cache isn't correctly scoped, one user's cached answer (which might reference account-specific details) could theoretically be served to a different user asking a similarly-worded but actually-different question — this needs deliberate scoping (e.g. per-user or per-tenant cache namespaces) in anything beyond a purely generic-knowledge assistant.

---

## 5. The architecture — `SemanticCache` and `SemanticCacheAdvisor`

Two pieces, following the exact same "interface + advisor" pattern already used for chat memory and RAG elsewhere in this series:

- **`SemanticCache`** (`org.springframework.ai.vectorstore.redis.cache.semantic.SemanticCache`) — the storage/lookup contract: given a query, check whether a similar-enough cached entry exists; if a new question/answer pair needs caching, store it. `DefaultSemanticCache` is the ready-made implementation.
- **`SemanticCacheAdvisor`** (`org.springframework.ai.chat.cache.semantic.SemanticCacheAdvisor`) — the `CallAdvisor` that actually wires a `SemanticCache` into the `ChatClient` request pipeline, following exactly the "logic wrapped around `chain.nextCall(...)`" shape from the advisors guide.

### 5.1 The request flow, step by step

```
User question
   │
   ▼
SemanticCacheAdvisor intercepts the call (before chain.nextCall)
   │
   ▼
Embed the question, similarity-search the SemanticCache's backing store
   │
   ├── Similar-enough cached entry found (≥ similarityThreshold)?
   │        │
   │       YES → return the cached answer immediately.
   │             chain.nextCall(...) is NEVER invoked — the model is not called at all.
   │
   │       NO  → chain.nextCall(...) proceeds normally
   │              → the actual model call happens
   │              → once the real response comes back, SemanticCacheAdvisor
   │                stores (question, answer) as a new entry in the cache,
   │                so a future similar question becomes a hit
   │
   ▼
Response returned to the caller (either the cached one, or freshly generated)
```

This is precisely the `CallAdvisor` "before and after `chain.nextCall(...)`" mechanism from the advisors guide, applied to caching specifically: the "before" half does the similarity lookup and can short-circuit the whole chain on a hit; the "after" half (when there was a miss) writes the fresh answer back into the cache for next time.

### 5.2 Why this lives as an *advisor*, not something you call manually

Exactly the same reasoning as every other advisor in this series: caching is a cross-cutting concern that should apply uniformly across every call through a given `ChatClient`, without every service method needing to remember to check-then-store by hand. Registering `SemanticCacheAdvisor` once (at the bean level, or per call) means every request through that client automatically benefits from caching — including requests from code you haven't written yet.

---

## 6. Implementation 1 — Redis, via a direct Jedis client connection

This is your first configuration, corrected and completed (your original was missing the `@Bean` annotation on the `semanticCacheAdvisor` method — without it, Spring never registers that bean at all, so nothing downstream could actually inject a `SemanticCacheAdvisor`):

```java
package com.EazyBytesSpringAi.EazyBytesSpringAi.Rag;

import org.springframework.ai.chat.cache.semantic.SemanticCacheAdvisor;
import org.springframework.ai.embedding.EmbeddingModel;
import org.springframework.ai.vectorstore.redis.cache.semantic.DefaultSemanticCache;
import org.springframework.ai.vectorstore.redis.cache.semantic.SemanticCache;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import redis.clients.jedis.JedisPooled;

@Configuration
public class RedisCacheConfiguration {

    @Bean
    public JedisPooled jedisClient() {
        return new JedisPooled("localhost", 6379);
    }

    @Bean
    public SemanticCache semanticCache(JedisPooled jedisClient, EmbeddingModel embeddingModel) {
        return DefaultSemanticCache.builder()
                .similarityThreshold(0.8)
                .jedisClient(jedisClient)
                .embeddingModel(embeddingModel)
                .indexName("eazybytes-semantic-cache")
                .prefix("cache:")
                .build();
    }

    @Bean
    public SemanticCacheAdvisor semanticCacheAdvisor(SemanticCache semanticCache) {
        return SemanticCacheAdvisor.builder()
                .cache(semanticCache)
                .build();
    }
}
```

**What changed from your version, and why:**
- **`JedisPooled` instead of a custom `RedisClient`** — Spring AI's Redis semantic cache integration is built against Jedis's own client types (`JedisPooled`), not a hand-rolled `RedisClient` wrapper — this is the actual type `DefaultSemanticCache.builder().jedisClient(...)` expects.
- **Added `.embeddingModel(embeddingModel)`** on the builder — the cache needs an `EmbeddingModel` to actually embed incoming queries before it can run a similarity search against what's stored; this was referenced as a constructor parameter in your bean method but never actually passed into the builder.
- **Added the missing `@Bean`** on `semanticCacheAdvisor(...)` — without it, Spring treats that method as a plain helper method, not a bean definition, so `SemanticCacheAdvisor` would never actually exist in the application context to be injected anywhere.

**Dependencies needed:**

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-vector-store-redis-semantic-cache</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-openai</artifactId> <!-- or whichever model provides your EmbeddingModel -->
</dependency>
```

With this starter, Spring Boot can also auto-configure the semantic cache entirely from `application.yml`, as an alternative to the manual `@Configuration` class above:

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
    vectorstore:
      redis:
        semantic-cache:
          enabled: true
          host: localhost
          port: 6379
          index-name: eazybytes-semantic-cache
```

---

## 7. Implementation 2 — backed by any `VectorStore` (your Qdrant example)

```java
package com.EazyBytesSpringAi.EazyBytesSpringAi.Rag;

import io.qdrant.client.QdrantClient;
import org.springframework.ai.chat.cache.semantic.SemanticCacheAdvisor;
import org.springframework.ai.embedding.EmbeddingModel;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.ai.vectorstore.qdrant.QdrantVectorStore;
import org.springframework.ai.vectorstore.redis.cache.semantic.DefaultSemanticCache;
import org.springframework.ai.vectorstore.redis.cache.semantic.SemanticCache;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class VectorStoreCaching {

    @Bean
    public VectorStore cacheVectorStore(QdrantClient qdrantClient, EmbeddingModel embeddingModel) {
        return QdrantVectorStore.builder(qdrantClient, embeddingModel)
                .initializeSchema(true)
                .collectionName("semantic-cache")
                .build();
    }

    @Bean
    public SemanticCache semanticCache(@Qualifier("cacheVectorStore") VectorStore vectorStore,
                                        EmbeddingModel embeddingModel) {
        return DefaultSemanticCache.builder()
                .similarityThreshold(0.8)
                .vectorStore(vectorStore)
                .embeddingModel(embeddingModel)
                .build();
    }

    @Bean
    public SemanticCacheAdvisor semanticCacheAdvisor(SemanticCache semanticCache) {
        return SemanticCacheAdvisor.builder()
                .cache(semanticCache)
                .build();
    }
}
```

**What changed from your version:** `com.openai.models.embeddings.EmbeddingModel` was replaced with Spring AI's own `org.springframework.ai.embedding.EmbeddingModel` — the OpenAI Java SDK's own `EmbeddingModel` type isn't the type Spring AI's builders expect; Spring AI's portable `EmbeddingModel` interface is what every `VectorStore`/`SemanticCache` builder in this framework is written against, exactly like `ChatModel` elsewhere in this series. `.embeddingModel(embeddingModel)` was added for the same reason as Implementation 1 — the cache needs it to embed incoming queries.

**The key architectural point:** `DefaultSemanticCache.builder()` supports **two different backing strategies** — `.jedisClient(...)` for a direct, Redis-native connection (Implementation 1), or `.vectorStore(...)` for *any* `VectorStore` implementation (Implementation 2) — meaning the exact same semantic-caching logic can sit on top of Qdrant, or any other supported vector store, not just Redis specifically. This mirrors precisely how `RetrievalAugmentationAdvisor`'s `DocumentRetriever` could be backed by either a vector store or a live web search in the earlier guides — the caching *behavior* is decoupled from *where the vectors actually live*.

**Dependencies needed:**

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-redis-semantic-cache</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-vector-store-qdrant</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-openai</artifactId>
</dependency>
```

Note: here you need the lower-level `spring-ai-redis-semantic-cache` artifact (for the `SemanticCache`/`DefaultSemanticCache`/`SemanticCacheAdvisor` classes themselves) rather than the full `-starter-...-redis-semantic-cache` auto-configuration bundle from Implementation 1 — since you're manually wiring a Qdrant-backed `VectorStore` into the cache yourself rather than letting Redis-specific auto-configuration handle everything from `application.yml`.

---

## 8. Class-level and advisor-level interaction — how it all actually connects

```
┌──────────────────────────┐
│   SemanticCache (interface)│  — contract: check for a similar cached entry, store a new one
└─────────────┬────────────┘
              │ implemented by
              ▼
┌──────────────────────────┐
│    DefaultSemanticCache    │  — backed by EITHER:
│                            │      .jedisClient(...)  → direct Redis connection
│                            │      .vectorStore(...)  → any VectorStore (Qdrant, etc.)
│                            │    uses the injected EmbeddingModel to embed queries
└─────────────┬────────────┘
              │ wrapped by
              ▼
┌──────────────────────────┐
│    SemanticCacheAdvisor    │  — a CallAdvisor; registered on ChatClient
│                            │    like any other advisor (bean-level default or per-call)
└─────────────┬────────────┘
              │ sits in the advisor chain, intercepting
              ▼
┌──────────────────────────┐
│         ChatClient          │  — your application code calls this normally;
│                            │    caching is entirely transparent to the caller
└──────────────────────────┘
```

Registering it, exactly like every other advisor covered so far:

```java
@Bean
ChatClient chatClient(ChatClient.Builder builder, SemanticCacheAdvisor semanticCacheAdvisor) {
    return builder
            .defaultAdvisors(semanticCacheAdvisor) // applies to every call through this client
            .build();
}
```

```java
@Service
class SupportService {

    private final ChatClient chatClient;

    SupportService(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    public String answer(String question) {
        // No cache-specific code here at all — SemanticCacheAdvisor
        // transparently intercepts this call, checks for a similar
        // cached question, and either returns the cached answer or
        // lets the call proceed to the model and caches the fresh result.
        return chatClient.prompt()
                .user(question)
                .call()
                .content();
    }
}
```

Nothing in the service method changes based on whether caching is in play — this is the entire point of the advisor pattern: the service code stays identical whether a `SemanticCacheAdvisor` is registered or not, exactly as it did for memory and RAG advisors earlier in this series.

---

## 9. `similarityThreshold` — the one setting that matters most

Both implementations above set `.similarityThreshold(0.8)` — this single number is the main lever controlling the trade-off from §4.2: how similar does a new question need to be to a cached one before it's considered "the same question"?

- **Too high (closer to 1.0, near-identical required)** — fewer false-positive wrong answers, but also far fewer actual cache hits, since even minor rewording of the same real question might fall just short of the threshold — defeating much of the point of using *semantic* caching over exact-match caching in the first place.
- **Too low** — many more cache hits, but a real risk of two genuinely different questions (as in the subscription-plan example in §4.2) being treated as the same, silently returning a wrong cached answer.

There's no universal correct value — it depends on how semantically "close together" your application's genuinely-different questions tend to be, and is worth tuning empirically against real traffic rather than guessed once and left alone.

---

## 10. Recap

Semantic caching checks "have we essentially already answered something like this?" via embedding similarity, and when the answer is yes, skips the model call entirely and returns the stored answer — the same similarity-search machinery as RAG retrieval, pointed at past Q&A pairs instead of source documents. It meaningfully cuts cost and latency for applications with genuinely repetitive traffic, at the real cost of potential staleness and false-positive mismatches if `similarityThreshold` isn't tuned carefully. `SemanticCache`/`DefaultSemanticCache` is the storage contract (backed by either a direct Redis/Jedis connection or any `VectorStore`), and `SemanticCacheAdvisor` is what actually wires that cache into the `ChatClient` request pipeline as a transparent, before-and-after-`chain.nextCall(...)` advisor — meaning, just like every other advisor in this series, your service code never has to know caching is happening at all.
