# ToolSearchToolCallingAdvisor + MCP Client — Complete Guide

> Goal of this file: you should understand **why** this advisor exists, **how a request flows inside it**, **what to put in `pom.xml` and `application.yml`**, and **how to optimize it** when you use many MCP tools.
>
> Written for **Spring AI 2.0.x** (2.0.1 is the current stable line). In Spring AI 1.x this advisor does not exist in the same form.

---

## Table of Contents

1. The problem in simple words
2. The idea: "search first, call second"
3. Who is who (the actors)
4. How it works inside — the real flow
5. Full example, round by round (what the LLM actually sees)
6. Dependencies (pom.xml)
7. Configuration (application.yml) — every setting explained
8. The MCP client side — connecting servers
9. Complete working code (end to end)
10. The 3 search strategies (Regex / Lucene / Vector) and which to pick
11. Session ID — the most common mistake
12. Memory of discovered tools and eviction
13. How to optimize (checklist)
14. Limitations you should know
15. Debugging and troubleshooting
16. Quick summary / cheat sheet

---

## 1. The problem in simple words

When you use MCP, your Spring app (the MCP **client**) connects to MCP **servers** (GitHub, Slack, Jira, database, files…). Every server exposes many tools.

With the normal `ToolCallingAdvisor`, every request to the LLM carries **the full definition of every tool**: name, description, and the JSON schema of the parameters.

```
Request to LLM =  user question
                + system prompt
                + definition of tool 1
                + definition of tool 2
                + ...
                + definition of tool 80      <-- all of them, every single time
```

This causes three problems:

| Problem | Why it hurts |
|---|---|
| **Cost** | You pay for those tokens on every request. The Spring docs say a multi-server setup can easily have 50+ tools using 55,000+ tokens before the conversation even starts. |
| **Accuracy** | When the model sees 30+ similarly-named tools, it picks the wrong one more often. |
| **Context space** | Tool definitions eat the context window that you want to use for real conversation. |

---

## 2. The idea: "search first, call second"

`ToolSearchToolCallingAdvisor` fixes this with a simple trick (called *progressive tool disclosure*, the "Tool Search Tool" pattern):

> Do **not** show all tools to the LLM. Show **only one tool**: a search tool.
> When the LLM needs something, it **searches** for tools. Only the matching tool definitions are then shown to it.

Think of a library:

- **Normal way:** you hand the visitor the whole catalogue of 5000 books and say "pick one".
- **Tool search way:** you give the visitor a **search desk**. They ask "I need something about Jira tickets", and you hand over only 3 matching books.

Spring AI's version works with **any provider** (OpenAI, Anthropic, Gemini, Ollama…) because the search happens **inside your Spring app**, not inside the model provider. The reference docs report 34–64% fewer tokens in their benchmarks (28 tools: Gemini 60%, OpenAI 34%, Anthropic 64%). Your own number will depend on your tools.

---

## 3. Who is who (the actors)

| Actor | What it is | Role |
|---|---|---|
| **User** | Person / frontend calling your REST API | Asks the question |
| **Your Spring Boot app** | Contains `ChatClient` + advisors + MCP client | The "brain controller" |
| **`ChatClient`** | Spring AI fluent API | Entry point for the request |
| **`ToolSearchToolCallingAdvisor`** | Replaces the normal `ToolCallingAdvisor` | Runs the tool loop **and** hides/reveals tools |
| **`ToolIndex`** | Search engine for tools (Regex, Lucene or Vector) | Stores all tool info and answers searches |
| **`toolSearchTool`** | Built-in tool added by the advisor | The only tool the LLM sees at first |
| **`ToolCallingManager`** | Spring AI component | Actually executes the tool callback |
| **`ToolCallback` (MCP)** | Proxy object for one MCP tool | When called, it sends a request to the MCP server |
| **MCP Client** (`McpSyncClient`) | Connection to one MCP server | Talks the MCP protocol |
| **MCP Server** | Remote program (GitHub, Slack…) | Really performs the action |
| **LLM** | OpenAI / Claude / Gemini… | Decides what to call and writes the final answer |

