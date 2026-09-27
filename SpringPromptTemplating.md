# Prompt Templating in Spring AI — Dynamic System & User Messages

## 1. The simple approach, first (no templating at all)

Before touching templates, it's worth being crystal clear about what the "plain" version looks like, because templating is just a solution to a problem the plain version runs into.

```java
@Service
public class SimpleChatService {

    private final ChatClient chatClient;

    public SimpleChatService(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    public String getResponse(String query) {
        return chatClient.prompt()
                .system("You are a professional, formal consultant.")
                .user(query)
                .call()
                .content();
    }
}
```

This works fine as long as the system instruction is a **fixed string** — it never needs a piece of runtime data mixed into it. The `.system("...")` call here just wraps that literal text into a `SystemMessage` and sends it. There's no placeholder, no substitution, nothing dynamic — what you wrote is exactly what the model receives, every single time, for every user.

**Where this breaks down:** the moment your system instruction needs to change based on something you only know at request time — the user's name, their subscription tier, a language preference, a retrieved document, today's date, a persona selected via a dropdown — string concatenation becomes your only tool, and it gets messy fast:

```java
// Works, but ugly, error-prone, and impossible to keep in a separate file for a
// non-developer (like a prompt engineer or product manager) to edit.
String system = "You are a " + tone + " assistant named " + assistantName
        + ". Today's date is " + LocalDate.now() + ". Respond in " + language + ".";
```

This is exactly the gap prompt templating closes.

---

## 2. What prompt templating actually is

A **prompt template** is a piece of text — for either the system role or the user role — that contains **placeholders** (`{name}`, `{query}`, `{tone}`, etc.) instead of hardcoded values. At request time, you supply a `Map<String, Object>` of values, and Spring AI substitutes each placeholder with the corresponding value before the message is ever sent to the model.

Conceptually it's the same idea as a SQL prepared statement or a Thymeleaf/Freemarker view template: you write the *shape* of the content once, and fill in the *specific values* per request.

### Why we use it

