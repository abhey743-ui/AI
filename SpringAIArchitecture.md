# Spring AI Architecture — ChatClient, ChatModel, and Wiring It All Up

## 1. The big picture

Spring AI gives you **two APIs** that sit on top of each other. Almost every piece of confusion beginners have comes from not knowing which layer they're actually touching.

```
Your Controller / Service
          │
          ▼
      ChatClient            ← high-level, fluent, "developer experience" layer
          │
   (Advisors: memory, RAG, logging, tool-calling loop)
          │
          ▼
      ChatModel              ← low-level, portable contract (one impl per vendor)
          │
          ▼
  OpenAiChatModel / AnthropicChatModel / VertexAiGeminiChatModel / OllamaChatModel / ...
          │
          ▼
     Actual HTTP call to OpenAI / Anthropic / Gemini / local Ollama server / etc.
```

**One-sentence summary:** `ChatModel` is the plumbing that talks to a specific AI provider; `ChatClient` is the fluent front door you actually write your application code against, and it delegates down to a `ChatModel` to do the real work.

---

## 2. `ChatModel` — the low-level contract

`ChatModel` is an interface. Each supported provider has its own implementation:

| Provider | Implementation class |
|---|---|
| OpenAI / Azure OpenAI | `OpenAiChatModel`, `AzureOpenAiChatModel` |
| Anthropic | `AnthropicChatModel` |
| Google Gemini / Vertex | `VertexAiGeminiChatModel` |
| Ollama (local) | `OllamaChatModel` |
| Mistral | `MistralAiChatModel` |
| Amazon Bedrock | `BedrockProxyChatModel` / provider-specific Converse models |

**What it does, concretely:**
- Takes a `Prompt` object (a `List<Message>` + `ChatOptions`).
- Serializes it into whatever shape that vendor's REST API expects (this is where the role-translation from the previous guide happens — e.g. hoisting `SystemMessage` into Anthropic's separate `system` field).
- Makes the actual HTTP call.
- Deserializes the vendor's raw JSON response back into Spring AI's normalized `ChatResponse` (which wraps one or more `Generation` objects, each containing an `AssistantMessage`).

**When you'd use `ChatModel` directly:** you need fine-grained access to response metadata (token usage, finish reason, multiple candidate generations), you're building low-level infrastructure code, or you specifically don't want the extra behavior `ChatClient` adds.

```java
@RestController
class RawController {

    private final ChatModel chatModel;

    RawController(ChatModel chatModel) {
        this.chatModel = chatModel;
    }

    @GetMapping("/raw")
    String raw(@RequestParam String question) {
        Prompt prompt = new Prompt(new UserMessage(question));
        ChatResponse response = chatModel.call(prompt);
        return response.getResult().getOutput().getText();
    }
}
```

Notice how much manual work this is: build the `Prompt` yourself, call `.call()`, dig the text out of `ChatResponse` → `Generation` → `AssistantMessage` yourself.

---

## 3. `ChatClient` — the high-level fluent API

`ChatClient` is built **on top of** `ChatModel`. It's the recommended entry point for almost all application code — same relationship as `RestClient`/`WebClient` sitting on top of raw HTTP.

Spring Boot auto-configures a `ChatClient.Builder` for you (backed by whichever `ChatModel` bean is on the classpath), so you build your actual `ChatClient` once, typically as a `@Bean`:

```java
@Configuration
class AiConfig {

    @Bean
    ChatClient chatClient(ChatClient.Builder builder) {
        return builder
                .defaultSystem("You are a concise, helpful assistant.")
                .build();
    }
}
```

Then everywhere else, you just inject `ChatClient` and use the fluent API:

```java
@RestController
class ChatController {

    private final ChatClient chatClient;

    ChatController(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    @GetMapping("/chat")
    String chat(@RequestParam String question) {
        return chatClient.prompt()
                .user(question)
                .call()
                .content();
    }
}
```