Important idea: **the LLM never calls a tool by itself.** It only *asks* ("please call tool X with these arguments"). Your Spring app does the real call and sends the result back.

---

## 4. How it works inside — the real flow

### 4.1 What the advisor does (7 steps from the official docs)

1. **Indexing** — at the start of a session, all registered tools are put into the `ToolIndex`. **No tool definition is sent to the LLM.**
2. **Initial request** — the first request to the LLM contains only the built-in `toolSearchTool` definition (plus a system-message instruction telling the model to search before guessing).
3. **Discovery call** — when the model needs some capability, it calls `toolSearchTool` with a natural-language query.
4. **Search & expand** — the `ToolIndex` finds matching tools. Their definitions are added for the next iteration.
5. **Tool invocation** — now the model sees the real tool definition and makes a normal tool call.
6. **Tool execution** — `ToolCallingManager` runs the tool and returns the result.
7. **Response** — the model uses the result to write the final answer.

### 4.2 Sequence diagram (User → Client → LLM → Client → User)

```
 User        Your App (ChatClient + Advisor)      ToolIndex      LLM          MCP Server
  |                      |                           |            |               |
  |-- "create Jira       |                           |            |               |
  |    ticket + tell     |                           |            |               |
  |    team on Slack" -->|                           |            |               |
  |                      |                           |            |               |
  |         [A] Advisor starts. Takes ALL registered tools          |               |
  |             (local @Tool + MCP tools) and indexes them          |               |
  |                      |--- index(sessionId, tools)->            |               |
  |                      |                           |            |               |
  |         [B] REQUEST 1 to LLM: question + ONLY toolSearchTool    |               |
  |                      |-------------------------------------->|               |
  |                      |                           |            |               |
  |         [C] LLM replies: "call toolSearchTool(query='jira ticket')"            |
  |                      |<--------------------------------------|               |
  |                      |                           |            |               |
  |         [D] Advisor runs the search locally       |            |               |
  |                      |--- search("jira ticket") ->|            |               |
  |                      |<-- [jira_create_issue] ----|            |               |
  |                      |                           |            |               |
  |         [E] REQUEST 2 to LLM: history + toolSearchTool + jira_create_issue def  |
  |                      |-------------------------------------->|               |
  |                      |                           |            |               |
  |         [F] LLM replies: "call jira_create_issue(args...)"      |               |
  |                      |<--------------------------------------|               |
  |                      |                           |            |               |
  |         [G] ToolCallingManager -> MCP ToolCallback -> MCP client|               |
  |                      |------------------------------------------------------->|
  |                      |<------------------ result: ISSUE-123 ------------------|
  |                      |                           |            |               |
  |         [H] REQUEST 3 to LLM: history + tool result (+ same tools)              |
  |                      |-------------------------------------->|               |
  |                      |       (LLM may search again for "slack message" ...     |
  |                      |        and repeat steps C-G for the Slack tool)         |
  |                      |                           |            |               |
  |         [I] FINAL: LLM returns plain text answer (no tool calls)                |
  |                      |<--------------------------------------|               |
  |<-- "Created ISSUE-123 and notified #team" --|     |            |               |
```

### 4.3 The loop idea

`ToolSearchToolCallingAdvisor` **extends** `ToolCallingAdvisor`, and `ToolCallingAdvisor` is a **recursive advisor**. That means:

```
while (the LLM response contains tool calls) {
    run the tools
    add the results to the conversation
    call the LLM again
}
return the final text answer
```

The search advisor changes two parts of this loop:

- **Initialization hook** (start of the loop): index all tools, send only `toolSearchTool`.
- **Per-iteration hook** (before every LLM call): rebuild the tool list = `toolSearchTool` + the tools that were found by searches.

So one user question becomes **several LLM calls** (search round-trip(s) + tool call round-trip(s) + final answer). That extra round trip is the price you pay for saving tokens — remember this for the optimization section.

### 4.4 Where does it sit in the advisor chain?

When you use the Spring Boot auto-configuration, its order is `HIGHEST_PRECEDENCE + 300` (property `advisor-order`). Advisors with a lower order number run earlier. If you also use memory advisors, keep this in mind when you read logs (section 15).

