# Implementing RAG in Spring AI — Three Methods, From Manual to Advanced

This covers the three ways to actually wire RAG into a Spring AI application, from the fully manual approach up to the modular `RetrievalAugmentationAdvisor` — the class, architecture, and request flow behind each one, using your own three methods as the walkthrough.

---

## 1. Method 1 — Fully manual RAG (`manualRagSystemSimilarSearch`)

### 1.1 What this is, architecturally

No advisor involved at all. Every single RAG step — search, build context, inject it into the system prompt, call the model — is written out explicitly, by hand, inside the service method. This is the "ground truth" version of RAG: understanding this fully is what makes the advisor-based methods in §2 and §3 make sense, because they're automating *exactly these same steps*.

### 1.2 Your code, walked through step by step

```java
public String manualRagSystemSimilarSearch(String query) {

    SearchRequest searchRequest = SearchRequest.builder()
            .query(query)
            .similarityThreshold(0.6)
            .topK(10)
            .build();

    List<Document> docs = vectorStore.similaritySearch(searchRequest);

    List<String> data = docs.stream().map(Document::getText).toList();
    String contextData = String.join("\n\n", data);

    SystemPromptTemplate systemPromptTemplate = new SystemPromptTemplate(resource);
    Message message = systemPromptTemplate.createMessage(Map.of("context", contextData));

    return chatClient.prompt(query)
            .advisors(ad -> ad.param(ChatMemory.CONVERSATION_ID, "abhay"))
            .system(message.getText())
            .call()
            .content();
}
```

- **`SearchRequest.builder()...build()`** — builds the retrieval query explicitly: the raw `query` text, a `similarityThreshold(0.6)` (discard anything less than 60% similar — this is the guardrail from the RAG theory guide against stuffing in weakly-related, low-confidence matches), and `topK(10)` (retrieve up to 10 candidate chunks).
- **`vectorStore.similaritySearch(searchRequest)`** — the actual retrieval step (Part 1 of the vector store guide) — returns the matching `Document`s directly from Qdrant (or whichever store is configured).
- **`docs.stream().map(Document::getText).toList()`** then **`String.join("\n\n", data)`** — this *is* the manual version of a `DocumentJoiner` (covered properly in §3.5): taking several separate retrieved chunks and concatenating them into one combined context block, with a blank line between each.
- **`new SystemPromptTemplate(resource)`** + **`.createMessage(Map.of("context", contextData))`** — exactly the templating pattern from the prompt-templating guide: a `.st` resource file somewhere in your template has a `{context}` placeholder, and this line substitutes in the retrieved, joined text.
- **`chatClient.prompt(query)...system(message.getText())...call().content()`** — finally sends the request, with the retrieved context baked into the system message and the original question as the user message. The `.advisors(ad -> ad.param(ChatMemory.CONVERSATION_ID, "abhay"))` call is unrelated to RAG itself — it's just passing a (hardcoded, here) conversation ID to whatever memory advisor is registered as a bean-level default, per the chat memory guide.

### 1.3 The flow mechanism, end to end

```
User question
   → similaritySearch(query, topK=10, threshold=0.6) against VectorStore
   → retrieved Documents → extract .getText() from each → join with "\n\n"
   → combined context substituted into a SystemPromptTemplate via {context}
   → ChatClient call: system = templated context, user = original question
   → model's grounded answer returned
```

### 1.4 Why you'd still choose this, knowing advisors exist

- **Total control, no magic.** Every step is visible, inspectable, and independently testable — you can log the retrieved docs, tweak the join logic, or swap the template without reaching into an advisor's internals.
- **Easiest to debug when something goes wrong**, precisely because nothing is hidden behind a builder — if retrieval looks wrong, you can `System.out.println(docs)` right there.
- **The real cost:** this exact 15-line block has to be copy-pasted (or at best refactored into a shared helper) into every method that needs RAG — none of it is reusable as a drop-in piece of an advisor chain, and it doesn't compose with other advisors (logging, guardrails) the way a registered `Advisor` naturally does.

---

