# Prompt Stuffing in Spring AI — Concept, Demo, and Sizing It Correctly

## 1. What "prompt stuffing" actually means

Prompt stuffing is the technique of taking data your application already has — a product catalog, a knowledge-base article, a customer's order history, a chunk of a PDF — and **embedding it directly inside the prompt** you send to the model, instead of assuming the model already "knows" it.

Spring AI's own documentation describes this plainly: since a model only knows what was in its training data (plus whatever fits in its context window at request time), if you want it to reason over *your* data, the practical option is to **place that data directly into the prompt**. This is sometimes just called "stuffing the prompt," and it's the foundational idea underneath Retrieval-Augmented Generation (RAG) — RAG is really just prompt stuffing where the data being stuffed is selected dynamically via a similarity search against a vector store, rather than being a fixed block of text.

**Why we bother doing this at all, instead of just asking the question:**
- The model has no live connection to your database, your files, or anything that happened after its training cutoff — the only way to give it that information for a specific answer is to hand it over as part of the request.
- It lets you ground the model's answer in facts you control, which reduces hallucination compared to letting the model guess from general training knowledge.
- It's far cheaper and faster to set up than fine-tuning a model on your data — no training run required, just assembling the right text at request time.

---

## 2. Demo code — plain, fixed-content stuffing

The simplest form: you already have a block of text (a small catalog, a policy document, a FAQ) and you always stuff the same content into every request for a given endpoint.

```java
@Service
public class ProductInfoService {

    @Value("classpath:/data/product-catalog.txt")
    private Resource productCatalog;

    private final ChatClient chatClient;

    public ProductInfoService(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    public String answer(String question) throws IOException {

        String catalogText = productCatalog.getContentAsString(StandardCharsets.UTF_8);

        String systemText = """
                You are a product support assistant. Answer the user's question using
                ONLY the information in the catalog below. If the answer isn't in the
                catalog, say you don't have that information.

                CATALOG:
                {catalog}
                """;

        return chatClient.prompt()
                .system(sys -> sys.text(systemText).param("catalog", catalogText))
                .user(question)
                .call()
                .content();
    }
}
```

Here, the entire `product-catalog.txt` file is "stuffed" into the system message on every single call. This works well while the file is small — a few hundred lines of product descriptions, say — but notice the shape of the problem already: every request re-sends the whole file, whether or not the question actually needs most of it.

---

## 3. Demo code — dynamic stuffing via retrieval (the RAG pattern)

The more scalable version doesn't stuff *everything* — it retrieves only the pieces of your data that are actually relevant to the current question, using a vector store's similarity search, and stuffs just those.

```java
@Service
public class RagAnswerService {

    private final ChatClient chatClient;
    private final VectorStore vectorStore;

    public RagAnswerService(ChatClient chatClient, VectorStore vectorStore) {
        this.chatClient = chatClient;
        this.vectorStore = vectorStore;
    }

    public String answer(String question) {

        List<Document> relevantDocs = vectorStore.similaritySearch(
                SearchRequest.query(question).withTopK(4)
        );

        String context = relevantDocs.stream()
                .map(Document::getContent)
                .collect(Collectors.joining("\n---\n"));

        String systemText = """
                Answer the question using only the context provided below.
                If the context doesn't contain the answer, say so honestly.

                CONTEXT:
                {context}
                """;

        return chatClient.prompt()
                .system(sys -> sys.text(systemText).param("context", context))
                .user(question)
                .call()
                .content();
    }
}
```

The key difference from §2: `withTopK(4)` means only the 4 most relevant chunks of your knowledge base get stuffed into this particular request — not your entire dataset. Which 4 chunks get chosen changes per question, because the similarity search re-runs against the question every time. This is the actual mechanism that lets prompt stuffing scale to datasets far larger than any single context window could ever hold — you're never stuffing the *whole* dataset, only the slice that's relevant right now.

---

## 4. Why you can't (and shouldn't) stuff big data directly

Fixed, "stuff the whole file in every time" prompt stuffing — the §2 style — falls apart for a handful of very concrete reasons:

### 4.1 The context window is a hard ceiling
Every model has a maximum number of tokens it can process in one request — the **context window**. This includes your system message, your user message, any conversation history, *and* the model's own output. If your stuffed data plus everything else exceeds that limit, Spring AI and the underlying provider simply won't process the excess — the request either gets truncated or rejected outright, depending on the provider. A ten-page policy document might fit; your entire customer database will not.