---

## 5. Full example, round by round (what the LLM actually sees)

**Setup:** an "Engineering Assistant". You have 3 MCP servers connected: `github`, `jira`, `slack`. Together they expose about 60 tools. (Tool names below are illustrative; real names come from your servers.)

**User question:**
> "Our nightly build failed. Create a Jira ticket for it and post the ticket link in the #dev-team Slack channel."

### Without the search advisor (normal `ToolCallingAdvisor`)

```
Request 1 → LLM sees: question + 60 full tool definitions   (~20k+ tokens)
Request 2 → LLM sees: question + 60 tool definitions + result of jira call
Request 3 → LLM sees: question + 60 tool definitions + results of jira + slack
```
Every request repeats the 60 definitions.

### With `ToolSearchToolCallingAdvisor`

**Round 0 — indexing (inside your app, no LLM call):**
All 60 tools go into the `ToolIndex` under this session's ID.

**Round 1 — request to LLM:**
```
system:  "...you have a toolSearchTool. Search for tools before using them.
          Do not guess a tool name that was not returned by a search."
user:    "Our nightly build failed. Create a Jira ticket ... Slack ..."
tools:   [ toolSearchTool ]            <-- only 1 tool, tiny
```
LLM answers with a tool call:
```
toolSearchTool(query = "create jira ticket issue")
```

**Round 2 — advisor handles the search locally:**
`ToolIndex.search(...)` returns e.g. `jira_create_issue`. The advisor stores this name. It does **not** call the MCP server. It just prepares the next request.

**Round 2 — request to LLM:**
```
messages: [system, user, assistant(toolSearchTool call), tool(search result)]
tools:    [ toolSearchTool, jira_create_issue ]    <-- full definition of 1 tool added
```
LLM answers:
```
jira_create_issue(project="BUILD", summary="Nightly build failed", ...)
```

**Round 3 — execution:**
`ToolCallingManager` finds the MCP `ToolCallback` for `jira_create_issue`, which calls the Jira MCP server. Result: `ISSUE-123`.

**Round 3 — request to LLM:**
```
messages: [... previous ..., assistant(jira call), tool(result ISSUE-123)]
tools:    [ toolSearchTool, jira_create_issue ]
```
LLM answers with a *new* search because it still needs Slack:
```
toolSearchTool(query = "post message slack channel")
```

**Round 4:** the index returns `slack_post_message`. Next request has
`tools: [ toolSearchTool, jira_create_issue, slack_post_message ]`
(the Jira tool stays because `referenceToolNameAccumulation` defaults to `true`).

**Round 5:** LLM calls `slack_post_message(channel="#dev-team", text="Build failed. Ticket: ISSUE-123")` → executed through the Slack MCP server → result.

**Round 6:** LLM sees the result and answers in plain text:
> "Done. I created ISSUE-123 and posted it in #dev-team."

That text goes back through the advisor chain → `ChatClient` → your controller → the user.

**Token picture:** in every round the LLM saw 1–3 tool definitions instead of 60. But you made ~6 LLM calls instead of ~3. That trade-off is why the docs recommend this only for **bigger tool catalogs**.

---

## 6. Dependencies (pom.xml)

### Option A (recommended): the Spring Boot starter

Includes the advisor, the `ToolIndex` API, Apache Lucene and auto-configuration:

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-tool-search-advisor</artifactId>
</dependency>
```

### Option B: library only (manual `@Bean` wiring)

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-tool-search-advisor</artifactId>
</dependency>
```

If you only need the `ToolIndex` interfaces and implementations:

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-tool-search-tool</artifactId>
</dependency>
```

> **Version tip:** one third-party article states these tool-search artifacts are versioned separately and not covered by `spring-ai-bom` (both at 2.0.1 when written). If Maven says "version missing", add `<version>2.0.1</version>` explicitly (or the version matching your Spring AI line). Check Maven Central to be sure.

### The other dependencies you need

```xml
<!-- MCP client. Pick ONE of these two -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-mcp-client</artifactId>          <!-- JDK HttpClient + STDIO -->
</dependency>
<!-- production recommendation from the docs: -->
<!--
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-mcp-client-webflux</artifactId>  
</dependency>
-->

