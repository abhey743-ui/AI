# Using Roles in Spring AI — Service Level vs Bean Configuration Level

Roles (`SYSTEM`, `USER`, `ASSISTANT`, `TOOL`) can be attached to a request in two fundamentally different places in a Spring AI application:

1. **At the service level** — inside the method that actually handles a single request, where you decide the role content per call, often per user, per query.
2. **At the bean configuration level** — when you build the `ChatClient` itself, where you set defaults that apply to *every* call made through that client unless a service-level call overrides them.

Understanding both, and when to reach for which, is the difference between code that's flexible and code that's a mess of repeated system prompts scattered across twenty different service methods.

---

## 1. Setting roles at the service level

At the service level you have several genuinely different ways to attach a role to a message before it goes to the model. They are not just stylistic variants — each one gives you a different amount of control, and some interact differently with defaults set at the bean level.

### 1.1 Building `Message` objects by hand and assembling a `Prompt`

This is the most explicit, most low-level way to do it. You construct a `SystemMessage` and a `UserMessage` yourself, collect them into a `List<Message>`, wrap that list in a `Prompt`, and hand the whole `Prompt` object to the `ChatClient`.

The reasoning behind doing it this way: you get full control over exactly what messages exist, in what order, and you can add more than one message of the same type (e.g. several `UserMessage`s representing a multi-turn conversation you're replaying), or mix in an `AssistantMessage` to give the model a prior turn as context, before it ever leaves your service method.

```java
public String respondFormally(String query) {

    Message system = SystemMessage.builder()
            .text("Respond as a formal, professional consultant.")
            .build();

    Message user = UserMessage.builder()
            .text(query)
            .build();

    Prompt prompt = Prompt.builder()
            .messages(List.of(system, user))
            .build();

    return chatClient.prompt(prompt)
            .call()
            .content();
}
```

**When this shines:** conversation replay (rebuilding a whole multi-turn history from a database), dynamically deciding how many and which kinds of messages to include based on business logic, or anything where the shape of the conversation itself is data rather than a fixed template.

**Trade-off:** it's verbose for the simple case of "one system instruction plus one user question." You're paying construction ceremony for flexibility you may not need on every call.

### 1.2 The fluent shorthand — `.system(...)` and `.user(...)` on the request builder

`ChatClient`'s fluent API lets you skip building `Message` objects entirely. Calling `.system("...")` and `.user("...")` on the request spec builds the equivalent `SystemMessage`/`UserMessage` internally for you.

```java
public String respondCasually(String query) {
    return chatClient.prompt()
            .system("Respond in a relaxed, conversational tone.")
            .user(query)
            .call()
            .content();
}
```

**When this shines:** the everyday case — one system instruction, one user question, nothing fancier. It reads almost like a sentence and needs no imports beyond `ChatClient` itself.

**Trade-off:** you lose the ability to easily inject multiple prior turns or an `AssistantMessage` inline — for that you'd still reach for `.messages(...)` on the same builder, which accepts a `List<Message>` if you need to mix the fluent style with manually built messages.

### 1.3 Passing a plain string straight into `.prompt(...)`

`chatClient.prompt("some text")` is a convenience overload — it's shorthand for "treat this whole string as the user message, with no system message attached at all (from this call)." It's the fastest way to fire off a one-off question with no per-call role logic, relying entirely on whatever system-level defaults were configured on the bean (more on that in §2).

```java
public String quickAnswer(String query) {
    return chatClient.prompt(query)
            .call()
            .content();
}
```

**When this shines:** stateless, single-purpose endpoints where the personality/behavior of the assistant is fixed and already baked in as a bean-level default — the service method itself has nothing role-specific to say.

**Trade-off:** if no default system message was configured on the bean, the model gets zero behavioral guidance for that call — it's a bare user message with nothing else.

### 1.4 Passing a fully-built `Prompt` object directly

Instead of the fluent `.system()`/`.user()` chain, you can build a `Prompt` yourself (as in §1.1) and hand it straight to `chatClient.prompt(prompt)`. This is the point where the "manual message building" approach and the "fluent chat client" approach meet — you get the flexibility of constructing arbitrary message lists, but still get to use the rest of the fluent chain (`.call()`, `.stream()`, `.entity(...)`) afterward.

```java
public String respondWithHistory(List<Message> conversationSoFar, String newQuestion) {

    List<Message> messages = new ArrayList<>(conversationSoFar);
    messages.add(UserMessage.builder().text(newQuestion).build());

    Prompt prompt = Prompt.builder().messages(messages).build();

    return chatClient.prompt(prompt)
            .call()
            .content();
}
```

**When this shines:** anywhere you need to combine "arbitrary, dynamically assembled message history" with "still get the convenience methods `ChatClient` offers on top."

### 1.5 Attaching per-call advisor parameters

Roles decide *what* the model sees; advisors decide *what happens around* that. At the service level you can pass parameters into the advisor chain for that one call only — most commonly a conversation ID for memory, or a runtime toggle for a RAG advisor — without touching the bean-level advisor configuration at all.

```java
public String respondWithMemory(String conversationId, String query) {
    return chatClient.prompt()
            .user(query)
            .advisors(advisorSpec -> advisorSpec.param(ChatMemory.CONVERSATION_ID, conversationId))
            .call()
            .content();
}
```

This doesn't add a new role by itself, but it's worth knowing it exists at the same "per-call" level as the role-setting methods above — this is how a `MessageChatMemoryAdvisor` configured as a *default* on the bean (§2) knows which conversation's history to pull in for *this specific* call, without you re-declaring the advisor here.

---

## 2. Setting roles (and their defaults) at the bean configuration level

Everything above happens inside a service method, per call. But you can also bake a role's content in once, when the `ChatClient` bean itself is constructed — and it then applies automatically to *every* call made through that client, unless a service-level call overrides it.

```java
@Configuration
class AiConfig {

    @Bean
    ChatClient chatClient(ChatClient.Builder builder) {
        return builder
                .defaultSystem("Act like a professional consultant at all times.")
                .defaultAdvisors(new SimpleLoggerAdvisor())
                .defaultOptions(ChatOptions.builder().temperature(0.4).build())
                .build();
    }
}
```

- **`.defaultSystem(...)`** sets the baseline `SystemMessage` for every request made through this client. If a service method later calls `.system("something else")` on a specific request, that call's system text **replaces** the default for that one call — it does not stack on top of it.
- **`.defaultAdvisors(...)`** registers advisors (memory, RAG, logging, tool-calling) that run on every request automatically — this is where cross-cutting behavior belongs, rather than being repeated inside every service method.
- **`.defaultOptions(...)`** sets model parameters (temperature, max tokens, model name) that apply unless a specific request overrides them via `.options(...)`.
- **`.defaultTools(...)`** and **`.defaultUser(...)`** exist too, for registering tools and a default user-message template respectively, following the same "baseline unless overridden per call" rule.

The practical effect: your service methods can stay short — often just supplying the user's actual question — because the *behavioral* role (the system message defining "who" the assistant is) already lives centrally in one configuration class, not copy-pasted into every method that happens to call the model.

---

## 3. Comparing the two levels

| Aspect | Service level | Bean configuration level |
|---|---|---|
| **Scope of change** | Affects one call / one method | Affects every call through that `ChatClient` bean |
| **Best for** | Content that depends on the specific request — the actual user question, dynamically decided instructions, conversation history assembled from a database | Content that's constant across your whole application (or a whole feature) — the assistant's persona, safety instructions, default temperature |
| **Flexibility** | High — full programmatic control per call, can branch on business logic before deciding what role content to send | Low by design — fixed once at startup, deliberately so it doesn't drift between calls |
| **Repetition risk** | High if used carelessly — the same system instruction copy-pasted into ten different service methods becomes a maintenance headache | Low — one place to change the assistant's core behavior |
| **Overriding behavior** | A service-level `.system(...)` call **replaces** whatever `.defaultSystem(...)` was set at the bean level, for that call only | Acts purely as a fallback baseline; never overrides an explicit per-call value |
| **Typical role involved** | Mostly `UserMessage` (always changes) and occasionally a per-call `SystemMessage` override; `AssistantMessage`/`ToolResponseMessage` when replaying history | Almost always `SystemMessage` (persona/behavior) and advisor/tool registration |
| **Testability** | Easier to unit-test in isolation — you can call the service method with different inputs and inspect the constructed `Prompt` | Requires spinning up the Spring context (or manually constructing the client) to verify defaults |

The two are not mutually exclusive — in a well-structured application you use **both at once**: a bean-level default system message defines the assistant's baseline personality/rules, and service-level code supplies the actual user question (and, occasionally, overrides the system message for a specific specialized endpoint that needs different behavior than the rest of the app).

---

## 4. Best practices

- **Put stable, application-wide instructions at the bean level.** If every single call through a given `ChatClient` should behave the same way ("You are a customer support agent, never discuss competitors"), that belongs in `.defaultSystem(...)`, not repeated in every controller/service method.
- **Reserve service-level `.system(...)` overrides for genuinely different behavior**, not for the common case. If you find yourself writing a near-identical system string in five different service methods, that's a sign it should have been a bean-level default (or a second, purpose-named `ChatClient` bean — see the multi-model configuration guide for that pattern).
- **Keep the user's actual input as data, not as part of the system message.** Never concatenate untrusted user text into the system instruction string — build it as a separate `UserMessage` (or via `.user(...)`) so the model's instruction hierarchy stays intact and you avoid a trivial prompt-injection vector.
- **Prefer the fluent `.system()`/`.user()` shorthand for the common single-turn case**, and only drop down to manually building `Message` objects and a `Prompt` when you actually need to assemble a variable-length or historical message list — the manual approach is more powerful but also more code to maintain for no benefit in the simple case.
- **Don't recreate cross-cutting behavior in every service method.** Conversation memory, RAG retrieval, and logging belong in advisors registered once via `.defaultAdvisors(...)` at the bean level — using `.advisors(...)` at the service level should be reserved for passing per-call *parameters* into those already-registered advisors (like a conversation ID), not for registering a brand-new advisor on every call.
- **Be deliberate about override semantics.** Remember `.defaultSystem(...)` is a fallback, not a base you can append to — if a particular endpoint needs "the default instructions, plus one extra rule," you must construct that combined string yourself in the service method rather than expecting Spring AI to merge them for you.
- **Name your beans by purpose, not by provider, once you have more than one `ChatClient`.** A `formalAdvisorChatClient` or `supportAgentChatClient` communicates intent far better at the injection site than an unqualified `chatClient` whose behavior is only knowable by reading the configuration class.

---

## 5. One-paragraph summary

Roles can be attached either where the request is actually made (service level — flexible, per-call, best for data that changes with every request) or where the client is built (bean configuration level — fixed, global, best for behavior that should never drift between calls). Spring AI's fluent API gives you several equally valid ways to attach a role at the service level — hand-built `Message` objects and a `Prompt`, the `.system()`/`.user()` shorthand, or a bare string — chosen based on how much structural flexibility that particular call actually needs; and it gives you `.defaultSystem()`/`.defaultAdvisors()`/`.defaultOptions()` at the bean level for everything that should just always be true. The healthiest applications use both together, deliberately, rather than picking one and forcing every use case through it.
