# Spring AI — Running Two (or More) Models Side by Side

Configuration, beans, and endpoint wiring for a Spring Boot app that talks to more than one chat model at once (e.g. OpenAI **and** Anthropic in the same app).

---

## 1. The core problem

Spring AI auto-configures **one `ChatModel` bean per provider starter you add**, plus a `ChatClient.Builder` wired to whichever `ChatModel` it finds. The moment you add a **second** starter (or a second model of the same provider), Spring can no longer guess which one you want by default — you get a `NoUniqueBeanDefinitionException`, or auto-configuration of `ChatClient.Builder` backs off entirely because it can't pick one `ChatModel` to wire it to.

The fix in every case is the same pattern: **name your beans explicitly, then select between them with `@Qualifier`.**

---

## 2. `application.yml` — two different providers

```yaml
spring:
  ai:
    # Primary model: OpenAI
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4o-mini
          temperature: 0.7

    # Secondary model: Anthropic
    anthropic:
      api-key: ${ANTHROPIC_API_KEY}
      chat:
        options:
          model: claude-sonnet-4-5
          temperature: 0.3
          max-tokens: 2048

    # Important when you have >1 provider starter on the classpath:
    # stop Spring Boot from trying to auto-pick a single default ChatClient.
    chat:
      client:
        enabled: false
```

`spring.ai.chat.client.enabled: false` disables the *default* auto-configured `ChatClient` bean entirely, so you can define your own named `ChatClient` beans below without a "which one is primary" conflict. Each provider's own `ChatModel` bean (`openAiChatModel`, `anthropicChatModel`) is still auto-configured normally from the `spring.ai.<provider>.*` properties above — you just take over the `ChatClient` wiring yourself.

Maven dependencies to match:

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
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-starter-model-openai</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-starter-model-anthropic</artifactId>
    </dependency>
</dependencies>
```

---

## 3. Bean configuration — one `ChatClient` per model

```java
@Configuration
class AiConfig {

    @Bean
    @Primary
    ChatClient openAiChatClient(OpenAiChatModel openAiChatModel) {
        return ChatClient.builder(openAiChatModel)
                .defaultSystem("You are a fast, concise general-purpose assistant.")
                .build();
    }

    @Bean
    ChatClient anthropicChatClient(AnthropicChatModel anthropicChatModel) {
        return ChatClient.builder(anthropicChatModel)
                .defaultSystem("You are a careful assistant for long-form reasoning and writing.")
                .build();
    }
}
```

Notes:
- `OpenAiChatModel` and `AnthropicChatModel` are injected as constructor parameters — Spring Boot's auto-configuration already created both from the `application.yml` properties in §2, each under Spring's default bean name (`openAiChatModel`, `anthropicChatModel`), so no `@Qualifier` is even needed *here* since the parameter types are already distinct.
- `@Primary` on the OpenAI client means any place that just injects a plain `ChatClient` with no qualifier gets this one by default. Use this on whichever model is your "default" choice.
- Each `ChatClient` gets its **own** `.defaultSystem(...)`, and later its own advisors/tools if needed — they're fully independent configurations, just sharing the same Spring context.

---

## 4. Using two models of the **same** provider

Spring AI's auto-configuration only ever creates **one** `ChatModel` per provider. If you need e.g. two different Anthropic models (say, Haiku for quick replies and Sonnet for deep analysis), you must hand-build the second `ChatModel` yourself:

```yaml
spring:
  ai:
    anthropic:
      api-key: ${ANTHROPIC_API_KEY}
      chat:
        options:
          model: claude-haiku-4-5      # this becomes the auto-configured default
    # No auto-config block for the second model — it's built manually in Java below.
```

```java
@Configuration
class MultiAnthropicConfig {

    @Value("${spring.ai.anthropic.api-key}")
    private String anthropicApiKey;

    @Bean
    ChatClient haikuChatClient(AnthropicChatModel anthropicChatModel) {
        // Uses the auto-configured default model (claude-haiku-4-5 from yml)
        return ChatClient.builder(anthropicChatModel).build();
    }