- **Separation of concerns.** The instruction's wording lives in one place (often its own `.st` text file), while your Java service code only deals with supplying values — a prompt engineer or product owner can tweak the wording without touching Java at all.
- **Reusability.** The same template can back many different requests, each with different substituted values, instead of rewriting a slightly different string for every case.
- **Avoiding string-concatenation bugs.** No more manually gluing strings together and hoping you didn't forget a space or mis-order an argument.
- **Externalizing prompts from code.** Templates are commonly stored as classpath resources (`.st` files, by convention, since Spring AI's template engine is StringTemplate), which means they can be versioned, reviewed, and changed independently of a code deployment in some setups.
- **Consistency across a large app.** If ten different endpoints all need "act as {persona}, respond in {tone}," they can all point at the same template file rather than ten copies of similar-but-not-identical text.

---

## 3. Implementation — the standalone `PromptTemplate` / `SystemPromptTemplate` classes

This is the more explicit, lower-level way to do templating: you build a `PromptTemplate` (or its system-specific sibling `SystemPromptTemplate`) directly, pointing it either at inline text or at a `Resource`, then call `.createMessage(Map<String, Object>)` to get back an actual `Message` with the placeholders already filled in.

### 3.1 Loading the template text from a classpath resource

Put the template text in its own file — conventionally with a `.st` extension — for example `src/main/resources/PromptTemplates/SystemTemplate.st`:

```
You are a professional consultant helping with the following query: {query}.
Always answer with confidence and cite reasoning where relevant.
```

Inject it as a `Resource` and turn it into a message at request time:

```java
@Service
public class TemplatedSystemService {

    @Value("classpath:/PromptTemplates/SystemTemplate.st")
    private Resource systemTemplateResource;

    private final ChatClient chatClient;

    public TemplatedSystemService(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    public String getResponse(String query) {

        SystemPromptTemplate systemPromptTemplate =
                SystemPromptTemplate.builder().resource(systemTemplateResource).build();

        Message systemMessage = systemPromptTemplate.createMessage(Map.of("query", query));
        Message userMessage = new UserMessage(query);

        Prompt prompt = new Prompt(List.of(systemMessage, userMessage));

        return chatClient.prompt(prompt)
                .call()
                .content();
    }
}
```

`createMessage(Map.of(...))` is the moment the placeholder substitution actually happens — everywhere `{query}` appears in the template text gets replaced with the value you passed in, and the result comes back as a ready-to-use `SystemMessage`.

### 3.2 The same idea for the user role

`PromptTemplate` (the general-purpose, non-system-specific version) works exactly the same way for a `UserMessage`:

```java
PromptTemplate userTemplate = new PromptTemplate("Summarize the following in a {style} tone: {content}");
Message userMessage = userTemplate.createMessage(Map.of("style", "casual", "content", rawText));
```

Or, if all you need is the finished `Prompt` object rather than an individual `Message`, `PromptTemplate` also exposes `.create(Map)`, which returns a `Prompt` directly:

```java
PromptTemplate promptTemplate = new PromptTemplate("Tell me a {adjective} joke about {topic}");
Prompt prompt = promptTemplate.create(Map.of("adjective", "dry", "topic", "databases"));
```

---

## 4. Implementation — the fluent, inline way on `ChatClient`

Instead of building a standalone `PromptTemplate` object yourself, `ChatClient`'s fluent API lets you template inline, right at the call site, using a lambda:

```java
public String getResponse(String query, String tone) {
    return chatClient.prompt()
            .system(sys -> sys
                    .text("You are a {tone} assistant helping with: {query}")
                    .param("tone", tone)
                    .param("query", query))
            .user(query)
            .call()
            .content();
}
```

`.system(Consumer<PromptSystemSpec>)` gives you a spec object with `.text(...)` (accepting either an inline `String` or a `Resource`, so you can point it straight at your `.st` file too) and `.param(key, value)` to fill in each placeholder. The equivalent exists for the user role via `.user(Consumer<PromptUserSpec>)`:

```java
public String getResponse(String country) {
    return chatClient.prompt()
            .user(userSpec -> userSpec
                    .text("What is the capital of {country}?")
                    .param("country", country))
            .call()
            .content();
}
```

This is functionally identical to building a `PromptTemplate`/`SystemPromptTemplate` by hand and calling `.createMessage(...)` — it's just a more concise, inline way to express the exact same substitution, without a separate variable for the template object.

### 4.1 Using a resource file with the fluent style

You don't lose the "keep the wording in an external file" benefit by going fluent — `.text(...)` happily accepts a `Resource`:

```java
@Value("classpath:/PromptTemplates/SystemTemplate.st")
private Resource systemTemplate;

public String getResponse(String query) {
    return chatClient.prompt()
            .system(sys -> sys
                    .text(systemTemplate)
                    .param("query", query))
            .call()
            .content();
}
```

### 4.2 A pitfall worth calling out explicitly

`.system(...)` — whether called with a plain string or a lambda — **sets the entire system message for that request**; it does not append to whatever was set before it in the same chain. So writing something like:

```java
// Don't do this — the second .system(...) call silently replaces the first one.
chatClient.prompt()
        .system(someFixedText)
        .system(sys -> sys.text(templateResource).param("query", query))
        .call()
        .content();
```

...means only the **last** `.system(...)` call actually has any effect; the first one is built and then thrown away. If you need to combine a fixed instruction with a templated one, do it in a single `.system(...)` call — either by putting both pieces into one template file, or by concatenating the text yourself before passing it in — rather than chaining `.system(...)` multiple times expecting them to stack.

---

## 5. Implementation — templating at the bean configuration level

Templating isn't limited to the service level. `ChatClient.Builder` accepts the same `Consumer<PromptSystemSpec>` shape for its **default** system message, meaning the whole application's baseline instruction can itself be a template, with values supplied fresh on every request via advisors or per-call overrides:

```java
@Configuration
class AiConfig {

    @Bean
    ChatClient chatClient(ChatClient.Builder builder,
                           @Value("classpath:/PromptTemplates/AgentSystem.st") Resource agentSystemPrompt) {
        return builder
                .defaultSystem(p -> p
                        .text(agentSystemPrompt)
                        .param("assistantName", "Nova")
                        .param("today", LocalDate.now().toString()))
                .build();
    }
}
```

Anything supplied here becomes the baseline for every call through this client. A service-level `.system(...)` override, as always, replaces this default entirely for that specific call rather than merging with it — the same override rule applies to templated defaults as to plain-string ones.

---

## 6. Choosing between the two implementation styles

| | Standalone `PromptTemplate`/`SystemPromptTemplate` | Fluent inline lambda on `ChatClient` |
|---|---|---|
| **Verbosity** | More ceremony — you name a variable, call `.createMessage(...)`, then still have to assemble a `Prompt` yourself | Less code — everything happens inline in the same fluent chain |
| **Reusability across methods** | Easy — build the template once (e.g. as a field or a shared bean) and reuse `.createMessage(...)` with different maps everywhere | Reasonable, but the lambda is usually written per call site unless you factor it into a shared method |
| **Best for** | Cases where you need the `Message`/`Prompt` object itself for something else — logging it, storing it, mixing it into a hand-assembled multi-message `Prompt` | The common case — one call, one templated system or user message, nothing else needed |
| **Resource-file support** | Native — constructors/builders accept a `Resource` directly | Also supported — `.text(Resource)` works the same way |

Both ultimately go through the same StringTemplate-based substitution engine underneath; picking one over the other is a matter of how much extra structure you need around the resulting message, not a difference in templating capability.

---

## 7. Best practices

- **Externalize wording that's likely to change.** If a system prompt is going to be iterated on by someone other than a backend developer, put it in its own `.st` resource file rather than an inline Java string literal.
- **Keep placeholder names descriptive.** `{query}`, `{tone}`, `{assistantName}` read far better months later than `{a}`, `{b}`, `{x}`.
- **Never build the same message twice.** As shown in §4.2, calling `.system(...)` more than once per request is a common copy-paste mistake — only the final call survives. If you're combining a manually built `Message` *and* the fluent `.system(...)` call in the same request, pick one path, not both.
- **Validate your `Map` keys match your placeholders.** A typo'd param key silently leaves the placeholder text (`{query}`) untouched in the final prompt instead of failing loudly — worth a quick unit test on any template you rely on in production.
- **Prefer the fluent lambda for the common single-message case**, and reach for a standalone `PromptTemplate` only when you specifically need the `Message`/`Prompt` object itself before it reaches `ChatClient` — e.g. to log it, cache it, or splice it into a larger hand-built message list.
- **Template the bean-level default too, once the persona itself needs runtime data** (today's date, the deployed environment name, a tenant identifier) — you don't have to choose between "templating" and "defaults set once at the bean level"; they compose exactly as described in §5.
