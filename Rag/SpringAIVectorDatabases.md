# Spring AI Vector Databases — Part 1: The `VectorStore` Abstraction and Qdrant Setup

*This is part 1 of a two-part series. Part 1 covers the vector database concept, Spring AI's `VectorStore` abstraction, and a full Qdrant configuration walkthrough. Part 2 covers chunking — how documents actually get split before they're stored — including the exact manual and `TokenTextSplitter`-based approach you were already using.*

---

## 1. What a vector database actually is, in this context

Recall from the RAG guide: retrieval works by converting text into **embedding vectors** and finding which stored vectors are mathematically closest to a query's vector. A **vector database** is simply a database purpose-built to store large numbers of these vectors *alongside their original text and metadata*, and to answer "which of these are closest to this new vector?" efficiently — even across millions of entries, which a naive linear scan would struggle with.

In a Spring AI RAG pipeline, the vector database is the component that sits right in the middle of both phases: during indexing, your chunked, embedded documents get **written** into it; during a live request, the user's embedded question is used to **query** it for the most relevant chunks.

---

## 2. The `VectorStore` interface — Spring AI's portable abstraction

Just like `ChatModel` abstracts away provider differences for chat, `VectorStore` abstracts away vendor differences across vector database products — the same application code works whether you're backed by Qdrant, Redis, Postgres/PGVector, or any other supported store.

```java
public interface VectorStore extends DocumentWriter, VectorStoreRetriever {

    void add(List<Document> documents);

    void delete(List<String> idList);

    void delete(Filter.Expression filterExpression);

    default void delete(String filterExpression) { ... }

    List<Document> similaritySearch(String query);

    List<Document> similaritySearch(SearchRequest request);
}
```

- **`add(List<Document> documents)`** — embeds and stores a batch of documents. You never compute the embedding vector yourself — the `VectorStore` implementation calls the configured `EmbeddingModel` internally as part of `add(...)`.
- **`delete(List<String> idList)`** — removes documents by their IDs.
- **`delete(Filter.Expression)` / `delete(String filterExpression)`** — removes documents matching a metadata filter, rather than by explicit ID — e.g. "delete everything tagged `source == 'old-manual.pdf'`."
- **`similaritySearch(String query)`** — the simplest search: embed this query string, return the closest matches, using default settings (default top-K, no filters).
- **`similaritySearch(SearchRequest request)`** — the configurable version, letting you set `topK`, a similarity threshold, and metadata filter expressions explicitly.

### 2.1 The `Document` class — what actually gets stored

```java
Document document = Document.builder()
        .text("NovaTech Solutions was founded in 2018.")
        .metadata(Map.of("source", "company-overview.txt"))
        .build();
```

A `Document` bundles the actual text content with arbitrary metadata (source filename, page number, category tags, a timestamp — whatever you want to filter or trace by later). This is the unit `VectorStore.add(...)` and `similaritySearch(...)` both operate on — not raw strings.

### 2.2 `SearchRequest` — tuning a search

```java
List<Document> results = vectorStore.similaritySearch(
        SearchRequest.builder()
                .query("How do I cancel my subscription?")
                .topK(5)
                .similarityThreshold(0.75)
                .filterExpression("source == 'billing-faq.txt'")
                .build()
);
```

`topK` and `similarityThreshold` here are the exact same levers discussed conceptually in the RAG guide (§4 of that guide) — this is where you actually set them in code.

---

## 3. Supported vector store implementations

Spring AI ships `VectorStore` implementations for a wide range of vector database products, including (non-exhaustive, and growing over time): **Chroma, Elasticsearch, GemFire, MariaDB, Milvus, Neo4j, OpenSearch, Pinecone, PGVector (PostgreSQL), Qdrant, Redis, Typesense, Weaviate,** and **AWS S3 Vectors.** Each one is its own class (`QdrantVectorStore`, `RedisVectorStore`, `WeaviateVectorStore`, and so on) implementing the exact same `VectorStore` interface above — so switching vendors later is, in principle, a matter of swapping a dependency and a bit of config, not rewriting your retrieval logic.

---

## 4. Setting up Qdrant specifically