<!-- Your chat model, for example OpenAI -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-openai</artifactId>
</dependency>

<!-- Only if you choose tool-index-type: vector -->
<!-- a VectorStore starter (for example pgvector) + an embedding model -->
```

Baseline: Spring AI 2.0 uses a **Spring Boot 4** baseline.

---

## 7. Configuration (application.yml) — every setting explained

### 7.1 Minimum setup (just turn it on)

```yaml
spring:
  ai:
    chat:
      client:
        tool-search-advisor:
          enabled: true
```

What `enabled: true` does:

- Registers a `ToolSearchToolCallingAdvisor` builder that **replaces the default `ToolCallingAdvisor`** (it works because the default one is guarded by `@ConditionalOnMissingBean`). You do **not** need to add the advisor yourself in the `ChatClient`.
- Auto-registers a `ToolIndex` bean (unless you define your own).

### 7.2 All properties

Prefix: `spring.ai.chat.client.tool-search-advisor`

| Property | Meaning | Default |
|---|---|---|
| `enabled` | Turn the advisor on (it replaces the default `ToolCallingAdvisor`) | `false` |
| `tool-index-type` | Which search engine: `regex`, `lucene`, `vector` | `regex` |
| `max-results` | Max tools returned per search call. `null` means the LLM decides (the built-in tool description hints at 5) | `null` |
| `system-message-suffix` | Your own text appended to the system message, to teach the model how to use `toolSearchTool`. `null` = built-in template | `null` |
| `reference-tool-name-accumulation` | `true` = tools found in earlier searches stay available in later turns. `false` = only the most recent turn's results are used | `true` |
| `session-id-key-name` | Advisor-context key that carries the session/conversation ID | `chat_memory_conversation_id` |
| `advisor-order` | Position in the advisor chain | `HIGHEST_PRECEDENCE + 300` |
| `eviction.lru-max-sessions` | Max sessions kept in memory (LRU) | `1000` |
| `eviction.ttl` | Idle time before a session index is dropped (e.g. `30m`, `1h`). When set, LRU + TTL are combined | `null` |
| `lucene.min-score-threshold` | Minimum Lucene score for a hit to be returned (only for `lucene`) | `0.25` |

### 7.3 Recommended production-style config

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}

    chat:
      client:
        tool-search-advisor:
          enabled: true
          tool-index-type: lucene        # keyword search, no embedding cost
          max-results: 5                 # keep the expanded tool list small
          reference-tool-name-accumulation: true
          eviction:
            lru-max-sessions: 500
            ttl: 30m
          lucene:
            min-score-threshold: 0.25

    mcp:
      client:
        enabled: true
        name: engineering-assistant
        version: 1.0.0
        type: SYNC                       # do not mix SYNC and ASYNC
        request-timeout: 30s
        toolcallback:
          enabled: true                  # default; creates the ToolCallbackProvider
        streamable-http:
          connections:
            github:
              url: http://localhost:8081
              endpoint: /mcp             # default is /mcp
            jira:
              url: http://localhost:8082
        sse:
          connections:
            slack:
              url: http://localhost:8083
              sse-endpoint: /sse         # default is /sse
```

### 7.4 Other config examples

**Semantic (vector) search** — needs a `VectorStore` bean:
```yaml
spring.ai.chat.client.tool-search-advisor.enabled: true
spring.ai.chat.client.tool-search-advisor.tool-index-type: vector
```

**Multi-tenant — session id comes from a different key:**
```yaml
spring.ai.chat.client.tool-search-advisor.enabled: true
spring.ai.chat.client.tool-search-advisor.session-id-key-name: tenantId
```

---

## 8. The MCP client side — connecting servers

### 8.1 What the MCP starter does for you

- Connects to each configured MCP server (each connection = one MCP client instance).
- Discovers the tools on each server.
- Wraps each remote tool in a `ToolCallback` (a local proxy).
- Exposes all of them as one `ToolCallbackProvider` bean (`SyncMcpToolCallbackProvider` for SYNC, async counterpart for ASYNC).

