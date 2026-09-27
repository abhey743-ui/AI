# Spring AI Advisors — Concept, Built-ins, Configuration, and Writing Your Own

## 1. Why advisors exist — the problem they solve

Every non-trivial AI-powered service ends up needing the same handful of things on top of "send a prompt, get a response": remembering conversation history, blocking unsafe input, logging what actually went to the model, retrying, redacting sensitive output, injecting retrieved documents (RAG). If you wrote all of that by hand inside every service method that calls `chatClient`, you'd end up duplicating the same boilerplate everywhere, and any change to "how we log requests" or "how we check for bad input" would mean touching dozens of methods.

**Advisors are Spring AI's answer to this — they're aspect-oriented programming (AOP) for the model call.** An advisor is a piece of logic that sits *around* the actual call to the model: it can inspect or rewrite the request before it goes out, and inspect or rewrite the response before it comes back to your calling code, without your service method needing to know any of that logic exists. You register an advisor once, and it applies to every request that flows through the `ChatClient` it's attached to.

Concretely, advisors are what make the following possible without cluttering your service code:
- **Conversation memory** — automatically injecting prior turns of a conversation into the prompt (`MessageChatMemoryAdvisor`).
- **RAG** — automatically retrieving relevant documents and stuffing them into the prompt (`QuestionAnswerAdvisor`).
- **Input safety checks** — blocking a request before it ever reaches the model if it contains disallowed content (`SafeGuardAdvisor`).
- **Logging/observability** — recording exactly what was sent and received, for debugging or auditing (`SimpleLoggerAdvisor`).
- **Anything custom** — rate limiting, PII redaction, response validation, secret-leak detection, retries — all follow the same shape.

---

## 2. The core interfaces and classes

Advisors live in `org.springframework.ai.chat.client.advisor.api`. The interfaces you actually work with:

```java
public interface Advisor extends Ordered {
    String getName();
}
```

`Advisor` itself just requires a name and, by extending `Ordered`, a position in the chain (`getOrder()`). It's split into two sub-interfaces depending on whether you're intercepting a blocking call or a reactive stream:

```java
public interface CallAdvisor extends Advisor {
    ChatClientResponse adviseCall(ChatClientRequest chatClientRequest, CallAdvisorChain callAdvisorChain);
}

public interface StreamAdvisor extends Advisor {
    Flux<ChatClientResponse> adviseStream(ChatClientRequest chatClientRequest, StreamAdvisorChain streamAdvisorChain);
}
```

- **`CallAdvisor`** — for the synchronous `.call()` path. Implement this if your service only ever calls `.call()`.
- **`StreamAdvisor`** — for the reactive `.stream()` path, working with a `Flux<ChatClientResponse>` instead of a single response.
- If you need to support both, implement both interfaces on the same class (this is exactly what `SimpleLoggerAdvisor` does internally).

Each of these methods receives a **chain** object:

```java
public interface CallAdvisorChain extends AdvisorChain {
    ChatClientResponse nextCall(ChatClientRequest chatClientRequest);
}
```

Calling `chain.nextCall(request)` (or `chain.nextStream(request)` on the streaming side) is how your advisor actually **invokes the next thing in line** — either the next advisor, or, if you're last in the chain, the actual `ChatModel` call. This is the exact mechanism that lets an advisor run code *before* the model call (anything before `chain.nextCall(...)`), *after* it (anything after, once the response comes back), or **both**.

An advisor can also choose **not** to call `chain.nextCall(...)` at all — that's how a blocking guardrail like `SafeGuardAdvisor` works: if it decides the request is unsafe, it simply returns its own response immediately, and the model is never actually invoked.

---

## 3. Ordering — why it matters and how it works

Every `Advisor` has a `getOrder()` value (an `int`). When a request passes through a `ChatClient` with multiple advisors registered, **Spring AI runs them in ascending order of `getOrder()`** — the lowest number runs first (closest to your call), and the chain proceeds inward toward the actual model call, then unwinds back out in reverse as each advisor's "after" logic executes.

This matters more than it looks like it should. A concrete real example: `SafeGuardAdvisor` needs to see the user's actual raw input *before* anything else has a chance to modify it — so it generally needs a **lower** order value (runs earlier) than something like `MessageChatMemoryAdvisor`, which injects prior conversation history into the prompt. If a memory advisor runs first and a sensitive word happens to already be sitting in the injected history rather than the current user input, a poorly-ordered safety check can trigger (or fail to trigger) for the wrong reason entirely. **Order isn't cosmetic — it changes what each advisor actually sees, and can change your application's behavior.** Choose it deliberately, and cover important chains with tests.

```java
@Override
public int getOrder() {
    return 20; // lower runs earlier; Advisor default order constants exist too
}
```

---

## 4. Built-in default advisors

### 4.1 `SimpleLoggerAdvisor` — logging requests and responses

`SimpleLoggerAdvisor` implements both `CallAdvisor` and `StreamAdvisor`. It logs the outbound request and the inbound response — useful for debugging exactly what's actually being sent to the model (including anything other advisors, like memory or RAG, injected into it) and what came back.

