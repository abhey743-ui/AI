# Web-Search RAG in Spring AI — Retrieving Live Data Instead of a Vector Store

## 1. The idea — RAG doesn't require a vector store at all

Every RAG method covered so far (manual, `QuestionAnswerAdvisor`, `RetrievalAugmentationAdvisor`) retrieved context from one place: a **vector store**, holding documents you indexed ahead of time. But recall from the advanced-RAG guide that `RetrievalAugmentationAdvisor` treats retrieval as a **pluggable module** — `DocumentRetriever` is just an interface:

```java
public interface DocumentRetriever extends Function<Query, List<Document>> {
    List<Document> retrieve(Query query);
}
```

Nothing in that contract says the documents have to come from a vector database. **Anything that can take a query and hand back a `List<Document>` is a valid `DocumentRetriever`** — a database lookup, an internal REST API, a filesystem scan, or — the case here — a **live web search API**. This is exactly what your `WebSearchDocumentRetriever` is: a custom implementation that retrieves *current, real-time* information from the open web (via Tavily, a search API built specifically for feeding LLMs relevant, cleaned-up web results) instead of a pre-indexed knowledge base.

**Why this matters:** a vector store only knows what you put into it, whenever you last ran ingestion. It's perfect for your own static or slowly-changing data (product docs, policies, internal knowledge). It's structurally the wrong tool for "what's today's weather," "who won the game last night," or "what's the latest version of X" — a vector store simply has no way to know something that happened after your last ingestion run, no matter how well it's chunked or tuned. Web-search RAG fixes exactly this gap: retrieval happens **live**, against the actual current web, at the moment the question is asked.

---

## 2. Walking through your `WebSearchDocumentRetriever`

### 2.1 The class declaration

```java
public class WebSearchDocumentRetriever implements DocumentRetriever {
```

That's the entire integration point — implement one interface, one method, and this class becomes a drop-in replacement for `VectorStoreDocumentRetriever` anywhere a `DocumentRetriever` is expected, most importantly inside a `RetrievalAugmentationAdvisor`.

### 2.2 Construction — wiring up the HTTP client

```java
private static final String TAVILY_BASE_URL = "https://api.tavily.com/search";
private static final int DEFAULT_RESULT_LIMIT = 5;
private final int resultLimit;
private final RestClient restClient;

public WebSearchDocumentRetriever(RestClient.Builder clientBuilder, int resultLimit) {
    Assert.notNull(clientBuilder, "clientBuilder cannot be null");
    this.restClient = clientBuilder
            .baseUrl(TAVILY_BASE_URL)
            .defaultHeader(HttpHeaders.AUTHORIZATION, "Bearer " + TAVILY_API_KEY)
            .build();
    if (resultLimit <= 0) {
        throw new IllegalArgumentException("resultLimit must be greater than 0");
    }
    this.resultLimit = resultLimit;
}
```

