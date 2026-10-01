# Spring AI Vector Databases — Part 2: Chunking, Document Readers, and `TokenTextSplitter`

*Continues directly from Part 1. This part covers turning raw source files into properly chunked `Document` objects before they ever reach `vectorStore.add(...)`.*

---

## 1. What "chunking" actually means, concretely

A **chunk** is simply one piece of a larger source document — a paragraph, a section, a fixed-size slice of text — that becomes its own individual `Document` object, gets its own individual embedding vector, and is retrieved (or not retrieved) independently of every other chunk from the same source file.

The key word is **independently**. When you call `vectorStore.similaritySearch(...)`, Qdrant (or whichever store you're using) doesn't know or care that five different chunks all came from the same original PDF — each stored `Document` is just one vector competing for relevance against every other stored vector, regardless of its origin. Chunking is the decision of *where you draw those boundaries* before anything is embedded at all.

---

## 2. What happens if you store a whole document as a single chunk

This is the natural first instinct — read a file, wrap its entire content in one `Document`, call `vectorStore.add(List.of(document))` — and it's exactly the failure mode worth understanding clearly, because it's the most common real mistake in a first RAG attempt.

### 2.1 One embedding vector has to represent *everything* in the document
An embedding model compresses text into a fixed-size vector. If that text is one sentence, the vector can represent its meaning fairly precisely. If that text is an entire 40-page manual, the vector has to somehow represent the *average meaning* of everything in those 40 pages at once — onboarding instructions, billing policy, security details, API docs, all blended into one point in vector space. The more topics a single chunk covers, the less precisely its vector represents *any one* of them.

### 2.2 Relevant matches get diluted by irrelevant surrounding content
Say a user asks "what's your refund policy for annual plans?" and the actual answer is one sentence buried inside a 40-page document that's mostly about something else entirely. The single whole-document vector's similarity score to that question reflects the *entire* document's content, not just the one relevant sentence — so a short, laser-focused FAQ entry about refunds elsewhere in your knowledge base can easily out-rank the big document, even though the big document technically *contains* the correct answer.

### 2.3 You blow past context-window and cost limits fast
Even if a single whole-document chunk *does* get retrieved as the top match, you're now stuffing the model's prompt with the entire document's text just to answer one narrow question — paying for (and risking context-window overflow from) dozens of pages of content that have nothing to do with what was actually asked. This is precisely the §4 cost/latency/context-window problem from the earlier prompt-stuffing guide, just arrived at via a different path (poor chunking instead of poor prompt design).

### 2.4 You can't cite a precise source
One of RAG's real benefits is traceability — "this answer came from this specific passage." A single giant chunk can only ever tell you "this answer came from *somewhere in* this entire document," which is a much weaker, less useful citation than pointing at the specific paragraph that actually backs the claim.

**The fix, unsurprisingly, is to split documents into smaller, more focused chunks before they're embedded and stored** — which is what the rest of this guide actually covers.

---

## 3. Reading the source file first — `DocumentReader` and `TikaDocumentReader`

Before you can split anything, you need to actually get the file's content into Spring AI's `Document` shape in the first place. This is the job of a **`DocumentReader`** — an interface with implementations for different source formats.

### 3.1 The dependency

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-tika-document-reader</artifactId>
</dependency>
```

`TikaDocumentReader` is built on **Apache Tika**, a library specifically designed to extract text from a huge range of file formats — Word (`.docx`), PowerPoint (`.pptx`), PDF, HTML, RTF, and dozens more — through one unified reader, rather than needing a different library per format. (Spring AI also ships narrower, format-specific readers — e.g. `spring-ai-pdf-document-reader` for PDFs specifically, `spring-ai-markdown-document-reader` for Markdown — but `TikaDocumentReader`'s whole appeal is covering almost anything with a single dependency, which is exactly why it was the right choice for your `.docx` file.)

### 3.2 Using it

```java
@Value("classpath:/PromptTemplates/Person.docx")
private Resource resource;

TikaDocumentReader tikaDocumentReader = new TikaDocumentReader(resource);
List<Document> docs = tikaDocumentReader.get(); // or .read() — equivalent methods on the reader
```

At this point, `docs` typically contains **one `Document`** — the entire extracted text of `Person.docx`, as a single blob, with Tika having stripped out the Word-specific formatting/XML and left you with plain text. This is exactly the "whole document as a single chunk" situation from §2 — `TikaDocumentReader`'s job stops at *extraction*, not *chunking*. Splitting that one big `Document` into several smaller ones is a separate, deliberate next step.

---

## 4. Splitting — manual chunking, by hand

Before reaching for a built-in splitter, it's worth understanding what chunking is actually doing mechanically, by writing a naive version yourself.

### 4.1 The simplest possible approach — fixed character count

```java
public List<Document> splitManually(Document original, int chunkSizeChars) {

    String fullText = original.getText();
    List<Document> chunks = new ArrayList<>();

    for (int start = 0; start < fullText.length(); start += chunkSizeChars) {
        int end = Math.min(start + chunkSizeChars, fullText.length());
        String chunkText = fullText.substring(start, end);

        chunks.add(Document.builder()
                .text(chunkText)
                .metadata(original.getMetadata()) // carry source metadata into every chunk
                .build());
    }

    return chunks;
}
```

This works, but it has the exact weakness flagged in the earlier chunking discussion: it will happily cut a sentence — or a word — clean in half the moment the character count hits its limit, with zero regard for where a sentence, paragraph, or section actually ends.

### 4.2 A slightly better manual approach — split on paragraph boundaries

```java
public List<Document> splitByParagraph(Document original) {

    String[] paragraphs = original.getText().split("\\n\\s*\\n"); // blank-line-separated paragraphs

    return Arrays.stream(paragraphs)
            .map(String::strip)
            .filter(p -> !p.isEmpty())
            .map(p -> Document.builder().text(p).metadata(original.getMetadata()).build())
            .toList();
}
```

Better — chunk boundaries now respect the document's own paragraph structure rather than an arbitrary character count — but it introduces a new problem: paragraph lengths are wildly inconsistent. A one-line paragraph becomes a tiny, context-poor chunk; a ten-paragraph wall of text (common in poorly formatted source documents) becomes one oversized chunk right back into the §2 problem.

**The takeaway from writing it by hand:** good chunking has to balance *respecting natural text boundaries* (sentences, paragraphs) against *keeping chunk sizes reasonably consistent* — and doing both well, reliably, across arbitrary real-world documents, is exactly the harder problem `TokenTextSplitter` exists to solve for you.

---

## 5. `TokenTextSplitter` — the built-in, token-aware splitter

```java
import org.springframework.ai.transformer.splitter.TokenTextSplitter;
```

This class ships as part of Spring AI's core model artifact — no extra dependency beyond whatever model/vector-store starters you already have pulls it in, since it's a fundamental part of the document-transformation pipeline, not an optional add-on. Internally, it uses a proper **token-counting library (the same family of byte-pair-encoding tokenizers used by modern LLMs)** rather than naive character or word counting — meaning its chunk-size limit lines up with the way the actual model will later count tokens against its context window, which is a materially more accurate measure than counting characters.

### 5.1 Your exact code, walked through

```java
TokenTextSplitter tokenTextSplitter = TokenTextSplitter.builder()
        .withChunkSize(50)
        .build();

vectorStore.add(tokenTextSplitter.split(doc));
```

- **`.builder()`** — a fluent builder for configuring the splitter's behavior before use.
- **`.withChunkSize(50)`** — the target chunk size, **measured in tokens, not characters** — each resulting chunk aims for roughly 50 tokens' worth of text. This is deliberately the same unit you'd use when thinking about a model's context window or `maxTokens`, from the chat options guide — chunking and generation share the same underlying "cost" unit.
- **`.split(doc)`** — takes your `List<Document>` (note: despite the variable name `doc`, this is a `List<Document>`, matching what `TikaDocumentReader.get()` returned) and returns a **new, larger `List<Document>`** — the original document(s), broken into many smaller chunk-sized `Document`s, each one ready to be individually embedded.
- **`vectorStore.add(...)`** — now receives *many* smaller, focused `Document`s instead of one giant one, directly fixing the §2 problem: each chunk gets its own precise embedding, each can be retrieved independently, and only the genuinely relevant slice gets stuffed into a prompt later.

### 5.2 Other tunable options on the builder

```java
TokenTextSplitter splitter = TokenTextSplitter.builder()
        .withChunkSize(500)              // target tokens per chunk
        .withMinChunkSizeChars(350)      // avoid producing tiny, near-useless trailing chunks
        .withMinChunkLengthToEmbed(5)    // discard chunks too short to be meaningfully embedded at all
        .withMaxNumChunks(10000)         // a safety ceiling against runaway splitting on a huge document
        .build();
```

- **`withChunkSize`** — your primary lever, same idea as `.withChunkSize(50)` above, just a bigger number for a less granular split.
- **`withMinChunkSizeChars`** — a floor so the splitter doesn't produce a flood of tiny, nearly-empty fragments near the tail end of a document whose length doesn't divide evenly by the chunk size.
- **`withMinChunkLengthToEmbed`** — discards chunks so short (a stray heading, a page number that leaked through extraction) that embedding them would add noise rather than retrievable value.
- **`withMaxNumChunks`** — a hard ceiling, mostly a safety net against an unexpectedly enormous input document producing an unreasonable number of chunks (and therefore an unreasonable number of embedding API calls).

### 5.3 Why `TokenTextSplitter` is a meaningfully better default than either manual approach from §4
- It respects **token boundaries intelligently**, which correlates with the model's actual cost/context-window unit far better than a raw character count does.
- Its minimum-size and discard-tiny-chunk options directly address the "inconsistent paragraph length" problem the manual paragraph-based splitter in §4.2 ran into, without you having to hand-write that edge-case handling yourself.
- It's the same tool used consistently across Spring AI's own ETL/ingestion examples — meaning it's the well-tested, idiomatic choice rather than a bespoke one-off.

---

## 6. Putting it together — the full ingestion flow, your original code annotated

```java
@Component
@AllArgsConstructor
public class DemoDataLoader {

    private VectorStore vectorStore;

    @PostConstruct
    public void loadData() {

        // 1. READ: extract raw text from the source file via Apache Tika
        TikaDocumentReader tikaDocumentReader = new TikaDocumentReader(resource);
        List<Document> doc = tikaDocumentReader.get(); // usually one large Document at this point

        // 2. SPLIT: break that large Document into many smaller, token-sized chunks
        TokenTextSplitter tokenTextSplitter = TokenTextSplitter.builder()
                .withChunkSize(50)
                .build();

        // 3. EMBED + STORE: each resulting chunk gets its own embedding vector,
        //    written into the configured VectorStore (Qdrant, from Part 1)
        vectorStore.add(tokenTextSplitter.split(doc));
    }
}
```

This is the complete **Extract → Transform → Load (ETL)** pipeline for RAG ingestion, in miniature: `TikaDocumentReader` extracts, `TokenTextSplitter` transforms (chunks), `vectorStore.add(...)` loads. Running this once at startup (via `@PostConstruct`, as you have it) is fine for a small demo dataset; for a real, growing knowledge base, this same three-step shape is what a scheduled or event-triggered ingestion job would run instead, every time source documents are added, updated, or removed.

---

## 7. Optimization checklist — tuning the chunking step specifically

- **Pick a chunk size based on your content's natural unit, not an arbitrary number.** A FAQ made of short, self-contained Q&A pairs wants a smaller chunk size than a narrative technical manual where a full explanation genuinely needs several paragraphs together.
- **Don't go so small that a chunk loses its own context.** A single isolated sentence, cut loose from the paragraph that gave it meaning, can retrieve as a false-positive match (shares keywords) while being useless or misleading on its own once stuffed into the prompt.
- **Don't go so large that you're back to the §2 problem.** If you find yourself needing a `topK` of 1 because larger values just pull in redundant giant chunks, your chunk size is probably still too big.
- **Preserve metadata through every chunk**, as the manual examples in §4 did explicitly (`.metadata(original.getMetadata())`) — losing the source filename/page number when splitting means losing the ability to filter (`vectorStore.delete("source == '...'")`) or cite sources later.
- **Consider overlap for narrative, non-FAQ-style content.** Spring AI's `TokenTextSplitter` chunks sequentially without built-in overlap between consecutive chunks; if your content has ideas that span a chunk boundary, a custom splitter (or a pre-processing step that duplicates a small trailing slice into the next chunk) can reduce the odds of an important detail getting stranded right at a cut point — this is the same overlap concept flagged as a common mitigation in the RAG theory guide.
- **Re-run ingestion whenever source documents change**, and make sure old chunks from a replaced/deleted document are actually removed (`vectorStore.delete(...)` by a `source` metadata filter) — otherwise your vector store silently accumulates stale, outdated chunks that can still get retrieved and contradict the current, correct source.
- **Test retrieval quality directly, not just end-to-end answer quality.** Run `similaritySearch(...)` on a handful of real questions and manually inspect *which* chunks come back — this isolates whether a bad final answer is a retrieval problem (wrong/no chunks surfaced) or a generation problem (right chunks, model just didn't use them well), which are fixed in completely different places.