When the LLM asks for `jira_create_issue`, the proxy sends the MCP request to the Jira server and returns the result.

### 8.2 Transports

| Transport | Config prefix | Notes |
|---|---|---|
| Streamable HTTP | `spring.ai.mcp.client.streamable-http.connections.<name>` | `url`, `endpoint` (default `/mcp`). Default in Spring AI 2.0 |
| SSE | `spring.ai.mcp.client.sse.connections.<name>` | `url`, `sse-endpoint` (default `/sse`) |
| STDIO | `spring.ai.mcp.client.stdio.connections.<name>` | `command`, `args`, `env`; or `servers-configuration: classpath:mcp-servers.json` |

On **Windows**, STDIO commands like `npx` need `cmd.exe` with `/c` as the first argument, because Java `ProcessBuilder` cannot run `.cmd` batch files directly.

### 8.3 Features that matter for tool search

- **Tool name prefixes.** If two servers have a tool with the same name, `DefaultMcpToolNamePrefixGenerator` renames duplicates (e.g. `alt_1_search`), replaces special characters with `_`, and keeps names to max 64 chars. These **final names** are what gets indexed and what the LLM sees. Using `McpToolNamePrefixGenerator.noPrefix()` with duplicate names throws an `IllegalStateException`.
- **Tool filter (`McpToolFilter`).** You can remove tools **before** they reach the advisor. This is a very strong optimization: a smaller catalog = better search and cleaner results. Only one `McpToolFilter` bean is allowed.

```java
@Component
public class SafeToolFilter implements McpToolFilter {
    @Override
    public boolean test(McpConnectionInfo connectionInfo, McpSchema.Tool tool) {
        // never expose dangerous or useless tools to the model
        if (tool.name().startsWith("admin_")) return false;
        if (tool.description() != null && tool.description().contains("experimental")) return false;
        return true;
    }
}
```

---

## 9. Complete working code (end to end)

### 9.1 Project layout

```
src/main/java/com/example/assistant/
 ├── AssistantApplication.java
 ├── config/ChatConfig.java
 ├── tools/LocalTools.java        <- optional local @Tool methods
 ├── service/AssistantService.java
 └── web/AssistantController.java
src/main/resources/application.yml
```

### 9.2 Local tools (optional — they get indexed together with MCP tools)

```java
@Component
public class LocalTools {

    @Tool(description = "Get the current date and time in ISO format")
    public String currentTime() {
        return java.time.LocalDateTime.now().toString();
    }

    @Tool(description = "Look up the on-call engineer for a given team name")
    public String onCall(@ToolParam(description = "Team name, e.g. backend") String team) {
        return "alice@company.com"; // demo value
    }
}
```

### 9.3 ChatClient setup — with auto-configuration (`enabled: true`)

```java
@Configuration
public class ChatConfig {

    @Bean
    ChatClient chatClient(ChatClient.Builder builder,
                          SyncMcpToolCallbackProvider mcpTools,
                          LocalTools localTools,
                          ChatMemory chatMemory) {

        return builder
                .defaultSystem("""
                        You are an engineering assistant.
                        Use tools to perform actions. Always search for the right tool first.
                        """)
                .defaultToolCallbacks(mcpTools)          // all MCP tools (will be indexed, not sent)
                .defaultTools(localTools)                // local tools (also indexed)
                .defaultAdvisors(
                        MessageChatMemoryAdvisor.builder(chatMemory).build()  // gives conversation id support
                )
                .build();
    }
}
```

Notes:
- Because `enabled: true`, the injected `ChatClient.Builder` already uses the tool-search advisor instead of the plain `ToolCallingAdvisor`. If you build the client by hand with `ChatClient.builder(chatModel)`, add the advisor yourself (next section) to be safe.
- Do **not** register both the default tool advisor and the search advisor yourself.

### 9.4 ChatClient setup — manual (no auto-configuration)

Use only the library dependency, then:

```java
@Configuration
public class ManualToolSearchConfig {

    @Bean
    ToolIndex toolIndex() {
        return new LuceneToolIndex();               // or: new LuceneToolIndex(0.4f)
    }

    @Bean
    ChatClient chatClient(ChatModel chatModel,
                          ToolIndex toolIndex,
                          SyncMcpToolCallbackProvider mcpTools) {

        var toolSearchAdvisor = ToolSearchToolCallingAdvisor.builder()
                .toolIndex(toolIndex)                      // required
                .maxResults(5)
                .evictionStrategy(new CompositeEvictionStrategy(
                        new TtlEvictionStrategy(Duration.ofMinutes(30)),
                        new LruEvictionStrategy(500)))
                .build();

        return ChatClient.builder(chatModel)
                .defaultToolCallbacks(mcpTools)
                .defaultAdvisors(toolSearchAdvisor)
                .build();
    }
}
```

### 9.5 Service — **always pass a conversation ID**

```java
@Service
public class AssistantService {

    private final ChatClient chatClient;

    public AssistantService(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    public String ask(String conversationId, String question) {
        return chatClient.prompt()
                .user(question)
                .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, conversationId)) // REQUIRED
                .call()
                .content();
    }
}
```

### 9.6 Controller

```java
@RestController
@RequestMapping("/api/assistant")
public class AssistantController {

    private final AssistantService service;

    public AssistantController(AssistantService service) {
        this.service = service;
    }

    @PostMapping("/{conversationId}")
    public String chat(@PathVariable String conversationId, @RequestBody String question) {
        return service.ask(conversationId, question);
    }
}
```

### 9.7 Try it

```bash
curl -X POST http://localhost:8080/api/assistant/user-42-session \
     -H "Content-Type: text/plain" \
     -d "Nightly build failed. Create a Jira ticket and post the link in #dev-team"
```

Flow for this call = exactly the diagram in section 4.2: controller → service → `ChatClient` → advisor chain → LLM (search) → LLM (call) → MCP server → LLM (answer) → controller → user.

---

## 10. The 3 search strategies and which to pick

All implement `ToolIndex`.

| Strategy | Class | How it searches | Needs | Good for |
|---|---|---|---|---|
| **Regex** (default) | `RegexToolIndex` | Pattern match on tool **names** | Nothing | Tools with strict naming like `get_*_data` |
| **Keyword** | `LuceneToolIndex` | Apache Lucene keyword search; hits below a min score (default 0.25) are dropped | Lucene (bundled in starter) | Most MCP setups; fast, free, no embeddings |
| **Semantic** | `VectorToolIndex` | Embeds tool name + description; embeds the query; returns top-K similar | A `VectorStore` bean + embedding model | Natural-language queries where words differ ("make a ticket" vs `jira_create_issue`) |

**How to choose (simple rule):**

- Start with **Lucene** — it needs no extra infrastructure.
- Go **Vector** if the model's search words often do not match your tool words (synonyms, other languages, vague descriptions).
- **Regex** only if your tool names are very consistent. Because the default is regex, many people leave it by accident and then see "no tools found". Set `tool-index-type` explicitly.

You can also write your own `ToolIndex` (methods: `indexTool`, `indexTools`, `search`, `clearIndex`) — for example with role-based filtering so a user only finds tools they are allowed to use. Every method is scoped by `sessionId`.

---

## 11. Session ID — the most common mistake

The advisor indexes tools **per session**. So **every request must carry a session ID**, by default under `ChatMemory.CONVERSATION_ID`.

```java
chatClient.prompt()
    .user("...")
    .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, "user-42-session"))
    .call().content();
```

- If you already use `MessageChatMemoryAdvisor` with conversation IDs, the same key is already in the context — you get this "for free".
- If you use another key (`tenantId`, `userId`), tell the advisor: property `session-id-key-name: tenantId`, or `.sessionIdKeyName("tenantId")` in the builder.
- Different sessions have **isolated indexes** (important for multi-tenant apps).
- A bad idea: using one fixed ID for all users. Everyone would share one index and one discovered-tool state.

---

## 12. Memory of discovered tools and eviction

### 12.1 Accumulation (`reference-tool-name-accumulation`)

