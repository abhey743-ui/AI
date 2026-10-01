# Spring AI Chat Memory — `ChatMemory`, `ChatMemoryRepository`, and the Memory Advisor, End to End

## 1. The problem this solves

A language model API call is **stateless**. Every single request you send is evaluated in total isolation — the model has no idea a previous request from the same user ever happened, unless you explicitly re-send the earlier conversation as part of the current prompt. If you don't do that, every message feels like talking to someone with no short-term memory: ask "what's my name?" right after telling it your name, and it genuinely has no way to know, because nothing about the earlier exchange was sent this time.

A **message window** is the practical answer: keep the last N messages of a conversation around, and automatically re-inject them into every new request for that same conversation, so the model always sees enough recent history to respond coherently — without re-sending the *entire* history forever (which would eventually blow past the context window and cost you more on every single call).

Spring AI's chat memory system is built specifically to implement this pattern, and it does so by deliberately separating two different concerns into two different interfaces.

---

## 2. The internal architecture — two layers, two interfaces

```
┌─────────────────────────────────────────────────┐
│                  ChatMemory                       │
│   "What do we remember, and how much of it?"       │
│   — the window/truncation policy lives here        │
└───────────────────────┬─────────────────────────┘
                         │ delegates actual storage to
                         ▼
┌─────────────────────────────────────────────────┐
│              ChatMemoryRepository                  │
│   "Where do the messages physically live?"          │
│   — pure CRUD storage, no window logic at all       │
└─────────────────────────────────────────────────┘
```

This split matters: **`ChatMemory` decides policy** (how many messages to keep, when older ones get dropped), while **`ChatMemoryRepository` is pure storage** (save these messages, fetch these messages, delete these messages) with no opinion at all about how much should be kept. The same storage backend (say, a database table) can be used by different memory policies, and the same memory policy can sit on top of different storage backends — they're intentionally decoupled.

### 2.1 The `ChatMemory` interface — the policy layer

```java
public interface ChatMemory {

    String CONVERSATION_ID = "chat_memory_conversation_id";

    void add(String conversationId, Message message);

    void add(String conversationId, List<Message> messages);

    List<Message> get(String conversationId);

    void clear(String conversationId);
}
```

- **`add(...)`** — appends one or more new messages to a given conversation's history.
- **`get(conversationId)`** — retrieves the stored message history for that conversation, **already filtered according to whatever policy the implementation enforces** (e.g. only the last 20 messages, not necessarily everything ever stored).
- **`clear(conversationId)`** — wipes a conversation's history entirely — what you'd call when a user starts a fresh chat or explicitly asks to forget the conversation.
- **`CONVERSATION_ID`** is a constant string key, used specifically as the parameter name advisors look for when you need to tell the memory system *which* conversation a given request belongs to (more in §5).

### 2.2 The `ChatMemoryRepository` interface — the storage layer

```java
public interface ChatMemoryRepository {

    List<String> findConversationIds();

    List<Message> findByConversationId(String conversationId);

    void saveAll(String conversationId, List<Message> messages);

    void deleteByConversationId(String conversationId);
}
```

Notice this interface has **no concept of a window or a limit at all** — `findByConversationId` just returns whatever is stored, full stop. It's a plain storage contract, conceptually no different from a `CrudRepository` for any other entity — the "how much to keep" decision explicitly does **not** live here; it lives one layer up, in `ChatMemory`.

---

## 3. The default implementations

### 3.1 `MessageWindowChatMemory` — the default `ChatMemory` implementation

This is the policy Spring AI ships out of the box, and the one auto-configured for you unless you say otherwise. It implements a **sliding window**: it keeps up to a configured maximum number of messages per conversation, and once that limit is exceeded, the oldest messages are dropped to make room for new ones — exactly like a fixed-size queue.

```java
ChatMemory chatMemory = MessageWindowChatMemory.builder()
        .maxMessages(20)                        // the sliding window size; 20 is the documented default
        .chatMemoryRepository(new InMemoryChatMemoryRepository())
        .build();
```