    @Bean
    ChatClient sonnetChatClient() {
        AnthropicApi anthropicApi = AnthropicApi.builder()
                .apiKey(anthropicApiKey)
                .build();

        AnthropicChatModel sonnetModel = AnthropicChatModel.builder()
                .anthropicApi(anthropicApi)
                .defaultOptions(AnthropicChatOptions.builder()
                        .model("claude-sonnet-4-5")
                        .temperature(0.3)
                        .build())
                .build();

        return ChatClient.builder(sonnetModel).build();
    }
}
```

The same manual-construction pattern applies to any provider when you need N > 1 models from it — build the `ChatModel` by hand with `.builder()...build()` instead of relying on auto-configuration, since auto-configuration only ever gives you one.

---

## 5. Injecting with `@Qualifier` and exposing separate endpoints

```java
@RestController
@RequestMapping("/api/chat")
class MultiModelController {

    private final ChatClient openAiChatClient;
    private final ChatClient anthropicChatClient;

    MultiModelController(
            @Qualifier("openAiChatClient") ChatClient openAiChatClient,
            @Qualifier("anthropicChatClient") ChatClient anthropicChatClient) {
        this.openAiChatClient = openAiChatClient;
        this.anthropicChatClient = anthropicChatClient;
    }

    @PostMapping("/openai")
    String chatOpenAi(@RequestBody String question) {
        return openAiChatClient.prompt()
                .user(question)
                .call()
                .content();
    }

    @PostMapping("/anthropic")
    String chatAnthropic(@RequestBody String question) {
        return anthropicChatClient.prompt()
                .user(question)
                .call()
                .content();
    }
}
```

Two endpoints, `/api/chat/openai` and `/api/chat/anthropic`, each hard-wired to its own model. This is the simplest and most explicit setup — good when the client (frontend, another service) already knows which model it wants.

### 5.1 One endpoint, model chosen by a path/query parameter

If you'd rather have a single endpoint that picks the model dynamically:

```java
@RestController
@RequestMapping("/api/chat")
class DynamicModelController {

    private final Map<String, ChatClient> clientsByName;

    DynamicModelController(
            @Qualifier("openAiChatClient") ChatClient openAiChatClient,
            @Qualifier("anthropicChatClient") ChatClient anthropicChatClient) {
        this.clientsByName = Map.of(
                "openai", openAiChatClient,
                "anthropic", anthropicChatClient
        );
    }

    @PostMapping("/{model}")
    ResponseEntity<String> chat(@PathVariable String model, @RequestBody String question) {
        ChatClient client = clientsByName.get(model);
        if (client == null) {
            return ResponseEntity.badRequest().body("Unknown model: " + model);
        }
        return ResponseEntity.ok(client.prompt().user(question).call().content());
    }
}
```

Call it as `POST /api/chat/openai` or `POST /api/chat/anthropic` with the same handler code behind both — useful when you expect to add a third or fourth model later without writing a new method each time.

### 5.2 Task-based routing instead of provider-based routing

A more realistic pattern than exposing raw provider names to the outside world: name your beans (and endpoints) by **purpose**, and decide internally which underlying model backs each purpose.

```java
@Configuration
class RoleBasedAiConfig {

    @Bean
    ChatClient fastChatClient(OpenAiChatModel openAiChatModel) {
        return ChatClient.builder(openAiChatModel)
                .defaultSystem("Answer briefly. Optimize for speed over depth.")
                .build();
    }

    @Bean
    ChatClient deepReasoningChatClient(AnthropicChatModel anthropicChatModel) {
        return ChatClient.builder(anthropicChatModel)
                .defaultSystem("Think step by step. Prioritize correctness and nuance over speed.")
                .build();
    }
}
```

```java
@RestController
@RequestMapping("/api")
class TaskRoutedController {

    private final ChatClient fastChatClient;
    private final ChatClient deepReasoningChatClient;

    TaskRoutedController(
            @Qualifier("fastChatClient") ChatClient fastChatClient,
            @Qualifier("deepReasoningChatClient") ChatClient deepReasoningChatClient) {
        this.fastChatClient = fastChatClient;
        this.deepReasoningChatClient = deepReasoningChatClient;
    }