## 2. Method 2 — `QuestionAnswerAdvisor` (`ragUsingQAndAAdvisor`)

### 2.1 What this is, architecturally

`QuestionAnswerAdvisor` is Spring AI's **ready-made, single-advisor implementation of exactly the "naive RAG" flow from Method 1** — similarity search, then stuff the retrieved context into the prompt — packaged as one `CallAdvisor`/`StreamAdvisor` you register like any other advisor from the advisors guide. It doesn't give you the fine-grained module-by-module control that `RetrievalAugmentationAdvisor` does (§3) — it's intentionally the simple, single-step option.

**Dependency required:**

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-advisors-vector-store</artifactId>
</dependency>
```

### 2.2 Your code

```java
public String ragUsingQAndAAdvisor(String query) {
    return chatClient.prompt(query)
            .advisors(QuestionAnswerAdvisor.builder(vectorStore).build())
            .advisors(ad -> ad.param(ChatMemory.CONVERSATION_ID, "abhay"))
            .call()
            .content();
}
```

- **`QuestionAnswerAdvisor.builder(vectorStore).build()`** — constructs the advisor around your `VectorStore`. Internally, this advisor does, automatically, on every call: run a similarity search using the user's query, join the retrieved documents, and inject them into the prompt as context — the *entire* Method 1 pipeline, minus the explicit code.
- The second `.advisors(ad -> ...)` call passes the memory conversation ID param, same as before — unrelated to the RAG advisor itself, both advisors are simply present in the chain for this one call.
- You can tune it, e.g. passing a similarity threshold/topK/filter the same way `SearchRequest` did manually — commonly via `.advisors(a -> a.param(QuestionAnswerAdvisor.FILTER_EXPRESSION, "type == 'Spring'"))` for metadata filtering at call time, mirroring the `SearchRequest.filterExpression(...)` idea from Method 1 but expressed as an advisor param instead.

### 2.3 The flow mechanism

```
User question
   → QuestionAnswerAdvisor intercepts the call (before chain.nextCall)
   → runs similaritySearch against the configured VectorStore internally
   → joins retrieved documents, injects them into the prompt as context
   → chain.nextCall(...) continues to the model with the augmented prompt
   → model's grounded answer flows back out
```

This is structurally identical to Method 1's flow — the difference is entirely *where* the logic lives: inside a reusable, pre-built `Advisor` object instead of inline in your service method.

### 2.4 When this is the right choice

Exactly the common case: one vector store, similarity search is the only retrieval step you need, and you want RAG behavior without maintaining the manual plumbing from Method 1 — this is genuinely "RAG in two lines," and for a large share of real applications, it's enough.

---

## 3. Method 3 — `RetrievalAugmentationAdvisor` — the advanced, modular flow (`advancedRagSystem`)

### 3.1 What this is, architecturally — the key conceptual shift

`QuestionAnswerAdvisor` (§2) bundles retrieval and context-injection into one fixed, non-customizable step. `RetrievalAugmentationAdvisor` instead implements **Spring AI's Modular RAG architecture** — explicitly inspired by the "Modular RAG: Transforming RAG Systems into LEGO-like Reconfigurable Frameworks" paper — where the RAG pipeline is broken into **distinct, independently swappable stages**, each represented by its own interface. You assemble the pipeline you actually need out of these pieces, rather than accepting one fixed flow.

**Dependency required:**

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-rag</artifactId>
</dependency>
```

### 3.2 The stages, conceptually, in the order they run

```
Pre-retrieval                Retrieval                     Post-retrieval / Generation
┌────────────────┐    ┌──────────────────┐    ┌───────────────┐    ┌───────────────────┐
│ QueryTransformer │ →  │  QueryExpander    │ →  │ DocumentRetriever│ →  │  DocumentJoiner    │ → QueryAugmenter → model
│ (rewrite/clean   │    │ (one query →      │    │ (vector search    │    │ (merge retrieved    │
│  the raw query)  │    │  several queries) │    │  per query)        │    │  docs into context) │
└────────────────┘    └──────────────────┘    └───────────────┘    └───────────────────┘
```