This is plain Spring `RestClient` usage (Spring's modern synchronous HTTP client) — nothing RAG-specific here. A `RestClient` is built once, pre-configured with Tavily's base URL and the bearer token it needs for authentication, and `resultLimit` caps how many search hits get pulled back per query. `Assert.notNull`/the manual `IllegalArgumentException` check are defensive validation on construction — fail fast with a clear message rather than a confusing `NullPointerException` deep inside `retrieve(...)` later.

### 2.3 The actual retrieval — `retrieve(Query query)`

```java
@Override
public List<Document> retrieve(Query query) {
    logger.info("Processing query: {}", query.text());
    Assert.notNull(query, "query cannot be null");

    String q = query.text();
    Assert.hasText(q, "query.text() cannot be empty");

    TavilyResponsePayload response = restClient.post()
            .body(new TavilyRequestPayload(q, "advanced", resultLimit))
            .retrieve()
            .body(TavilyResponsePayload.class);

    if (response == null || CollectionUtils.isEmpty(response.results())) {
        return List.of();
    }

    List<Document> docs = new ArrayList<>(response.results().size());
    for (TavilyResponsePayload.Hit hit : response.results()) {
        Document doc = Document.builder()
                .text(hit.content())
                .metadata("title", hit.title())
                .metadata("url", hit.url())
                .score(hit.score())
                .build();
        docs.add(doc);
    }
    return docs;
}
```

This is the method `RetrievalAugmentationAdvisor` (or anything else holding a `DocumentRetriever`) actually calls, once per query, exactly where `VectorStoreDocumentRetriever` would have called `vectorStore.similaritySearch(...)` instead. Step by step:

1. **Pull the plain text out of the `Query` object** — Spring AI's `Query` wraps more than just text (it can carry conversation history and context too), but here only `query.text()` is needed to build the search request.
2. **POST to Tavily's search endpoint**, sending the query text, a `"advanced"` search depth (Tavily's own tuning knob for how thorough its search is), and the configured result cap — `RestClient` serializes `TavilyRequestPayload` to JSON and deserializes the JSON response straight into `TavilyResponsePayload` automatically.
3. **Handle an empty/missing result gracefully** — returning `List.of()` rather than throwing, so a web search that genuinely finds nothing doesn't crash the whole RAG pipeline; downstream, this behaves exactly like a vector store's similarity search coming back empty (relevant to the `allowEmptyContext` setting from the advanced-RAG guide).
4. **Map each Tavily hit into a Spring AI `Document`** — this is the genuinely important adapter step: Tavily's response shape (`title`, `url`, `content`, `score`) knows nothing about Spring AI, so this loop translates each web result into the exact same `Document` shape the rest of the RAG pipeline (joiners, augmenters, the model prompt) already knows how to work with. `title`/`url` go into metadata (useful for citing the source later), `content` becomes the document's actual text, and `score` carries Tavily's own relevance score along for the ride — the same `.score(...)` field a vector store's similarity search would populate with a cosine-similarity value.

### 2.4 The request/response DTOs

```java
@JsonNaming(PropertyNamingStrategies.SnakeCaseStrategy.class)
record TavilyRequestPayload(String query, String searchDepth, int maxResults) {}

record TavilyResponsePayload(List<Hit> results) {
    record Hit(String title, String url, String content, Double score) {}
}
```

Plain Java records used purely as JSON DTOs for Jackson to (de)serialize against. `@JsonNaming(PropertyNamingStrategies.SnakeCaseStrategy.class)` is the detail worth calling out: Tavily's actual API expects `search_depth`/`max_results` (snake_case), while idiomatic Java uses `searchDepth`/`maxResults` (camelCase) — this annotation tells Jackson to automatically convert between the two, so the Java code stays idiomatic while the wire format still matches what Tavily's API actually expects.

### 2.5 The builder

```java
public static Builder builder() { return new Builder(); }

public static class Builder {
    private RestClient.Builder clientBuilder;
    private int resultLimit = DEFAULT_RESULT_LIMIT;

    public Builder restClientBuilder(RestClient.Builder clientBuilder) { ... }
    public Builder maxResults(int maxResults) { ... }
    public WebSearchDocumentRetriever build() { ... }
}
```

Following the exact same fluent-builder convention every other Spring AI component in this series uses (`VectorStoreDocumentRetriever.builder()`, `SearchRequest.builder()`, and so on) — this is why swapping retrievers inside a `RetrievalAugmentationAdvisor` feels natural: they all share the same construction idiom.

---

## 3. A security note worth flagging explicitly

```java
private static final String TAVILY_API_KEY = "tvly-dev-PxdVn-...";
```

A hardcoded API key as a `static final` string, committed directly in source, is a real risk — anyone with access to the repository (or a public GitHub history, if this ever gets pushed) has your live API key. The fix is the same pattern used throughout this whole series for every other provider key:

```java
@Value("${tavily.api-key}")
private String tavilyApiKey;
```

```yaml
# application.yml
tavily:
  api-key: ${TAVILY_API_KEY}
```

Pull it from configuration (and ultimately an environment variable), exactly like `spring.ai.openai.api-key` or `spring.ai.anthropic.api-key` have been throughout this series, rather than a literal in the class. If this key has already been shared anywhere (including here), treat it as compromised and rotate it in the Tavily dashboard.

---

## 4. Wiring it into `RetrievalAugmentationAdvisor`

This is the entire payoff — once `WebSearchDocumentRetriever` exists, it plugs into the exact same modular pipeline from the advanced-RAG guide, just swapped in for `VectorStoreDocumentRetriever`:

```java
@Configuration
class WebRagConfig {

    @Bean
    DocumentRetriever webSearchDocumentRetriever(RestClient.Builder restClientBuilder) {
        return WebSearchDocumentRetriever.builder()
                .restClientBuilder(restClientBuilder)
                .maxResults(5)
                .build();
    }
}
```

```java
@Service
class WebRagService {

    private final ChatClient chatClient;
    private final DocumentRetriever webSearchDocumentRetriever;

    WebRagService(ChatClient chatClient, DocumentRetriever webSearchDocumentRetriever) {
        this.chatClient = chatClient;
        this.webSearchDocumentRetriever = webSearchDocumentRetriever;
    }

    public String answerWithLiveWebSearch(String query) {

        RetrievalAugmentationAdvisor advisor = RetrievalAugmentationAdvisor.builder()
                .documentRetriever(webSearchDocumentRetriever) // ← the only swap vs. the vector-store version
                .documentJoiner(new ConcatenationDocumentJoiner())
                .queryAugmenter(ContextualQueryAugmenter.builder()
                        .allowEmptyContext(true)
                        .build())
                .build();

        return chatClient.prompt(query)
                .advisors(advisor)
                .call()
                .content();
    }
}
```

Every other module — `QueryTransformer`, `QueryExpander`, `DocumentJoiner`, `QueryAugmenter` — works completely unchanged, because they all operate on the same `Document`/`Query` abstractions regardless of where the documents actually came from. This is the real value of Spring AI's modular RAG design: swapping the *source* of retrieval is a one-line change, not a pipeline rewrite.

### 4.1 Combining both sources — web search *and* your vector store

Nothing stops you from retrieving from both at once and merging the results — this is a genuinely common real pattern ("check our internal docs, and also check the live web"):

```java
RetrievalAugmentationAdvisor advisor = RetrievalAugmentationAdvisor.builder()
        .documentRetriever(query -> {
            List<Document> fromStore = vectorStoreDocumentRetriever.retrieve(query);
            List<Document> fromWeb = webSearchDocumentRetriever.retrieve(query);
            return Stream.concat(fromStore.stream(), fromWeb.stream()).toList();
        })
        .documentJoiner(new ConcatenationDocumentJoiner())
        .queryAugmenter(ContextualQueryAugmenter.builder().allowEmptyContext(true).build())
        .build();
```

Since `DocumentRetriever` is a functional interface (`extends Function<Query, List<Document>>`), a lambda combining two existing retrievers is all it takes — no need to write a whole new class just to merge two sources.

---

## 5. Web-search RAG vs. vector-store RAG — when to reach for each

| | Vector-store RAG | Web-search RAG |
|---|---|---|
| **Data freshness** | As current as your last ingestion run | Always current — live at request time |
| **Best for** | Your own private/internal/static data (docs, policies, product info) | Current events, rapidly changing facts, anything outside your own data |
| **Cost shape** | One-time (or scheduled) embedding cost at ingestion, cheap per-query retrieval afterward | A live external API call (and often its own associated cost) on every single request |
| **Reliability/control** | Fully within your control — you chose what's indexed | Dependent on a third-party search API's availability, quality, and rate limits |
| **Traceability** | Points back to your own known source documents | Points back to live URLs — genuinely useful citations, but to external, less controlled sources |

In practice, many real production assistants use **both**, exactly as shown in §4.1 — a vector store for the organization's own grounded knowledge, and a web search retriever as a fallback (or a parallel source) for anything current or outside that knowledge base's coverage, with `allowEmptyContext(true)` ensuring the assistant doesn't hard-fail just because one of the two sources came up empty for a given question.

## 6. Recap

`DocumentRetriever` is just an interface — `Query` in, `List<Document>` out — and nothing about the modular RAG architecture cares whether those documents came from a vector store or a live web search API. `WebSearchDocumentRetriever` implements exactly that contract against Tavily's search API, mapping each web hit into a standard Spring AI `Document` so every other module in a `RetrievalAugmentationAdvisor` pipeline (query transformation, expansion, joining, augmentation) works with it completely unchanged. The practical payoff is real-time grounding for questions a static vector store structurally cannot answer — at the cost of a live external API call, and a dependency on that provider's uptime, on every single request that uses it.