### 4.2 Cost scales directly with what you stuff
Hosted models bill by token count, for both input and output. If you stuff 50,000 tokens of context into every single request just so the model can maybe use 200 tokens of it to answer a specific question, you are paying for all 50,000 tokens, every single time, for every user, forever. This adds up fast at any real volume.

### 4.3 Latency scales with input size too
More tokens in means more tokens the model has to process before it can even start generating a response. A stuffed prompt that's needlessly huge measurably slows down every single request, even when the actual answer only depended on a tiny fraction of what was stuffed.

### 4.4 Quality degrades with irrelevant bulk — the "lost in the middle" effect
Even when a large stuffed prompt technically fits inside the context window, models are demonstrably worse at finding and using information buried in the middle of a very long context than information near the beginning or end. Stuffing everything "just in case" doesn't just cost more — it can make the model's actual answer worse, because the genuinely relevant sentence is now competing with thousands of irrelevant ones for the model's attention.

### 4.5 It doesn't scale past a certain dataset size, structurally
A support knowledge base of 5 articles can be stuffed whole. A support knowledge base of 50,000 articles physically cannot — no current context window is large enough, and even if one were, sections 4.2–4.4 mean you wouldn't want to anyway. Past a certain size, retrieval (as in §3) isn't an optimization, it's the only workable approach.

---

## 5. How to decide how much data to stuff

There's no single fixed number — it's a judgment call based on several concrete factors you can actually reason about:

### 5.1 Start from the model's context window, and budget backwards
Look up the specific model's context window (e.g. a given model might support anywhere from ~8K to over 200K tokens, depending on which one you pick). Then subtract:
- Tokens reserved for the system instructions themselves (the non-data part).
- Tokens reserved for the user's actual question.
- Tokens reserved for the model's response (you must leave room for the output, or it gets cut off).
- A safety margin — don't design right up against the hard limit, since token-counting off by a little is easy and the failure mode (a rejected or truncated request) is bad.

Whatever's left is your real budget for stuffed data, not the full context window number.

### 5.2 Measure actual token counts, don't guess from word counts
Words and tokens aren't the same thing — a rough rule of thumb is that a token is roughly ¾ of a word in English, but this varies. Use a proper tokenizer/token-counting utility for the specific model you're targeting when deciding how much text you can safely stuff, rather than eyeballing it.

### 5.3 Prefer relevance-based retrieval over raw size once your data grows
The moment your underlying dataset is larger than "a document or two," the right question isn't "how much can I stuff?" but "how do I pick only the relevant slice to stuff?" — that's exactly what `similaritySearch(...)` with a `topK` value is for (§3). Tune `topK` empirically: too low and you risk missing the passage that actually answers the question; too high and you're back to paying the costs in §4 for marginal or irrelevant chunks.

### 5.4 Chunk large source documents instead of stuffing them whole
If a single source document (a long PDF, a large policy manual) is itself too big to stuff in one piece, break it into smaller chunks (a few hundred tokens each is a common starting point) before it ever goes into the vector store. This is what makes fine-grained retrieval possible in the first place — you can't retrieve "the relevant paragraph" if the whole 80-page document was stored as a single unsplit blob.

### 5.5 Weigh accuracy against cost and latency for your specific use case
- A one-off internal tool with light traffic can afford to stuff more context generously — cost and latency matter less there.
- A high-traffic, customer-facing endpoint should be tuned much more conservatively — smaller `topK`, tighter chunking, aggressive relevance filtering — because both cost and response time compound across thousands of requests.
- If accuracy is suffering with a small `topK`, that's a signal to improve retrieval quality (better embeddings, better chunking, a similarity threshold) before reflexively just stuffing more raw text and hoping the model finds the needle in the haystack.

### 5.6 Set a similarity threshold, not just a count
Alongside `topK`, most vector store integrations let you set a minimum similarity score, so that if none of your stored documents are actually relevant to the question, you stuff *nothing* rather than the "closest" — but still irrelevant — matches. This avoids confidently feeding the model garbage context just because you asked for a fixed number of results.

---

## 6. Recap

Prompt stuffing is simply putting your own data into the prompt so the model can reason over it — everything from a tiny fixed reference file (§2) up through full retrieval-driven RAG (§3) is a variation on the same core idea. It breaks down at scale because context windows are finite, tokens cost money, latency grows with input size, and quality can actually degrade under irrelevant bulk. The fix isn't to stuff less arbitrarily — it's to stuff *only what's relevant*, sized against your model's real context budget, chosen dynamically via retrieval rather than fixed in advance, and tuned (via `topK` and a similarity threshold) against your specific accuracy, cost, and latency requirements.