- **Pre-retrieval** — improve the query itself before it's ever used to search anything, because a raw user question is often ambiguous, verbose, or phrased in a way that doesn't match how your knowledge base's text is written.
- **Retrieval** — actually go fetch candidate documents, potentially for *multiple* queries at once if expansion produced more than one.
- **Post-retrieval / generation** — merge whatever was retrieved into one coherent context block, then weave that context into the final prompt sent to the model.

### 3.3 Your code, module by module

```java
public String advancedRagSystem(String query) {

    RetrievalAugmentationAdvisor advisor = RetrievalAugmentationAdvisor.builder()
            .queryTransformers(RewriteQueryTransformer.builder()
                    .chatClientBuilder(chatClient.mutate().clone()).build())
            .queryExpander(MultiQueryExpander.builder()
                    .chatClientBuilder(chatClient.mutate().clone()).build())
            .documentRetriever(VectorStoreDocumentRetriever.builder()
                    .vectorStore(vectorStore).build())
            .documentJoiner(new ConcatenationDocumentJoiner())
            .queryAugmenter(ContextualQueryAugmenter.builder()
                    .allowEmptyContext(true).build())
            .build();

    return chatClient.prompt(query)
            .advisors(advisor)
            .call()
            .content();
}
```

#### `RewriteQueryTransformer` — a `QueryTransformer`

A `QueryTransformer`'s whole job is to take the user's raw query and hand back a (possibly different) query better suited for retrieval. `RewriteQueryTransformer` specifically uses a model call to **rewrite** the original query into a cleaner, more specific, more search-friendly version — e.g. turning a vague, pronoun-heavy follow-up question into a fully self-contained one, or tightening an overly verbose question down to its actual information need. It needs a `ChatClient.Builder` (hence `.chatClientBuilder(chatClient.mutate().clone())`) because the rewriting itself is done *by calling a model* — this is a model call that happens **before** retrieval, distinct from the final model call that generates the user-facing answer.

*(Other `QueryTransformer` implementations exist for different pre-retrieval needs — e.g. `TranslationQueryTransformer` for translating a query into the language your knowledge base is written in, or `CompressionQueryTransformer` for condensing a long multi-turn conversation history down into a single, focused search query — `RewriteQueryTransformer` is just the one you chose here.)*

#### `MultiQueryExpander` — a `QueryExpander`

Where a `QueryTransformer` turns one query into one (improved) query, a `QueryExpander`'s job is to turn **one query into several**. `MultiQueryExpander` uses a model call to generate multiple *semantically different variations* of the same underlying question — different phrasings, different angles, different emphasis — because a single embedding of a single phrasing of a question can miss relevant chunks that would have matched a *differently worded* version of the same underlying intent. By default, `MultiQueryExpander` includes the original query itself alongside the generated variants, so you end up retrieving against several queries at once rather than just one.

#### `VectorStoreDocumentRetriever` — the `DocumentRetriever`

This is the actual retrieval step — conceptually the same `vectorStore.similaritySearch(...)` call from Method 1, but now wrapped as a reusable `DocumentRetriever` component, and run **once per query** produced by the expansion step above (so if `MultiQueryExpander` produced 4 query variants, this retriever runs 4 separate similarity searches, one per variant, under the hood). It also accepts a `FilterExpression`, the same metadata-filtering idea seen in the earlier vector store guide — and can be passed at call time via `.advisors(a -> a.param(VectorStoreDocumentRetriever.FILTER_EXPRESSION, "..."))`, exactly like `QuestionAnswerAdvisor`'s filter param in §2.

#### `ConcatenationDocumentJoiner` — the `DocumentJoiner`

Since multiple queries can each retrieve their own (overlapping, duplicate-prone) set of documents, something has to merge all those results back down into one single, de-duplicated context block before it's usable. `ConcatenationDocumentJoiner` is the straightforward implementation of that — it combines the retrieved documents from every query into one list (this is, again, the automated equivalent of your manual `String.join("\n\n", data)` step from Method 1, just operating across potentially several retrieval passes instead of one).