- `true` (default): once a tool is found, it stays in the list for later turns of that conversation. Good for multi-step tasks (create ticket → comment on ticket → close ticket).
- `false`: the tool list is rebuilt each turn from the **latest** search only (including all parallel searches in that turn). Smaller requests, but the model may need to search again for a tool it already used.

### 12.2 Eviction (freeing memory)

Per-session indexes use memory. Strategies:

| Strategy | Behavior |
|---|---|
| `LruEvictionStrategy(n)` (default, 1000) | Drops the least-recently-used session when more than `n` are active |
| `TtlEvictionStrategy(duration)` | Drops sessions idle longer than the duration |
| `CompositeEvictionStrategy(...)` | Evict if **any** inner strategy says so |
| `NeverEvictStrategy.INSTANCE` | Never evicts automatically; you call `advisor.evictSession(sessionId)` |
| `AlwaysEvictStrategy.INSTANCE` | Re-indexes before every request (testing, or tool set changes each request) |

Eviction is checked lazily on each request (no background thread). You can free a session on logout with `advisor.evictSession(sessionId)`.

*My note (inference, not from the docs):* the index is built at session start, so if an MCP server adds/removes tools while the app runs, a long-lived session may not see the change. Use a TTL, or evict the session, if your MCP tool list changes often.

---

## 13. How to optimize (checklist)

1. **Use it only when it pays off.** Docs say it is a good fit with ~10+ tools, tool definitions above ~10K tokens per request, multi-server MCP, or accuracy problems. The Dynamic Tool Discovery guide uses slightly different numbers (20+ tools, >5K tokens), so treat these as rough guides and **measure**. With under 10 tools, or tools that are used in every session, or very compact definitions, the extra search round trips can cost more than they save — stay with `ToolCallingAdvisor`.
2. **Write great tool descriptions.** Search quality = description quality. "Creates a Jira issue in a project with summary and description" beats "Jira tool". For Vector/Lucene this is the biggest lever.
3. **Filter the catalog first** with `McpToolFilter`. Remove admin, duplicate, experimental and irrelevant tools.
4. **Tune `max-results`.** 3–5 is a good start. Too high = you are sending many definitions again; too low = the right tool may not be returned.
5. **Choose the right index** (section 10). Set `tool-index-type` explicitly.
6. **Tune `lucene.min-score-threshold`.** Raise it for stricter matches; lower it if searches return nothing.
7. **Customize `system-message-suffix`** if the model forgets to search or guesses tool names. Keep it short and clear. The default template tells the model not to guess a tool name that was not returned by a search.
8. **Decide on accumulation** (section 12.1) based on whether your tasks are multi-step.
9. **Control memory** with TTL + LRU eviction.
10. **Split by use case.** If one flow uses only 4 tools every time, give it its own `ChatClient` with the normal `ToolCallingAdvisor`. Use the search advisor on the big "general assistant" client.
11. **Measure.** Compare `usage` (prompt tokens, completion tokens) with and without the advisor, and also **latency** (number of LLM calls goes up). Use the same set of real questions for both runs.
12. **Keep MCP timeouts sane** (`request-timeout`), because one user question can trigger several MCP calls.

---

## 14. Limitations you should know

