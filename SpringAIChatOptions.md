# Spring AI ChatOptions — Parameters, Why They Matter, and How to Configure Them

## 1. What `ChatOptions` are, and why we use them

`ChatOptions` is Spring AI's portable representation of **model generation parameters** — the knobs that control *how* the model generates text, as opposed to *what* it's being asked (that's the job of messages/roles). Things like how creative vs. deterministic the output is, how long the response can be, which exact model variant answers the request, and when generation should stop early.

```java
public interface ChatOptions {
    String getModel();
    Double getFrequencyPenalty();
    Integer getMaxTokens();
    Double getPresencePenalty();
    List<String> getStopSequences();
    Double getTemperature();
    Integer getTopK();
    Double getTopP();
    // ...
    static ChatOptions.Builder builder() { ... }
}
```

**Why we use them at all, instead of just always taking the model's defaults:**
- **Cost control.** `maxTokens` directly caps how many output tokens a call can generate, and output tokens are billed — an unbounded response on a high-traffic endpoint is an unbounded cost.
- **Consistency vs. creativity trade-off.** A classification or data-extraction endpoint wants near-deterministic output every time; a brainstorming or creative-writing endpoint wants variety. `temperature`/`topP`/`topK` are exactly the levers for that.
- **Predictable formatting.** `stopSequences` lets you cut generation off cleanly at a marker you control, instead of trusting the model to stop exactly where you want.
- **Reducing repetition or steering topic diversity.** `frequencyPenalty`/`presencePenalty` exist specifically to fight the failure mode of a model looping on the same phrase, or to nudge it toward covering new ground.
- **Model selection itself is an option.** `model` lets the same code target a cheaper/faster model for simple tasks and a stronger model for complex ones, without changing anything else about how the request is built.

Every one of these can be set with a sensible **default** and then **overridden per request** — which is exactly why Spring AI gives you three places to configure them (yml, bean, service level — covered in §3).

---

## 2. The parameters, explained

### `model`
Which specific model variant handles the request (e.g. `gpt-4o-mini` vs `gpt-4o`, or `claude-haiku-4-5` vs `claude-sonnet-4-5`). This isn't a "creativity" knob like the others — it's a straight choice of *which* model answers, which in turn affects cost, speed, and capability. Setting it as an option (rather than hardcoding it elsewhere) is what makes "use a cheap model for quick replies, a stronger one for deep analysis" a one-line change.

### `temperature`
Controls **randomness** in how the next token is picked. Conceptually: near `0.0` makes the model pick the single most likely next word almost every time (deterministic, focused, "boring" in a good way for factual tasks); higher values (commonly up to `1.0`, sometimes higher depending on provider) let it sample from a wider spread of plausible next words, producing more varied, creative, sometimes less predictable output.

- **Low temperature (e.g. `0.1`–`0.3`)** — data extraction, classification, code generation, anything with one "correct" answer.
- **High temperature (e.g. `0.8`–`1.2`)** — creative writing, brainstorming, casual conversation.

### `topP` (nucleus sampling)
An alternative (or companion) way of controlling randomness: instead of picking from *all* possible next tokens weighted by probability, the model only considers the smallest set of tokens whose combined probability adds up to `topP`. A `topP` of `0.1` means "only consider tokens from the top 10% of the probability mass" — a much narrower, more focused set of candidates than `1.0` (consider everything).

**Providers commonly advise against tuning both `temperature` and `topP` at the same time** — since they both affect the same underlying sampling process, changing both together makes the actual effect harder to predict. Pick one as your primary lever and leave the other at its default.

### `topK`
A related but distinct idea: instead of a probability-mass cutoff (`topP`), `topK` restricts the model to only the **K most likely next tokens**, full stop, regardless of how much combined probability they represent. A `topK` of `40` means "only ever consider the 40 most probable next words at each step."

**Important nuance:** `topK` is **not universally supported**. It's a real, commonly-tuned parameter for Anthropic, Google Gemini/Vertex, and Ollama — but the OpenAI Chat Completions API itself has no `top_k` concept, so setting it when targeting OpenAI is simply ignored by that provider (Spring AI's portable `ChatOptions` still exposes the field, but it can't create an effect the underlying API doesn't support).