#### `ContextualQueryAugmenter` — the `QueryAugmenter`

The final step: take the joined context and the (original or transformed) query, and actually build the augmented prompt that goes to the model — conceptually the same job your manual `SystemPromptTemplate` + `.system(message.getText())` step did in Method 1. `.allowEmptyContext(true)` is a specific, important setting: by default, if retrieval comes back with *nothing* relevant, a `QueryAugmenter` can be configured to refuse to proceed (since answering with zero grounding defeats the point of RAG) — setting `allowEmptyContext(true)` instead tells it to proceed anyway, letting the model answer from its own general knowledge rather than failing outright when the knowledge base simply doesn't have anything relevant to this particular question. This is a genuine trade-off: `true` keeps the assistant generally helpful even outside its knowledge base's coverage, at the cost of occasionally answering ungrounded; `false` keeps every answer strictly grounded, at the cost of a hard failure/refusal whenever retrieval comes up empty.

### 3.4 The full flow mechanism for your `advancedRagSystem` method

```
User question ("query")
   │
   ▼
RewriteQueryTransformer    — a model call rewrites/cleans up the query for better retrieval
   │
   ▼
MultiQueryExpander         — a model call expands the (rewritten) query into several variants
   │                          (original query included by default)
   ▼
VectorStoreDocumentRetriever — runs a similarity search against vectorStore for EACH query variant
   │
   ▼
ConcatenationDocumentJoiner — merges/de-duplicates all the retrieved documents into one list
   │
   ▼
ContextualQueryAugmenter   — builds the final augmented prompt from the joined context
   │                          (proceeds even with no context, since allowEmptyContext(true))
   ▼
chain.nextCall(...)        — the actual model call, now with a heavily pre-processed,
                               multi-query-retrieved, context-augmented prompt
   │
   ▼
Grounded answer returned
```

Notice this involves **at least three separate model calls** for a single user question: one for the query rewrite, one (or more, depending on how many variants are requested) for query expansion, and finally one for the actual answer generation. This is the direct cost of the extra retrieval quality — more latency, more token spend, in exchange for meaningfully better odds of retrieving the right chunks for ambiguous or awkwardly-phrased questions.

### 3.5 Why go through all of this instead of just `QuestionAnswerAdvisor`

- **A single raw user query is often a poor retrieval query.** Users ask vague, pronoun-laden, overly casual, or overly long questions — none of which necessarily embeds close to how the *answer* is actually phrased in your source documents. Rewriting and expanding the query directly attacks this mismatch.
- **Multiple phrasings cast a wider net.** `MultiQueryExpander` increases the odds that *at least one* of the generated variants embeds close to the relevant chunk, even if the user's original exact wording wouldn't have.
- **Every stage is independently swappable.** Don't need query expansion for your use case? Simply omit `.queryExpander(...)` from the builder (it's `@Nullable`). Need translation instead of rewriting? Swap `RewriteQueryTransformer` for `TranslationQueryTransformer`. This modularity is the entire point — you compose exactly the pipeline your application needs, rather than being stuck with one fixed shape.
- **The trade-off is real, though:** more model calls per user question means more latency and more cost than Method 2's single-step flow — this is squarely a "more accurate retrieval, slower and pricier per request" trade, worth reserving for cases where retrieval quality genuinely struggles with raw queries, rather than applying by default to every single endpoint in an application.

---

## 4. Comparing all three methods side by side

| | Method 1 — Manual | Method 2 — `QuestionAnswerAdvisor` | Method 3 — `RetrievalAugmentationAdvisor` |
|---|---|---|---|
| **Control** | Full, explicit, line-by-line | Low — fixed internal flow | High — pick and configure each module |
| **Code reuse across methods** | None, unless manually refactored | High — one registered `Advisor` | High — build once per desired pipeline shape, reuse the `Advisor` |
| **Model calls per user question** | 1 (just the final answer) | 1 (just the final answer) | 3+ (query rewrite, query expansion, final answer) |
| **Retrieval quality on ambiguous/awkward queries** | As good as the raw query allows | As good as the raw query allows | Meaningfully better — rewriting/expansion compensates for a weak raw query |
| **Best for** | Learning/understanding RAG internals; cases needing custom logic no advisor supports | The common case — one vector store, straightforward questions | Production systems where retrieval quality on real, messy user queries genuinely matters |
| **Dependency** | Just `VectorStore` + `ChatClient` | `spring-ai-advisors-vector-store` | `spring-ai-rag` |