Internally, every call to `add(...)` stores the new message(s) via the injected `ChatMemoryRepository`, then checks whether the conversation's total stored message count now exceeds `maxMessages` — if so, it trims from the oldest end before the next `get(...)` call returns the (already-trimmed) history. This is precisely the "read-process-write" orchestration that the plain storage layer doesn't know anything about.

### 3.2 `InMemoryChatMemoryRepository` — the default storage implementation

A simple, in-process map-backed implementation — conversation history lives purely in application memory, for as long as the JVM process is running. If the app restarts, every conversation's history is gone. This is exactly what Spring AI auto-configures for you by default, with zero extra setup, which is great for development and for apps that genuinely don't need conversations to survive a restart.

```java
ChatMemoryRepository repository = new InMemoryChatMemoryRepository();
```

**Its real limitation:** it doesn't scale across multiple application instances (each instance has its own separate in-memory map — a user's conversation "disappears" if their next request lands on a different instance behind a load balancer), and it has no durability at all. For anything production-grade with persistence or multi-instance requirements, you reach for a real backing store instead — most commonly, `JdbcChatMemoryRepository`.

### 3.3 `JdbcChatMemoryRepository` — a persistent, database-backed storage implementation

Same `ChatMemoryRepository` contract, but messages are actually persisted to a relational database via JDBC, surviving restarts and working correctly across multiple application instances sharing the same database.

```java
ChatMemoryRepository chatMemoryRepository = JdbcChatMemoryRepository.builder()
        .jdbcTemplate(jdbcTemplate)
        .dialect(new PostgresChatMemoryRepositoryDialect()) // or the dialect matching your DB
        .build();
```