**Registering it at the bean level:**

```java
@Configuration
class AiConfig {

    @Bean
    ChatClient chatClient(ChatClient.Builder builder) {
        return builder
                .defaultAdvisors(SimpleLoggerAdvisor.builder().build())
                .build();
    }
}
```

**Registering it at the service level, per call:**

```java
public String chat(String question) {
    return chatClient.prompt()
            .user(question)
            .advisors(SimpleLoggerAdvisor.builder().build())
            .call()
            .content();
}
```

**Enabling its log output via `application.yml`:** `SimpleLoggerAdvisor` logs at `DEBUG` level, so simply registering it isn't enough on its own — your logging configuration also needs to actually let `DEBUG` messages through for its package:

```yaml
logging:
  level:
    org.springframework.ai.chat.client.advisor: debug
```

Without this, the advisor is running, but its log lines are being filtered out before you ever see them — a common point of confusion.

### 4.2 `SafeGuardAdvisor` — a basic input guardrail

`SafeGuardAdvisor` blocks the call to the model entirely if the user's input contains any word from a list of "sensitive words" you configure — and returns a canned failure response instead, without the request ever reaching the model.

```java
@GetMapping("/guard")
public String guard(@RequestParam String message) {

    var safeGuardAdvisor = SafeGuardAdvisor.builder()
            .sensitiveWords(List.of("password", "ssn", "credit card"))
            .failureResponse("I can't help with that request.")
            .build();

    return chatClient.prompt()
            .user(message)
            .advisors(safeGuardAdvisor)
            .call()
            .content();
}
```

**Important limitation to know about:** the built-in check is a plain **case-sensitive substring match**. `"PASSWORD"` in all caps sails right through a list containing only `"password"`, and a word like `"pass"` will false-positive on completely innocent input like "how do I **pass** an argument to a method?" It's a useful first layer, but not something to rely on as your only line of defense — treat it as one guardrail among several, not a complete solution.

You can also register it as a bean-level default, exactly like `SimpleLoggerAdvisor`, if you want it applied to every request through a given client rather than opted into per call:

```java
@Bean
ChatClient chatClient(ChatClient.Builder builder) {
    return builder
            .defaultAdvisors(
                    SafeGuardAdvisor.builder()
                            .sensitiveWords(List.of("password", "ssn"))
                            .failureResponse("Request blocked.")
                            .build(),
                    SimpleLoggerAdvisor.builder().build()
            )
            .build();
}
```

---

## 5. Bean level vs. service level — where to register advisors

Just like system messages and options (covered in earlier guides), advisors can be attached in two places, and the choice follows the same logic:

| | Bean level (`.defaultAdvisors(...)`) | Service level (`.advisors(...)`) |
|---|---|---|
| **Scope** | Applies to every request through that `ChatClient` | Applies only to this one call |
| **Best for** | Advisors that should always run — logging, memory, a baseline guardrail | Advisors specific to one endpoint's needs, or ones that need per-call configuration |
| **Combining behavior** | Registered once, at startup | **Adds to**, rather than replaces, the bean-level defaults — advisors passed at the service level are appended to the chain alongside the defaults, not instead of them |

```java
@Configuration
class AiConfig {

    @Bean
    ChatClient chatClient(ChatClient.Builder builder) {
        return builder
                .defaultAdvisors(SimpleLoggerAdvisor.builder().build()) // always runs
                .build();
    }
}
```

```java
@Service
class SupportService {

    private final ChatClient chatClient;

    SupportService(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    public String handle(String question) {
        return chatClient.prompt()
                .user(question)
                // This SafeGuardAdvisor only applies to calls through this specific method,
                // in ADDITION to the SimpleLoggerAdvisor already registered as a default.
                .advisors(SafeGuardAdvisor.builder()
                        .sensitiveWords(List.of("refund fraud"))
                        .failureResponse("This request cannot be processed.")
                        .build())
                .call()
                .content();
    }
}
```

### 5.1 Passing runtime parameters into an already-registered advisor

Sometimes you don't want to register a *new* advisor per call — you want to pass a **parameter** into an advisor that's already sitting in the chain as a default, so it behaves correctly for this specific request. The most common example is telling `MessageChatMemoryAdvisor` which conversation's history to load:

```java
public String chatWithMemory(String conversationId, String question) {
    return chatClient.prompt()
            .user(question)
            .advisors(advisorSpec -> advisorSpec.param(ChatMemory.CONVERSATION_ID, conversationId))
            .call()
            .content();
}
```

Here, `.advisors(Consumer<AdvisorSpec>)` isn't adding a new advisor to the chain — it's reaching into the shared advise-context that every advisor in the chain can read from, and setting a key/value pair (`ChatMemory.CONVERSATION_ID` → your actual ID) that the memory advisor (registered separately, usually as a bean-level default) picks up when it runs. This is the mechanism that lets a bean-level advisor still behave differently per request, per user, or per conversation, without you having to rebuild or reconfigure the advisor itself on every call.