    @PostMapping("/quick-reply")
    String quickReply(@RequestBody String question) {
        return fastChatClient.prompt().user(question).call().content();
    }

    @PostMapping("/deep-analysis")
    String deepAnalysis(@RequestBody String question) {
        return deepReasoningChatClient.prompt().user(question).call().content();
    }
}
```

This way, if you later swap which provider powers "deep analysis," only `RoleBasedAiConfig` changes — the controller and its API contract stay untouched.

---

## 6. Full reference `application.yml` (three-model example)

Putting it together for OpenAI (fast default), Anthropic (deep reasoning), and a local Ollama model (offline/dev fallback):

```yaml
spring:
  ai:
    chat:
      client:
        enabled: false   # we define our own named ChatClient beans

    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4o-mini
          temperature: 0.7

    anthropic:
      api-key: ${ANTHROPIC_API_KEY}
      chat:
        options:
          model: claude-sonnet-4-5
          temperature: 0.3
          max-tokens: 2048

    ollama:
      base-url: http://localhost:11434
      chat:
        options:
          model: llama3.1
          temperature: 0.5

logging:
  level:
    org.springframework.ai: DEBUG   # handy while wiring this up; drop to INFO/WARN in prod
```

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-starter-model-openai</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-starter-model-anthropic</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-starter-model-ollama</artifactId>
    </dependency>
</dependencies>
```

```java
@Configuration
class AiConfig {

    @Bean
    @Primary
    ChatClient openAiChatClient(OpenAiChatModel model) {
        return ChatClient.builder(model).build();
    }

    @Bean
    ChatClient anthropicChatClient(AnthropicChatModel model) {
        return ChatClient.builder(model).build();
    }

    @Bean
    ChatClient ollamaChatClient(OllamaChatModel model) {
        return ChatClient.builder(model).build();
    }
}
```

---

## 7. Checklist / common pitfalls

| Symptom | Cause | Fix |
|---|---|---|
| `NoUniqueBeanDefinitionException: ChatModel` | Two+ provider starters on the classpath, something autowires a bare `ChatModel`/`ChatClient` with no qualifier | Add `@Qualifier("xChatModel")` or `@Qualifier("xChatClient")`, or mark one `@Primary` |
| `NoSuchBeanDefinitionException: ChatClient.Builder` | Spring Boot's auto-config backed off because it couldn't resolve a single `ChatModel` to wire the default builder to | Set `spring.ai.chat.client.enabled=false` and define your own `ChatClient` beans manually, as in §3 |
| Only one model ever responds correctly / wrong model called | Constructor parameter type doesn't disambiguate (e.g. both injected as plain `ChatModel`) and no qualifier given | Prefer injecting the **concrete** type (`OpenAiChatModel`, `AnthropicChatModel`) where possible — Spring resolves by type first, qualifier only when types collide (e.g. two of the same provider) |
| Second model of the *same* provider is ignored | Auto-configuration only creates one `ChatModel` per provider, no matter how many "extra" properties you add | Build the second `ChatModel` manually with `.builder()...build()`, as in §4 |
| API key not picked up for the second config block | Property path typo'd or under the wrong provider namespace | Double check `spring.ai.<provider>.api-key` matches exactly; use `spring.ai.<provider>.chat.api-key` only if you specifically need to override just the chat model's key separately from other model types (embeddings, image, etc.) under the same provider |

---

## 8. Recap

- One `@Bean ChatClient` method per model, each wrapping its own auto-configured (or manually built) `ChatModel`.
- `@Qualifier("beanName")` at every injection point once you have more than one candidate of the same type.
- `spring.ai.chat.client.enabled=false` in YAML whenever you're defining your own `ChatClient` beans instead of relying on the single auto-configured default.
- Same-provider, multiple-models needs manual `ChatModel` construction — auto-configuration caps out at one per provider.
- Route to the right client either by explicit endpoint (`/api/chat/openai`), a dynamic path/query parameter, or — best for real apps — by **purpose-named** beans/endpoints (`fastChatClient`, `deepReasoningChatClient`) so callers never need to know which vendor is actually behind the scenes.