Spring AI supports multiple relational databases through a **dialect abstraction** (there's a `JdbcChatMemoryRepositoryDialect` interface you can implement for a database not already supported out of the box), and ships a table it manages for you — `SPRING_AI_CHAT_MEMORY` — to actually hold the stored messages.

**One concrete limitation worth knowing:** `JdbcChatMemoryRepository` does not support storing tool-call messages — if your conversation involves tool calling (covered in the earlier architecture guide), keep that in mind when choosing this backend for an agentic use case.

---

## 4. Dependencies — exactly what you need, per implementation

### 4.1 Just the `ChatMemory` abstraction (default in-memory behavior)

If all you want is the default — `MessageWindowChatMemory` backed by `InMemoryChatMemoryRepository` — you need the chat memory starter, which is what brings in the auto-configuration that wires up the default `ChatMemory` bean for you:

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-chat-memory</artifactId>
</dependency>
```

With just this (and your usual model starter), Spring Boot auto-configures a ready-to-inject `ChatMemory` bean — no `@Bean` method required on your part unless you want to override the defaults.

### 4.2 Persistent storage via JDBC

Swap/add the JDBC-backed repository starter:

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-chat-memory-repository-jdbc</artifactId>
</dependency>
```

This starter brings in the `JdbcChatMemoryRepository` auto-configuration, but **you still need an actual JDBC driver** for whichever database you're pointing at, since the starter itself doesn't bundle one. For a quick local/demo setup with H2:

```xml
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
</dependency>
```

Swap `h2` for `postgresql`, `mysql-connector-j`, etc. for a real database, matched with the corresponding `JdbcChatMemoryRepositoryDialect` if you're not using one of the dialects Spring AI already ships support for.

---

## 5. `application.yml` — configuration for the JDBC repository

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:chatmemorydb
    driver-class-name: org.h2.Driver
    username: sa
    password:

  ai:
    chat:
      memory:
        repository:
          jdbc:
            initialize-schema: always   # creates the SPRING_AI_CHAT_MEMORY table on startup
            # schema: classpath:custom-schema.sql   # override the default schema script if needed
            # platform: postgresql                   # override auto-detected DB platform if needed
```

`initialize-schema` is the property that controls whether Spring AI creates the `SPRING_AI_CHAT_MEMORY` table for you automatically on startup — `always` is convenient for development; in a real production setup you'd typically manage that table via your own migration tooling (Flyway/Liquibase) instead and leave this unset or `never`.

---

## 6. Bean-level configuration

### 6.1 Just accepting the auto-configured default

With the starter from §4.1 on the classpath and nothing else configured, you can inject `ChatMemory` directly — Spring Boot already built it for you as `MessageWindowChatMemory` + `InMemoryChatMemoryRepository`:

```java
@Service
class SomeService {
    private final ChatMemory chatMemory; // already auto-configured, ready to use
    SomeService(ChatMemory chatMemory) { this.chatMemory = chatMemory; }
}
```

### 6.2 Customizing the window size, still in-memory

```java
@Configuration
class ChatMemoryConfig {

    @Bean
    ChatMemory chatMemory() {
        return MessageWindowChatMemory.builder()
                .maxMessages(50) // override the default of 20
                .build();        // no .chatMemoryRepository(...) call → defaults to InMemoryChatMemoryRepository
    }
}
```

### 6.3 Swapping in persistent JDBC-backed storage

```java
@Configuration
class ChatMemoryConfig {

    @Bean
    ChatMemory chatMemory(ChatMemoryRepository chatMemoryRepository) {
        return MessageWindowChatMemory.builder()
                .chatMemoryRepository(chatMemoryRepository) // the JdbcChatMemoryRepository auto-configured from §4.2/§5
                .maxMessages(30)
                .build();
    }
}
```

Note that `chatMemoryRepository` here is injected, not built by hand — with the JDBC starter and matching `application.yml` properties in place, Spring Boot already auto-configures a `JdbcChatMemoryRepository` bean for you; you're just wiring it into your custom `MessageWindowChatMemory` bean.

---

## 7. Service-level usage — the manual, low-level way

You can use `ChatMemory` entirely by hand, without any advisor at all, orchestrating the read-add-write cycle yourself — this is worth understanding because it's exactly what the advisor automates for you under the hood:

```java
@Service
class ManualMemoryChatService {

    private final ChatModel chatModel;
    private final ChatMemory chatMemory;

    ManualMemoryChatService(ChatModel chatModel, ChatMemory chatMemory) {
        this.chatModel = chatModel;
        this.chatMemory = chatMemory;
    }

    String chat(String conversationId, String userText) {

        chatMemory.add(conversationId, new UserMessage(userText));

        ChatResponse response = chatModel.call(new Prompt(chatMemory.get(conversationId)));

        chatMemory.add(conversationId, response.getResult().getOutput());

        return response.getResult().getOutput().getText();
    }

    void forget(String conversationId) {
        chatMemory.clear(conversationId);
    }
}
```

Walk through what's happening: the new user message is stored first, then the **entire currently-remembered window** for that conversation (not just the new message) is sent to the model as the prompt, and finally the model's own reply is stored back into memory too — so the *next* call sees this exchange as part of its history. This manual loop is tedious to repeat in every service method, which is exactly the gap the advisor closes.

---

## 8. The advisor — `MessageChatMemoryAdvisor`

Rather than manually calling `chatMemory.add(...)`/`.get(...)` in every service method, register `MessageChatMemoryAdvisor` once, and it performs that exact read-before/write-after cycle automatically as part of the advisor chain (recall the advisors guide: logic wrapped around `chain.nextCall(...)`).

> **Naming note:** an older class, `PromptChatMemoryAdvisor`, did a similar job but is deprecated since Spring AI 1.1.3 in favor of `MessageChatMemoryAdvisor` — use the latter in new code.

### 8.1 Registering it as a bean-level default

```java
@Configuration
class AiConfig {

    @Bean
    ChatClient chatClient(ChatClient.Builder builder, ChatMemory chatMemory) {
        return builder
                .defaultAdvisors(MessageChatMemoryAdvisor.builder(chatMemory).build())
                .build();
    }
}
```

Every request through this `ChatClient` now automatically has its conversation history injected before the call, and the exchange is automatically saved back after — exactly like §7's manual loop, but you never write that loop yourself again.

### 8.2 Telling it *which* conversation, per call

The advisor needs to know which conversation's history to load — that's what `ChatMemory.CONVERSATION_ID` is for, passed as a per-call advisor parameter (covered in the advisors guide's §5.1):

```java
@Service
class MemoryBackedChatService {

    private final ChatClient chatClient;

    MemoryBackedChatService(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    String chat(String conversationId, String userText) {
        return chatClient.prompt()
                .user(userText)
                .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, conversationId))
                .call()
                .content();
    }
}
```

No manual `add`/`get`/`clear` calls anywhere in this method — the advisor, registered once at the bean level, handles all of it on every call, keyed by whatever `conversationId` you pass in.

---

## 9. Putting it all together — a full, real example

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-openai</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-chat-memory-repository-jdbc</artifactId>
</dependency>
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
</dependency>
```

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:h2:mem:chatmemorydb
    driver-class-name: org.h2.Driver
    username: sa
    password:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4o-mini
    chat:
      memory:
        repository:
          jdbc:
            initialize-schema: always