### `maxTokens`
A hard ceiling on how many tokens the model is allowed to generate in its response. This is your primary lever for **controlling both response length and cost** — since output tokens are billed, and an unexpectedly long response is both a cost risk and, for something like a chat UI, a UX risk (walls of text nobody asked for). If the model would have kept going past this limit, the response is simply cut off at the boundary — so set it generously enough for the task to actually finish making its point.

### `frequencyPenalty`
A value (typically `-2.0` to `2.0`) that penalizes tokens **based on how often they've already appeared** in the text generated so far. A positive value makes the model progressively less likely to repeat the same words/phrases it's already used — directly fighting the common failure mode of a model getting stuck looping on a phrase or sentence structure.

### `presencePenalty`
Similar range and shape to `frequencyPenalty`, but the distinction matters: `presencePenalty` penalizes a token simply for **having appeared at all**, regardless of how many times — it doesn't scale with repeat count the way `frequencyPenalty` does. A positive `presencePenalty` nudges the model toward introducing genuinely new topics/words it hasn't used yet in the response, rather than specifically punishing verbatim repetition.

*(A useful mental shortcut: `frequencyPenalty` fights "saying the same thing over and over"; `presencePenalty` encourages "talking about something new.")*

### `stopSequences`
A list of specific strings that, if the model generates them, cause generation to stop immediately — the stop sequence itself is not included in the output. This is how you cleanly bound a response to a format you control, e.g. stopping right when the model would start a new "User:" turn in a simulated dialogue, or ending a list at a specific delimiter you define, rather than hoping the model naturally stops at the right spot.

---

## 3. Provider support isn't uniform — a quick reality check

`ChatOptions` is Spring AI's **portable** abstraction — it exposes a common shape across providers, but not every provider's underlying API actually implements every field:

| Parameter | OpenAI | Anthropic | Notes |
|---|---|---|---|
| `temperature` | ✅ | ✅ | Universally supported |
| `topP` | ✅ | ✅ | Universally supported |
| `topK` | ❌ (no effect) | ✅ | OpenAI's API has no `top_k` concept |
| `maxTokens` | ✅ | ✅ (required by Anthropic) | Anthropic's API actually requires `max_tokens` to be set on every request |
| `frequencyPenalty` | ✅ | ❌ (no effect) | Not part of Anthropic's Messages API |
| `presencePenalty` | ✅ | ❌ (no effect) | Not part of Anthropic's Messages API |
| `stopSequences` | ✅ | ✅ | Universally supported, naming may differ under the hood |

When you need a parameter that's genuinely provider-specific and not part of the portable `ChatOptions` interface at all (like OpenAI's `seed` for deterministic output, or Anthropic's `thinking` budget), use that provider's own options class instead — `OpenAiChatOptions.builder()...`, `AnthropicChatOptions.builder()...` — which extends the portable interface and adds the provider-specific extras on top.

---

## 4. Configuring options — the three levels

### 4.1 `application.yml` — application-wide defaults