**What `ChatClient` adds on top of `ChatModel`:**
- A fluent builder: `.system(...)`, `.user(...)`, `.messages(...)`, `.tools(...)`, `.options(...)`.
- **Advisors** — an interceptor chain that runs before/after the actual model call (see §4).
- Structured output mapping straight to a POJO: `.entity(MyRecord.class)` instead of parsing JSON by hand.
- Both synchronous (`.call()`) and streaming (`.stream()`) response modes.
- Automatic tool-calling orchestration (see §5) — you don't have to manually detect "the model wants to call a tool," execute it, and re-send the result; `ChatClient` loops that for you.

A subtlety that trips people up: calling `.call()` or `.stream()` **doesn't itself execute the request** — it just selects sync vs. streaming mode. The request is actually sent only when you chain a terminal operation like `.content()`, `.chatResponse()`, or `.entity()`.

---

## 4. How they interact — the Advisor chain

This is the mechanism that actually connects `ChatClient` to `ChatModel`. Every `ChatClient` request passes through an ordered chain of **Advisors** before it ever reaches the underlying `ChatModel`:

```
chatClient.prompt()...call()
        │
        ▼
 ┌─────────────────────────────────────────┐
 │            Advisor chain                 │
 │  MessageChatMemoryAdvisor (inject history)│
 │  QuestionAnswerAdvisor (RAG: inject docs) │
 │  SimpleLoggerAdvisor (log request/response)│
 │  Tool-calling loop (run tools, re-call)   │
 └─────────────────────────────────────────┘
        │
        ▼
     ChatModel.call(prompt)
        │
        ▼
   Vendor API (OpenAI / Anthropic / ...)
```

Each advisor can:
1. **Mutate the request before it's sent** — e.g. `MessageChatMemoryAdvisor` pulls prior conversation turns from a `ChatMemory` store and prepends them as `UserMessage`/`AssistantMessage` history; `QuestionAnswerAdvisor` runs a similarity search against a vector store and stuffs the retrieved context into the prompt (this is how RAG is implemented).
2. **Inspect/mutate the response after it comes back** — e.g. logging, or, for tool calling, noticing the model asked for a tool call, running it, and *re-entering the chain* with the tool result appended, looping until the model gives a final answer.

You attach advisors either as defaults on the builder, or per-request:

```java
ChatClient chatClient = builder
        .defaultAdvisors(
                MessageChatMemoryAdvisor.builder(chatMemory).build(),
                new SimpleLoggerAdvisor()
        )
        .build();
```

**Key takeaway:** `ChatModel` never knows about memory, RAG, or tool-calling orchestration — it just executes one `Prompt` → `ChatResponse` round trip. All of that "smart" behavior lives in the advisor chain that `ChatClient` runs *around* that single round trip.

---

## 5. Setting up the AI integration, step by step

### Step 1 — Add the starter dependency for your provider

Maven, for OpenAI:

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-openai</artifactId>
</dependency>
```

Swap the artifact for the provider you want — `spring-ai-starter-model-anthropic`, `spring-ai-starter-model-vertex-ai-gemini`, `spring-ai-starter-model-ollama`, etc. You also need Spring AI's BOM in `dependencyManagement` to pin compatible versions:

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-bom</artifactId>
            <version>1.0.x</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### Step 2 — Configure credentials

`application.properties` (or `application.yml`):

```properties
spring.ai.openai.api-key=${OPENAI_API_KEY}
spring.ai.openai.chat.options.model=gpt-4o-mini
spring.ai.openai.chat.options.temperature=0.7
```

Swapping providers later is mostly a matter of swapping the starter dependency + the `spring.ai.<provider>.*` properties — your `ChatClient`/`ChatModel` application code barely changes, because both are provider-agnostic abstractions.

### Step 3 — Let Spring Boot auto-configure the beans

Adding the starter is enough for Spring Boot to auto-configure:
- A `ChatModel` bean (e.g. `OpenAiChatModel`) wired with your credentials/options.
- A `ChatClient.Builder` bean, pre-wired to that `ChatModel`.

You don't construct either by hand in a typical app.

### Step 4 — Build your `ChatClient` bean with defaults

```java
@Configuration
class AiConfig {