```

```java
@Configuration
class AiConfig {

    @Bean
    ChatMemory chatMemory(ChatMemoryRepository chatMemoryRepository) {
        return MessageWindowChatMemory.builder()
                .chatMemoryRepository(chatMemoryRepository) // auto-configured JdbcChatMemoryRepository
                .maxMessages(25)
                .build();
    }

    @Bean
    ChatClient chatClient(ChatClient.Builder builder, ChatMemory chatMemory) {
        return builder
                .defaultSystem("You are a helpful assistant. Keep track of what the user has told you.")
                .defaultAdvisors(MessageChatMemoryAdvisor.builder(chatMemory).build())
                .build();
    }
}
```

```java
@RestController
@RequestMapping("/api/chat")
class ChatController {

    private final ChatClient chatClient;
    private final ChatMemory chatMemory;

    ChatController(ChatClient chatClient, ChatMemory chatMemory) {
        this.chatClient = chatClient;
        this.chatMemory = chatMemory;
    }

    @PostMapping("/{conversationId}")
    String chat(@PathVariable String conversationId, @RequestBody String message) {
        return chatClient.prompt()
                .user(message)
                .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, conversationId))
                .call()
                .content();
    }

    @DeleteMapping("/{conversationId}")
    void forget(@PathVariable String conversationId) {
        chatMemory.clear(conversationId);
    }
}
```

With this in place: `POST /api/chat/session-42` with `"My name is Dave"`, followed later by `POST /api/chat/session-42` with `"What's my name?"`, correctly gets back an answer referencing Dave — because the advisor transparently re-injected the stored history for `session-42` on the second call. `DELETE /api/chat/session-42` wipes that conversation clean, and a fresh `conversationId` for a different user starts with a completely empty window, isolated from everyone else's history.

---

## 10. Best practices

- **Let auto-configuration do the work for the common case.** Unless you need a custom window size or persistent storage, you don't need to write any `ChatMemory`/`ChatMemoryRepository` bean at all — the default is already wired up.
- **Pick a `conversationId` strategy deliberately.** A session ID, a user ID, or a composite of both — whatever it is, it's the only thing that scopes one user's history away from another's; get this wrong and conversations bleed into each other.
- **Move to `JdbcChatMemoryRepository` (or another persistent backend) the moment you have more than one application instance, or need history to survive a restart** — `InMemoryChatMemoryRepository` is fine for development, not for a real multi-instance deployment.
- **Size `maxMessages` deliberately, not arbitrarily.** A larger window means the model sees more context (better continuity) but also means a bigger prompt on every single call (more cost, more latency) — tune it against your actual conversations' typical length and your cost/latency budget.
- **Remember `ChatMemoryRepository` has no window logic of its own.** If you ever bypass `MessageWindowChatMemory` and talk to a `ChatMemoryRepository` directly, you're responsible for any truncation yourself — the repository will happily return an ever-growing, unbounded history.
- **Give users (or your own cleanup job) a way to call `clear(conversationId)`.** Conversations that are truly done should be explicitly cleared, both for privacy and to avoid needlessly growing storage forever.
- **Watch the tool-call limitation on `JdbcChatMemoryRepository`** if your conversations involve tool calling — verify this still fits your use case before committing to it as your persistent backend for an agentic flow.