- **All tools are deferred.** There is currently no flag to keep some "core" tools always visible. An open issue in the Spring AI repo (#7047, against 2.0.1) describes exactly this: the advisor replaces the request's tool callbacks with `toolSearchTool` plus the tools the last search named. Workarounds people use: subclass the advisor, or run core tools in a separate `ChatClient`. Check the issue for current status before relying on either.
- **Extra latency.** At least one additional LLM round trip for the search.
- **Search can miss.** If the right tool is not returned, the model will not use it. This is why descriptions and `max-results` matter.
- **Model must cooperate.** It has to call `toolSearchTool` and not invent tool names. Smaller local models may do this less reliably — test with your model.
- **Not worth it for small tool sets.**
- **Spring AI 2.0 feature.** The advisor lives in separate modules (`spring-ai-tool-search-advisor`, `spring-ai-tool-search-tool`). Check the upgrade notes if you come from older versions or earlier milestone artifacts (the package/module of the class has moved before).

---

## 15. Debugging and troubleshooting

### See what is really sent to the LLM

Add Spring AI's logger advisor and turn on debug logging:

```java
.defaultAdvisors(new SimpleLoggerAdvisor(), MessageChatMemoryAdvisor.builder(chatMemory).build())
```
```yaml
logging:
  level:
    org.springframework.ai.chat.client.advisor: DEBUG
```

Watch for: Request 1 should list **only** `toolSearchTool`; later requests should list `toolSearchTool` + discovered tools. If Request 1 lists all your tools, the search advisor is not active.

### Common problems

| Symptom | Likely cause | Fix |
|---|---|---|
| Error about a missing session/conversation ID | You did not pass `ChatMemory.CONVERSATION_ID` | Add `.advisors(a -> a.param(ChatMemory.CONVERSATION_ID, id))` or set `session-id-key-name` |
| All tools still sent to the LLM | `enabled` is not `true`, or the client was built by hand without the advisor | Set the property, or add the advisor in the builder |
| Search returns nothing | Regex index with descriptive queries, or Lucene threshold too high | Switch to `lucene`/`vector`; lower `min-score-threshold` |
| `vector` fails at startup | No `VectorStore` bean / embedding model | Add a vector store starter and an embedding model |
| Model answers without using any tool | Weak system prompt, or model ignores the search tool | Add `system-message-suffix`; test a stronger model |
| `IllegalStateException` about duplicate tool names | `noPrefix()` generator with servers sharing tool names | Use the default prefix generator |
| STDIO MCP server will not start on Windows | `npx` is a `.cmd` file | Use `cmd.exe` with args `/c`, `npx`, ... |
| Startup error mixing clients | SYNC and ASYNC mixed | Use only one `spring.ai.mcp.client.type` |
| Memory grows over time | Many sessions, never evicted | Configure `eviction.ttl` and `lru-max-sessions` |
| Tool list never updates | Old session index | Evict the session, or use TTL |

---

## 16. Quick summary / cheat sheet

**What it is:** a `ToolCallingAdvisor` subclass that hides your tools and gives the LLM one search tool; tools are revealed on demand.

**Why:** fewer tokens, better tool choice, big MCP catalogs become manageable.

**Flow in one line:**
`User → Controller → ChatClient → Advisor indexes tools → LLM(search) → local ToolIndex search → LLM(call discovered tool) → MCP client → MCP server → result → LLM(final text) → User`

**Minimum to use it:**
1. Dependency: `spring-ai-starter-tool-search-advisor`
2. `spring.ai.chat.client.tool-search-advisor.enabled=true`
3. `tool-index-type: lucene` (recommended start)
4. MCP client starter + `connections` config
5. Pass `ChatMemory.CONVERSATION_ID` on every request
6. Give `defaultToolCallbacks(mcpToolCallbackProvider)` to the `ChatClient`

**Best optimization levers:** good descriptions → `McpToolFilter` → `max-results` → right index → TTL/LRU eviction → measure tokens **and** latency.

---

## Sources (read these for the latest details)

- Spring AI reference — Tool Search Tool: https://docs.spring.io/spring-ai/reference/api/tools/tool-search-tool.html
- Spring AI guide — Dynamic Tool Discovery: https://docs.spring.io/spring-ai/reference/guides/dynamic-tool-search.html
- Spring AI reference — MCP Client Boot Starter: https://docs.spring.io/spring-ai/reference/api/mcp/mcp-client-boot-starter-docs.html
- Spring blog — Smart Tool Selection (34–64% token savings): https://spring.io/blog/2025/12/11/spring-ai-tool-search-tools-tzolov
- Spring blog — Composable Tool Calling in Spring AI 2.0: https://spring.io/blog/2026/06/15/spring-ai-composable-tool-calling/
- GitHub issue #7047 (keeping some tools always declared): https://github.com/spring-projects/spring-ai/issues/7047
- Original pattern by Anthropic: https://www.anthropic.com/engineering/advanced-tool-use