    @Bean
    ChatClient chatClient(ChatClient.Builder builder, ChatMemory chatMemory) {
        return builder
                .defaultSystem("You are a helpful customer support agent.")
                .defaultAdvisors(MessageChatMemoryAdvisor.builder(chatMemory).build())
                .build();
    }
}
```

### Step 5 — Inject and use it

```java
@Service
class SupportAgent {

    private final ChatClient chatClient;

    SupportAgent(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    String ask(String question) {
        return chatClient.prompt()
                .user(question)
                .call()
                .content();
    }
}
```

That's a fully working AI integration: five steps, no manual HTTP/JSON handling.

---

## 6. Performing actions — tool calling (the "agentic" part)

So far the model can only *talk*. To let it **perform actions** — look up real data, call an API, write to a database, send an email — you expose Java methods as **tools** and register them with the `ChatClient`. The model then decides, based on the conversation, when to call one.

### 6.1 Define a tool

```java
@Component
class OrderTools {

    private final OrderRepository orderRepository;

    OrderTools(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    @Tool(description = "Get order details by order ID")
    public OrderDetails getOrder(@ToolParam(description = "The order ID") String orderId) {
        return orderRepository.findById(orderId);
    }

    @Tool(description = "Process a refund for a delivered order")
    public RefundResult refund(@ToolParam(description = "The order ID") String orderId) {
        return orderRepository.refund(orderId);
    }
}
```

Spring AI reads the `@Tool` description and the method signature (plus `@ToolParam` descriptions) and auto-generates the JSON schema the model needs to know the tool exists and how to call it.

### 6.2 Register the tool with `ChatClient`

```java
@Bean
ChatClient chatClient(ChatClient.Builder builder, OrderTools orderTools) {
    return builder
            .defaultSystem("You are a support agent. Only refund delivered orders.")
            .defaultTools(orderTools)
            .build();
}
```

Or per-request instead of as a default: `chatClient.prompt().tools(orderTools)...`

### 6.3 What actually happens under the hood

```
1. User: "Can I get a refund for order #4821?"
2. ChatClient sends the prompt to the model, *along with the tool schema* for getOrder/refund.
3. Model replies with an AssistantMessage containing a tool-call request:
       "call refund(orderId='4821')"
   (no natural-language answer yet — it wants data first, or wants to act)
4. Spring AI's tool-calling advisor intercepts this, actually invokes your
   OrderTools.refund("4821") Java method.
5. The return value is wrapped in a ToolResponseMessage and sent back to the
   model as part of the same logical conversation.
6. The model reads the tool result and produces the final natural-language
   reply: "Your refund for order #4821 has been processed."
7. ChatClient returns that final text to your calling code.
```

This request/execute/respond loop can repeat multiple times in a single `.call()` if the model needs to chain several tool calls before it can answer — you never write that loop yourself; it lives inside the advisor chain described in §4.

### 6.4 Important practical notes

- **Not every model supports tool calling.** Check the provider's capabilities before relying on it in production.
- Tool execution runs as **plain Java** on your server — the model never runs your code directly, it only requests that you run it and gives you the arguments. This means normal security/authorization checks still belong inside your `@Tool` method, exactly as if it were any other service method.
- Tool call arguments/results are not included in Micrometer observability spans by default (they may contain sensitive data); opt in explicitly with `spring.ai.tools.observations.include-content=true` if you want them traced.

---

## 7. Putting it all together — mental model recap

| Concept | Role |
|---|---|
| `ChatModel` | Provider-specific adapter: `Prompt` in, `ChatResponse` out, one HTTP round trip |
| `ChatClient` | Fluent façade over `ChatModel`; where you actually write application code |
| `Advisor` | Middleware around the `ChatClient` call: memory, RAG, logging, tool-calling loop |
| `@Tool` method | A Java method exposed to the model as something it can request be executed |
| Auto-configuration | Spring Boot wires the provider's `ChatModel` + a `ChatClient.Builder` for you once the starter is on the classpath |

Build against `ChatClient` for 95% of application code. Drop to `ChatModel` only when you need raw control. Add advisors for memory/RAG/logging. Add `@Tool` methods when the model needs to actually *do* something in your system, not just talk about it.
