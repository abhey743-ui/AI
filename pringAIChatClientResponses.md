# Spring AI `ChatClient` Responses — `content()`, `chatResponse()`, `chatClientResponse()`, `entity()`, and Friends

## 1. Where these methods actually sit in the fluent chain

Every `ChatClient` interaction follows the same shape:

```java
chatClient.prompt()      // 1. build the request
        .user("...")     // 2. add messages, options, advisors...
        .call()           // 3. choose synchronous mode
        .content();       // 4. ← THIS is what this guide is about
```

Step 3 — `.call()` (or `.stream()` for the reactive path) — **doesn't actually send anything to the model yet**. It only decides *how* you're going to receive the answer: synchronously as one complete response, or as a stream of incremental chunks. The real HTTP request to the AI provider is fired the moment you call one of the **terminal methods** in step 4 — `content()`, `chatResponse()`, `chatClientResponse()`, `entity(...)`, or `responseEntity(...)`.

This guide is entirely about that step 4 — the different shapes you can ask for the model's answer to come back in, and when to reach for each one.

---

## 2. `content()` — just give me the text

```java
String answer = chatClient.prompt()
        .user("Tell me a joke")
        .call()
        .content();
```

This is the simplest, most common terminal method. It reaches into the model's response, pulls out the generated text, and hands you back a plain `String` — nothing else. No metadata, no token counts, no way to inspect *how* the answer was produced, just the answer itself.

**When to reach for it:** the overwhelming majority of the time — a chatbot reply, a generated summary, a translated sentence, anything where all you actually need is the text itself and nothing about the surrounding response matters to your calling code.

---

## 3. `chatResponse()` — the text, plus everything about how it was generated

```java
ChatResponse chatResponse = chatClient.prompt()
        .user("Tell me a joke")
        .call()
        .chatResponse();
```

`ChatResponse` is the full, structured response object Spring AI gets back from the underlying `ChatModel`. Where `content()` threw almost everything away except the text, `chatResponse()` keeps it all:

- **`getResult()`** — the primary `Generation`, which itself wraps an `AssistantMessage` (this is where the actual text ultimately lives, via `.getOutput().getText()`).
- **`getResults()`** — a `List<Generation>`, because some providers/configurations can return *multiple* candidate completions for a single request (`n > 1`), not just one.
- **`getMetadata()`** — everything about the *generation itself*: token usage (`getUsage()` — prompt tokens, completion tokens, total tokens), the finish reason (did it complete normally, or get cut off by `maxTokens`?), and provider-specific metadata.

```java
ChatResponse response = chatClient.prompt().user("Explain recursion").call().chatResponse();

String text = response.getResult().getOutput().getText();
long totalTokens = response.getMetadata().getUsage().getTotalTokens();
String finishReason = response.getResult().getMetadata().getFinishReason();
```

**When to reach for it:** anywhere you need to *do something* with metadata about the call itself — logging token usage for cost tracking, checking whether the response got cut off early, inspecting multiple candidate generations, or any observability/auditing use case where the raw text alone isn't enough.

---

## 4. `chatClientResponse()` — the response, plus the ChatClient's own execution context

```java
ChatClientResponse clientResponse = chatClient.prompt()
        .user("Tell me a joke")
        .call()
        .chatClientResponse();
```

This one is easy to confuse with `chatResponse()` because the names are so close, but they answer different questions:

- **`chatResponse()`** answers "what did the *model* say, and what did the *model* report about generating it?"
- **`chatClientResponse()`** answers "what did the whole *ChatClient pipeline* — including the advisor chain — do with this request?"

`ChatClientResponse` is a wrapper containing the underlying `ChatResponse` **plus** the shared advise-context — the same context map that advisors read from and write to as a request/response passes through the advisor chain (recall from the advisors guide: things like a conversation ID, or state a custom advisor stashed for another advisor downstream to pick up). If you need to inspect something an *advisor* attached to the exchange — not something the *model* reported — this is the object that has it.

```java
ChatClientResponse response = chatClient.prompt()
        .user("Continue our conversation")
        .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, "session-42"))
        .call()
        .chatClientResponse();

ChatResponse underlyingModelResponse = response.chatResponse();
Map<String, Object> advisorContext = response.context();
```

**When to reach for it:** advanced scenarios involving custom advisors — when you need access to whatever an advisor put into the shared context during the request/response lifecycle, not just what the model itself returned.

---

## 5. `entity(...)` — map the response straight into a Java object

Often the answer you actually want isn't prose at all — it's structured data you intend to immediately deserialize into a POJO/record. `entity(...)` does the "ask the model to produce JSON, then parse that JSON into your type" work for you, in one call.

```java
record ActorFilms(String actor, List<String> movies) {}

ActorFilms actorFilms = chatClient.prompt()
        .user("Generate the filmography for a random actor.")
        .call()
        .entity(ActorFilms.class);
```

You now have a real `ActorFilms` object — no manual JSON parsing, no `ObjectMapper` boilerplate in your service code. Spring AI handles instructing the model to produce a schema-conforming response and converting the result for you.

### 5.1 Generic types — `entity(ParameterizedTypeReference<T>)`

A plain `Class<T>` can't express a generic type like `List<ActorFilms>` (type erasure), so there's an overload that accepts a `ParameterizedTypeReference` instead:

```java
List<ActorFilms> filmographies = chatClient.prompt()
        .user("Generate the filmography of 5 movies for Tom Hanks and Bill Murray.")
        .call()
        .entity(new ParameterizedTypeReference<List<ActorFilms>>() {});
```

### 5.2 A custom converter — `entity(StructuredOutputConverter<T>)`

For full control over exactly how the raw output gets converted into your type (rather than relying on the default schema-based conversion), there's an overload accepting your own `StructuredOutputConverter<T>` implementation.

### 5.3 Reliability options on `entity(...)`

Structured output isn't always perfect on the first try — models occasionally produce output that doesn't quite match the requested schema. Every `entity(...)` (and `responseEntity(...)`) overload optionally accepts a `Consumer<EntityParamSpec>` for extra reliability:

```java
ActorFilms actorFilms = chatClient.prompt()
        .user("Generate the filmography for a random actor.")
        .call()
        .entity(ActorFilms.class, spec -> spec.validateSchema());
```

`validateSchema()` turns on a **self-correcting retry loop**: if the response doesn't validate against the expected schema, Spring AI feeds the specific validation error back into the prompt and retries automatically (a bounded number of attempts) before giving up — meaningfully more robust than a bare `entity(...)` call for anything you actually depend on structurally.

**When to reach for `entity(...)`:** any time the actual value you want back is structured data — a record, a list, a small DTO — rather than free-form prose you'll display as-is.

---

## 6. `responseEntity(...)` — structured data *and* the full model response together

`entity(...)` gives you the parsed object and nothing else — if you also need token usage or other response metadata for that same call, `entity(...)` alone can't give you both without triggering the request a second time. `responseEntity(...)` solves exactly this: it returns a `ResponseEntity<ChatResponse, T>` carrying **both** pieces from the same single request.

```java
ResponseEntity<ChatResponse, ActorFilms> result = chatClient.prompt()
        .user("Generate the filmography for a random actor.")
        .call()
        .responseEntity(ActorFilms.class);

ActorFilms films = result.entity();
ChatResponse rawResponse = result.response();
long totalTokens = rawResponse.getMetadata().getUsage().getTotalTokens();
```

Same generic-type and custom-converter overloads exist here as with `entity(...)` (`responseEntity(ParameterizedTypeReference<T>)`, `responseEntity(StructuredOutputConverter<T>)`), and the same `Consumer<EntityParamSpec>` reliability options (`validateSchema()`, etc.) apply too.

**When to reach for it:** whenever you need structured output *and* metadata (token usage, finish reason) from the same call — logging cost per structured extraction, for instance — without wastefully calling the model twice to get both pieces separately.

---

## 7. The streaming counterparts

Everything above assumed `.call()` — the synchronous path. Swap it for `.stream()` and the same idea applies, but every terminal method now returns a reactive `Flux` instead of a plain value, emitting incremental pieces as the model generates them:

```java
Flux<String> streamedText = chatClient.prompt()
        .user("Write a short story")
        .stream()
        .content();               // Flux<String> — one chunk of text at a time

Flux<ChatResponse> streamedResponses = chatClient.prompt()
        .user("Write a short story")
        .stream()
        .chatResponse();          // Flux<ChatResponse> — one partial ChatResponse per chunk
```

The conceptual mapping is exactly the same as the synchronous versions (`content()` → just text, `chatResponse()` → full structured response) — the only difference is *how many times* you receive a value and *when* (incrementally, as the model generates, rather than once at the end).

---

## 8. Choosing the right one — summary table

| Method | Returns | Use it when... |
|---|---|---|
| `content()` | `String` | You just need the answer text — the overwhelming default case |
| `chatResponse()` | `ChatResponse` | You need token usage, finish reason, or multiple candidate generations |
| `chatClientResponse()` | `ChatClientResponse` | You need something an **advisor** stashed in the shared execution context, not just what the model said |
| `entity(Class<T>)` | `T` | You want the answer parsed directly into a POJO/record |
| `entity(ParameterizedTypeReference<T>)` | `T` (generic) | Same as above, but the target type is generic (e.g. `List<Foo>`) |
| `entity(T, spec -> spec.validateSchema())` | `T` (retried) | Same as `entity(...)`, with an automatic self-correcting retry if the schema doesn't validate |
| `responseEntity(Class<T>)` | `ResponseEntity<ChatResponse, T>` | You need **both** the parsed object *and* the raw `ChatResponse` metadata, from a single call |
| `.stream().content()` | `Flux<String>` | You want incremental text chunks as they're generated (e.g. a typing-effect UI) |
| `.stream().chatResponse()` | `Flux<ChatResponse>` | Same as above, but each streamed chunk keeps its own response metadata |

---

## 9. One-paragraph recap

`.call()`/`.stream()` only pick the *mode*; the terminal method you chain after them decides the *shape* of what comes back. `content()` is the everyday default — plain text, nothing else. `chatResponse()` unlocks the model's own metadata (tokens, finish reason, multiple candidates). `chatClientResponse()` goes one layer further out, exposing what the advisor chain itself did with the request/response, not just what the model returned. `entity(...)` and `responseEntity(...)` are for when the actual payload you want is structured data rather than prose — the latter being the one to reach for when you need the parsed object *and* the response metadata from the same single call, rather than paying for two calls to get both.