## 5. Dependencies — exactly what each method needs

All three methods assume you already have the two foundational pieces from the earlier guides in place — a model starter (for the `ChatClient`/`ChatModel` and for whatever embedding model your vector store uses) and a vector store starter (Qdrant, in the earlier example):

```xml
<!-- Always required, regardless of which RAG method you use -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-openai</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-vector-store-qdrant</artifactId>
</dependency>
```

On top of that shared baseline, here's exactly what each of the three methods additionally needs:

### Method 1 — Manual RAG

**No extra dependency at all.** Everything used — `VectorStore`, `SearchRequest`, `Document`, `SystemPromptTemplate`, `ChatClient` — already ships as part of the core `spring-ai-model`/`spring-ai-client-chat` artifacts that the model and vector store starters above already pull in transitively. This is, in fact, one of the quiet advantages of the manual approach: it introduces zero new libraries to your classpath.

### Method 2 — `QuestionAnswerAdvisor`

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-advisors-vector-store</artifactId>
</dependency>
```

This single artifact brings in `QuestionAnswerAdvisor` (and, alongside it, `VectorStoreChatMemoryAdvisor`, a related advisor for storing chat memory itself inside a vector store rather than the `ChatMemory`/`ChatMemoryRepository` system from the memory guide — a different, less common memory strategy worth knowing exists, but outside this guide's scope).

### Method 3 — `RetrievalAugmentationAdvisor`

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-rag</artifactId>
</dependency>
```

This one artifact is where the entire modular RAG toolkit actually lives — `RetrievalAugmentationAdvisor` itself, plus every module used to build a pipeline with it: `QueryTransformer` and its implementations (`RewriteQueryTransformer`, `TranslationQueryTransformer`, `CompressionQueryTransformer`), `QueryExpander`/`MultiQueryExpander`, `DocumentRetriever`/`VectorStoreDocumentRetriever`, `DocumentJoiner`/`ConcatenationDocumentJoiner`, and `QueryAugmenter`/`ContextualQueryAugmenter`. You don't need separate dependencies per module — this single artifact is the complete modular RAG building-block library.

### 5.1 Full `pom.xml` snippet covering all three methods at once

Since all three methods in this guide can coexist in the same service class (as yours do), here's the complete dependency block that supports running all three side by side:

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-bom</artifactId>
            <version>1.0.3</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <!-- Base: model + vector store (required by all three methods) -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-starter-model-openai</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-starter-vector-store-qdrant</artifactId>
    </dependency>

    <!-- Method 2: QuestionAnswerAdvisor -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-advisors-vector-store</artifactId>
    </dependency>

    <!-- Method 3: RetrievalAugmentationAdvisor and its modules -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-rag</artifactId>
    </dependency>
</dependencies>
```

Gradle equivalent, for the two RAG-specific artifacts:

```groovy
implementation 'org.springframework.ai:spring-ai-advisors-vector-store'
implementation 'org.springframework.ai:spring-ai-rag'
```

---

## 6. Recap

All three methods are, underneath, doing the same conceptual RAG job — retrieve relevant context, inject it into the prompt, generate a grounded answer — just at different levels of abstraction and sophistication. Method 1 shows you exactly what's happening, by hand. Method 2 packages that exact same flow into one reusable advisor for the common case. Method 3 breaks that flow into independently configurable modules — query transformation, query expansion, retrieval, joining, and augmentation — letting you build a pipeline specifically tuned for retrieval quality on real-world, imperfectly-phrased questions, at the cost of extra model calls per request. Choosing between them is really a question of how much raw-query imperfection your real users actually produce, and how much that's currently hurting your retrieval quality — not a strict "always use the most advanced one" default.