---

## 6. Writing your own custom advisor

When none of the built-ins fit — a common real case is redacting a secret that leaked into the model's *response*, which `SafeGuardAdvisor` can't catch because it only inspects the incoming request — you implement `CallAdvisor` (and `StreamAdvisor` too, if you need streaming support) yourself.

### 6.1 A custom "before" advisor — masking sensitive input before it reaches the model

```java
public class PiiMaskingAdvisor implements CallAdvisor {

    private static final Pattern EMAIL = Pattern.compile("\\b[\\w.-]+@[\\w.-]+\\.\\w+\\b");
    private static final Pattern CREDIT_CARD = Pattern.compile("\\b(?:\\d[ -]*?){13,16}\\b");

    @Override
    public ChatClientResponse adviseCall(ChatClientRequest request, CallAdvisorChain chain) {

        String originalText = request.prompt().getUserMessage().getText();
        String maskedText = mask(originalText);

        ChatClientRequest maskedRequest = originalText.equals(maskedText)
                ? request
                : request.mutate()
                        .prompt(request.prompt().augmentUserMessage(maskedText))
                        .build();

        // Only after masking do we let the request continue down the chain toward the model.
        return chain.nextCall(maskedRequest);
    }

    private String mask(String text) {
        text = EMAIL.matcher(text).replaceAll("[EMAIL REDACTED]");
        text = CREDIT_CARD.matcher(text).replaceAll("[CARD REDACTED]");
        return text;
    }

    @Override
    public String getName() {
        return this.getClass().getSimpleName();
    }

    @Override
    public int getOrder() {
        return 10; // run early, before memory/logging/model advisors
    }
}
```

### 6.2 A custom "after" advisor — catching a secret that leaked into the response

```java
public class SecretLeakAdvisor implements CallAdvisor {

    private final List<String> secrets;

    public SecretLeakAdvisor(List<String> secrets) {
        this.secrets = secrets;
    }

    @Override
    public ChatClientResponse adviseCall(ChatClientRequest request, CallAdvisorChain chain) {

        // Let the model actually respond first — we need the real response to inspect it.
        ChatClientResponse response = chain.nextCall(request);

        String reply = response.chatResponse().getResult().getOutput().getText();
        boolean leaked = secrets.stream()
                .anyMatch(secret -> reply.toLowerCase().contains(secret.toLowerCase()));

        if (leaked) {
            return response.mutate()
                    .chatResponse(buildRedactedResponse("I'm not able to share that."))
                    .build();
        }

        return response;
    }

    @Override
    public String getName() {
        return this.getClass().getSimpleName();
    }

    @Override
    public int getOrder() {
        return 30; // runs after the model call, inspecting the real output
    }
}
```

### 6.3 Wiring a custom advisor in

Custom advisors are used exactly like the built-ins — as a bean-level default, or per call:

```java
@Bean
ChatClient chatClient(ChatClient.Builder builder) {
    return builder
            .defaultAdvisors(
                    new PiiMaskingAdvisor(),
                    SimpleLoggerAdvisor.builder().build(),
                    new SecretLeakAdvisor(List.of("sunrise50"))
            )
            .build();
}
```

```java
// Or opted into just one specific call:
chatClient.prompt()
        .user(message)
        .advisors(new SecretLeakAdvisor(List.of("sunrise50")), SimpleLoggerAdvisor.builder().build())
        .call()
        .content();
```

**The pattern to remember:** whether an advisor runs *before* the model, *after* it, or both is entirely a function of where you place your logic relative to `chain.nextCall(...)` inside `adviseCall(...)`. Code before that line runs before the model sees anything; code after that line runs once the real response exists. This single mechanism is what backs memory injection, RAG, guardrails, logging, and any custom cross-cutting concern you write yourself — they're all just different logic wrapped around the exact same `chain.nextCall(...)` call.

---

## 7. Recap

- Advisors are AOP for the model call — logic that wraps `chain.nextCall(...)`, running before the request goes out, after the response comes back, or both.
- `CallAdvisor`/`StreamAdvisor` are the interfaces you implement; `getOrder()` decides sequencing and genuinely changes behavior, not just log ordering.
- `SimpleLoggerAdvisor` needs its package's log level explicitly raised to `debug` in configuration, or its output is silently filtered.
- `SafeGuardAdvisor` is a real but limited input guardrail — case-sensitive substring matching only, best used as one layer among several.
- Register advisors as bean-level defaults for things that should always run, and at the service level (either as new advisors or as parameters into existing ones via `.advisors(spec -> spec.param(...))`) for anything that needs to vary per call or per conversation.
- Writing a custom advisor is just implementing `CallAdvisor`/`StreamAdvisor` and deciding what happens before and/or after `chain.nextCall(...)` — the exact same shape whether you're masking PII, catching leaked secrets, rate limiting, or anything else your application needs around every model call.