This is the baseline every request gets unless something overrides it in code. Property names accept either camelCase or kebab-case (Spring's relaxed binding):

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4o-mini
          temperature: 0.7
          top-p: 0.9
          max-tokens: 1024
          frequency-penalty: 0.3
          presence-penalty: 0.2
          stop:
            - "User:"
            - "###"
```

For Anthropic, the same idea (note `top-k` is meaningful here, `frequency-penalty`/`presence-penalty` are not, since Anthropic's API doesn't accept them):

```yaml
spring:
  ai:
    anthropic:
      api-key: ${ANTHROPIC_API_KEY}
      chat:
        options:
          model: claude-sonnet-4-5
          temperature: 0.5
          top-k: 40
          top-p: 0.9
          max-tokens: 2048
```

This yml-level configuration is what Spring Boot's auto-configuration uses to build the underlying `ChatModel` bean's default options — you don't write any Java for this layer to take effect.

### 4.2 Bean configuration level — `.defaultOptions(...)`

Set on `ChatClient.Builder` when you build your `ChatClient` bean, this overrides whatever the yml-level defaults were, for every request made through this specific client:

```java
@Configuration
class AiConfig {

    @Bean
    ChatClient chatClient(ChatClient.Builder builder) {
        return builder
                .defaultOptions(ChatOptions.builder()
                        .model("gpt-4o-mini")
                        .temperature(0.3)
                        .maxTokens(500)
                        .topP(0.8)
                        .frequencyPenalty(0.5)
                        .presencePenalty(0.3)
                        .stopSequences(List.of("###"))
                        .build())
                .build();
    }
}
```

You can also build a second `ChatClient` bean with a **different** set of defaults, for a different purpose — e.g. a low-temperature client for structured data extraction, and a high-temperature client for creative content — following the same purpose-named-bean pattern covered in the multi-model configuration guide:

```java
@Bean
ChatClient extractionChatClient(OpenAiChatModel model) {
    return ChatClient.builder(model)
            .defaultOptions(ChatOptions.builder().temperature(0.1).maxTokens(300).build())
            .build();
}

@Bean
ChatClient creativeChatClient(OpenAiChatModel model) {
    return ChatClient.builder(model)
            .defaultOptions(ChatOptions.builder().temperature(1.0).maxTokens(1500).build())
            .build();
}
```

### 4.3 Service level — `.options(...)` per call

This is the most specific, highest-priority override — it applies only to the single call it's attached to, replacing the bean-level defaults for whichever fields it sets:

```java
@Service
public class SummaryService {

    private final ChatClient chatClient;

    public SummaryService(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    public String summarizeStrictly(String longText) {
        return chatClient.prompt()
                .user("Summarize the following in exactly 3 bullet points:\n" + longText)
                .options(ChatOptions.builder()
                        .temperature(0.0)   // as deterministic as possible for this specific call
                        .maxTokens(150)     // keep the summary genuinely short
                        .build())
                .call()
                .content();
    }

    public String brainstormFreely(String topic) {
        return chatClient.prompt()
                .user("Brainstorm wild, unconventional ideas about: " + topic)
                .options(ChatOptions.builder()
                        .temperature(1.1)
                        .presencePenalty(0.6) // push toward covering more distinct ideas
                        .maxTokens(800)
                        .build())
                .call()
                .content();
    }
}
```

Two very different requests, same `ChatClient` bean, same underlying model — the actual generation behavior is entirely shaped by the `.options(...)` supplied per call.

### 4.4 Using a provider-specific options class when you need provider-only parameters

When a field lives outside the portable `ChatOptions` interface — like OpenAI's `seed` (for reproducible output) or Anthropic's extended `thinking` budget — reach for that provider's own options class, which is a superset of the portable one:

```java
public String deterministicExtraction(String input) {
    return chatClient.prompt()
            .user(input)
            .options(OpenAiChatOptions.builder()
                    .temperature(0.0)
                    .seed(42) // OpenAI-specific: same seed + same input → same output
                    .build())
            .call()
            .content();
}
```

---

## 5. How the three levels combine — precedence

From lowest to highest priority (highest wins, for any field it explicitly sets):

```
application.yml defaults (spring.ai.<provider>.chat.options.*)
        │  (used to build the provider's ChatModel's baseline options)
        ▼
Bean-level .defaultOptions(...) on ChatClient.Builder
        │  (overrides yml values, applies to every call through this ChatClient)
        ▼
Service-level .options(...) per call
        │  (overrides everything above, applies only to this one request)
        ▼
   Actual request sent to the model
```

A field left unset at a higher-priority level simply falls through to whatever the level below it specified — you don't have to repeat every field at the service level just to change one; only the fields you explicitly set in `.options(...)` override, the rest inherit from the bean/yml defaults underneath.

---

## 6. Best practices

- **Set the safe, general-purpose defaults in yml or at the bean level**, and reserve service-level `.options(...)` overrides for endpoints that genuinely need different behavior — a summarizer needing low temperature, a brainstorming endpoint needing high temperature, and so on.
- **Always set `maxTokens` deliberately, don't leave it unbounded**, especially on any endpoint exposed to end users — it's your primary defense against a single unexpectedly long (and expensive) response.
- **Don't tune `temperature` and `topP` simultaneously** unless you specifically understand the interaction — pick one as your main lever per use case.
- **Check the provider-support table (§3) before relying on a parameter** — setting `topK` against OpenAI, or a penalty against Anthropic, will silently do nothing rather than error, which can be a confusing debugging trip if you don't already know the limitation.
- **Reach for the provider-specific options class** (`OpenAiChatOptions`, `AnthropicChatOptions`, etc.) the moment you need something outside the portable set — don't try to force a provider-specific need through the generic `ChatOptions` interface.
- **Test extraction/classification-style prompts at low temperature and creative prompts at higher temperature explicitly** — the "right" value is genuinely task-dependent, and the default a provider ships with is rarely optimal for both ends of that spectrum at once.