### 4.1 Dependency

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-vector-store-qdrant</artifactId>
</dependency>
```

This starter brings in Spring Boot auto-configuration for a `QdrantVectorStore` bean, implementing `VectorStore`. You'll also need a model starter providing an `EmbeddingModel` bean (e.g. `spring-ai-starter-model-openai`, which provides `OpenAiEmbeddingModel` in addition to the chat model) — the vector store needs *something* to actually turn text into vectors before it can store or search them; it doesn't do that embedding step itself.

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-openai</artifactId>
</dependency>
```

### 4.2 `application.yml`

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}   # used both for chat and for the embedding model

    vectorstore:
      qdrant:
        host: localhost
        port: 6334                  # Qdrant's gRPC port, not its REST port
        api-key: ${QDRANT_API_KEY}  # omit entirely for a local instance with no auth
        collection-name: novatech-docs
        use-tls: false
        initialize-schema: true     # let Spring AI create the collection on startup
```

| Property | Meaning | Default |
|---|---|---|
| `spring.ai.vectorstore.qdrant.host` | Qdrant server host | `localhost` |
| `spring.ai.vectorstore.qdrant.port` | Qdrant's **gRPC** port (not the REST port) | `6334` |
| `spring.ai.vectorstore.qdrant.api-key` | API key for authentication, if your Qdrant instance requires one | — |
| `spring.ai.vectorstore.qdrant.collection-name` | Which Qdrant collection to store/search in | `vector_store` |
| `spring.ai.vectorstore.qdrant.use-tls` | Whether to connect over TLS | `false` |
| `spring.ai.vectorstore.qdrant.initialize-schema` | Whether Spring AI creates the collection automatically on startup | `false` |

With `initialize-schema: false` (the default), you're expected to have already created the target collection yourself in Qdrant beforehand — worth knowing, since it's a common "why isn't anything being stored" trap for a fresh setup.

### 4.3 Manual bean configuration (when you need more control)

Auto-configuration covers the common case, but you can build the `QdrantVectorStore` bean yourself when you need to customize something beyond what properties expose:

```java
@Bean
public VectorStore vectorStore(QdrantClient qdrantClient, EmbeddingModel embeddingModel) {
    return QdrantVectorStore.builder(qdrantClient, embeddingModel)
            .collectionName("custom-collection")       // defaults to "vector_store"
            .contentFieldName("page_content")           // useful when pointing at a pre-existing collection
            .initializeSchema(true)
            .build();
}
```

### 4.4 Using it — the actual interaction, end to end

```java
@Service
public class KnowledgeBaseService {

    private final VectorStore vectorStore;

    public KnowledgeBaseService(VectorStore vectorStore) {
        this.vectorStore = vectorStore;
    }

    public void storeFacts(List<String> facts) {
        List<Document> documents = facts.stream()
                .map(fact -> Document.builder().text(fact).build())
                .toList();

        vectorStore.add(documents); // embeds + stores each one in Qdrant
    }

    public List<Document> findRelevant(String question, int topK) {
        return vectorStore.similaritySearch(
                SearchRequest.builder()
                        .query(question)
                        .topK(topK)
                        .build()
        );
    }

    public void removeBySource(String sourceFileName) {
        vectorStore.delete("source == '" + sourceFileName + "'");
    }
}
```

And wiring retrieval directly into a `ChatClient` call, stuffing the retrieved chunks into the prompt (the actual RAG pattern from the RAG guide, now with a real `VectorStore` behind it):

```java
public String answer(String question) {

    List<Document> relevant = vectorStore.similaritySearch(
            SearchRequest.builder().query(question).topK(4).build());

    String context = relevant.stream()
            .map(Document::getText)
            .collect(Collectors.joining("\n---\n"));

    return chatClient.prompt()
            .system(sys -> sys
                    .text("Answer using only this context:\n{context}")
                    .param("context", context))
            .user(question)
            .call()
            .content();
}
```

---

## 5. What's next

This covers getting documents *into* and *out of* a vector store once you already have `Document` objects ready to go — but notice `storeFacts(...)` above assumed you already had clean, bite-sized strings. In practice, your source material is a PDF, a Word document, a web page — one large blob of text — and *how you break that blob into the right-sized `Document` chunks before calling `vectorStore.add(...)`* is its own, genuinely important topic. That's exactly what Part 2 covers: `DocumentReader`s like `TikaDocumentReader`, why storing a whole document as a single chunk causes real problems, and how `TokenTextSplitter` (plus manual chunking) solves it.
