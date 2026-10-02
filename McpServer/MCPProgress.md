# MCP Progress — Simple Step-by-Step Guide (with your HelpDesk example)

You will learn this in 4 small steps. Each step has **complete code**, **what it does in plain words**, and **what you will see in the console**.

| Step | What you build | Result |
|---|---|---|
| 1 | Understand what really happens (with real numbers) | No code, just the idea |
| 2 | Server sends progress | Your `getTicketStatus` tool reports 10%, 20% … 100% |
| 3 | Client receives progress | Your console prints the percentage |
| 4 | Many tools (LLM decides which) | Each tool gets its own progress, no mixing |
| 5 (optional) | Show progress in the browser | Live progress bar data |

---

## Step 1 — What really happens

### The idea in a story

You order food and the restaurant gives you a **ticket number: 57**.
While the kitchen cooks, a waiter walks to you and says:

- "Order **57** — 20% done"
- "Order **57** — 60% done"
- "Order **57** — ready" (the real result)

The ticket number matters because the waiter may carry many orders. **Ticket number = `progressToken`.**

The most important rule: **the waiter only talks if you gave him a ticket number.** In MCP: if the client does not send a `progressToken`, the server has nothing to report to and sends nothing.

### The real messages (what travels between client and server)

**1) Client calls the tool and includes a token:**

```json
{
  "method": "tools/call",
  "params": {
    "name": "getTicketStatus",
    "arguments": { "username": "john" },
    "_meta": { "progressToken": "req-1" }
  }
}
```

**2) Server sends progress messages (several times, while it works):**

```json
{ "method": "notifications/progress",
  "params": { "progressToken": "req-1", "progress": 10, "total": 100, "message": "Step 1 of 10" } }

{ "method": "notifications/progress",
  "params": { "progressToken": "req-1", "progress": 20, "total": 100, "message": "Step 2 of 10" } }
```

**3) Server sends the final result** (progress must stop after this):

```json
{ "result": { "content": [ ...tickets... ] } }
```

### What each field means

| Field | Meaning | Example |
|---|---|---|
| `progressToken` | The ticket number. Links progress to the request | `"req-1"` |
| `progress` | How much work is **done so far** | `40` |
| `total` | How much work there is **in total** (optional) | `100` |
| `message` | Text for humans (optional) | `"Step 4 of 10"` |

### The percentage calculation (this is the only math)

```
percentage = progress / total * 100
```

| Server sends | Calculation | Result |
|---|---|---|
| progress=40, total=100 | 40 / 100 × 100 | 40% |
| progress=3, total=10 (3 of 10 items) | 3 / 10 × 100 | 30% |
| progress=0.5, total=1.0 | 0.5 / 1.0 × 100 | 50% |
| progress=7, **no total** | cannot calculate | unknown — show "working…" |

Two rules:
1. `progress` must **always grow** (10, 20, 30 …). It can never go down.
2. **Always send `total`.** Without it, the client must guess if "40" means 40% or 40 rows.

---

## Step 2 — Server: send progress

This is your `HelpDeskTools`. The tool has to do 3 things:

1. Ask: "did the client send a token?" (`ctx.request().progressToken()`)
2. If yes, send `progress` and `total` while working.
3. Reach **exactly** `total` at the end, then return the result.

### Complete code

```java
package com.eazybytes.mcpserverremote.tool;

import io.modelcontextprotocol.server.McpSyncServerExchange;
import io.modelcontextprotocol.spec.McpSchema.ProgressNotification;
import org.springframework.ai.mcp.annotation.McpTool;
import org.springframework.ai.mcp.annotation.McpToolParam;
import org.springframework.ai.mcp.annotation.context.McpSyncRequestContext;
// ... your other imports (entity, service, lombok, slf4j)

@Component
@RequiredArgsConstructor
public class HelpDeskTools {

    private static final Logger LOGGER = LoggerFactory.getLogger(HelpDeskTools.class);

    private final HelpDeskTicketService service;

    @McpTool(name = "getTicketStatus", description = "Fetch the status of the tickets based on a given username")
    List<HelpDeskTicket> getTicketStatus(
            @McpToolParam(description = "Username to fetch the status of the help desk tickets") String username,
            McpSyncRequestContext ctx) throws InterruptedException {

        LOGGER.info("Fetching tickets for user: {}", username);

        // The work has 10 steps, so total = 10
        int totalSteps = 10;

        // Stage 1: real work
        List<HelpDeskTicket> tickets = service.getTicketsByUsername(username);

        // Stage 2: pretend heavy processing, 10 steps
        for (int step = 1; step <= totalSteps; step++) {
            Thread.sleep(1000);                                   // <- the real work of this step
            sendProgress(ctx, step, totalSteps, "Step " + step + " of " + totalSteps);
        }

        return tickets;                                           // after return: send NOTHING more
    }

    // Small helper: sends one progress message, only if the client asked for it
    private void sendProgress(McpSyncRequestContext ctx, double done, double total, String message) {
        Object token = ctx.request().progressToken();            // the "ticket number" from the client
        McpSyncServerExchange exchange = ctx.exchange();

        if (token == null || exchange == null) {
            return;                                               // client did not ask -> do nothing
        }
        exchange.progressNotification(
                ProgressNotification.builder(String.valueOf(token), done)   // token + progress
                        .total(total)                                       // total
                        .message(message)                                   // text
                        .build());
    }
}
```

### What this does, line by line (only the new ideas)

- `McpSyncRequestContext ctx` — a special parameter. Spring injects it. The AI (LLM) never sees it, it is **not** part of the tool's inputs.
- `ctx.request().progressToken()` — the token the client sent. It is `null` if the client did not send one.
- `sendProgress(ctx, step, totalSteps, ...)` — sends "step X of 10". Because `total = 10`, the client can calculate: step 4 → 4/10 = 40%.
- The loop goes `1..10` (not `0..9`), and we report **after** the work of the step is finished. So the last message is 10 of 10 = 100%.

### The mistake in your original loop

```java
for (int i = 0; i < 10; i++) {
    Thread.sleep(1000);
    int percent = (i * 100) / 10;      // i = 0..9  -> 0, 10, 20 ... 90   (never 100!)
    ctx.progress(spec -> spec.progress(percent)....);
}
```

- It reaches 90, never 100.
- It sends a percent number without `total`, so the receiver has to guess the scale.

(Your `ctx.progress(...)` approach also works. If you keep it, use `(i + 1) * 100 / 10` and make sure the client treats the number as a percent. The version above is safer because `total` explains the scale.)

> Also in `createTicket`: `LOGGER.info("... user: {} with details: {}", ticketRequest)` has two `{}` but one value. Give two values or remove one `{}`.

---

## Step 3 — Client: receive progress

The client needs 3 things:

1. **Send a token** with the request (otherwise the server stays silent).
2. **A listener** with `@McpProgress` to receive messages.
3. **Calculate the percentage** with the formula from Step 1.

### 3.1 Send the token

In the Spring AI MCP client, a value that you put into the **tool context** is forwarded to the server as request metadata. So the entry named `progressToken` becomes `_meta.progressToken`.

```java
@Service
public class AssistantService {

    private final ChatClient chatClient;

    public AssistantService(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    public String ask(String question) {
        return chatClient.prompt()
                .user(question)
                .toolContext(Map.of("progressToken", "req-1"))     // <- the ticket number
                .call()
                .content();
    }
}
```

### 3.2 The listener

```java
package com.eazybytes.mcpclient.util;

import io.modelcontextprotocol.spec.McpSchema;
import org.springframework.ai.mcp.annotation.McpProgress;
import org.springframework.stereotype.Component;
// slf4j imports

@Component
public class HelpDeskToolProgressListener {

    private static final Logger LOGGER = LoggerFactory.getLogger(HelpDeskToolProgressListener.class);

    @McpProgress(clients = "eazybytes")      // must equal the connection name in application.yml
    public void onProgress(McpSchema.ProgressNotification notification) {

        double progress = notification.progress();   // done so far
        Double total = notification.total();         // can be null

        if (total != null && total > 0) {
            double percent = progress / total * 100;
            LOGGER.info("Request {} -> {}% ({} of {}) - {}",
                    notification.progressToken(), Math.round(percent),
                    progress, total, notification.message());
        } else {
            LOGGER.info("Request {} -> working... (counter={}) - {}",
                    notification.progressToken(), progress, notification.message());
        }
    }
}
```

What it does: for every progress message from the `eazybytes` server, it calculates the percent and prints it. If the server sent no `total`, it prints "working…" because a percentage is impossible.

### 3.3 Config

```yaml
spring:
  ai:
    mcp:
      client:
        type: SYNC
        request-timeout: 60s            # default is 20s; a tool slower than this fails
        streamable-http:
          connections:
            eazybytes:                  # this name = clients = "eazybytes"
              url: http://localhost:8081
```

### 3.4 What you should see

Ask: *"Show the tickets for john"*. After the LLM decides to call `getTicketStatus`, the client console prints, one line per second:

```
Request req-1 -> 10% (1.0 of 10.0) - Step 1 of 10
Request req-1 -> 20% (2.0 of 10.0) - Step 2 of 10
...
Request req-1 -> 100% (10.0 of 10.0) - Step 10 of 10
```

**If you see nothing**, check in this order:
1. Is `progressToken` in `toolContext`? (no token = no progress)
2. Does `clients = "eazybytes"` match the yml connection name exactly?
3. Does the server use a normal (stateful) transport, not stateless?

---

## Step 4 — Many tools (the LLM chooses which, in any order)

### The problem

User says: *"Show john's tickets and then create a ticket for his printer issue."*

The LLM will call `getTicketStatus`, then `createTicket`. **You don't know in advance** which tools it will call or in which order.

With Step 3's code, we put one fixed token `req-1` in the tool context. So **both tools use the same token `req-1`**:

```
getTicketStatus:  req-1 -> 10 ... 100
createTicket:     req-1 -> 5 ...        <- progress goes DOWN from 100 to 5 — not allowed!
```

Also, the notification contains only the token, **not the tool name**, so you can't tell which tool the 20% belongs to.

### The solution in plain words

> Give **every tool call its own new ticket number**, and write it in a **notebook**: "ticket t-1 belongs to getTicketStatus", "ticket t-2 belongs to createTicket".
> When a progress message arrives, look in the notebook to know which tool it is.

How do we catch every tool call without knowing the tools in advance? Every MCP tool the LLM can call is a `ToolCallback` object. We **wrap** each one in a small class. The wrapper:

1. Creates a new token (UUID).
2. Writes it in the notebook (a `Map`).
3. Calls the real tool, passing that token.
4. Removes the token from the notebook when the tool finishes.

Because we wrap **all** tools, it doesn't matter which one the LLM picks.

```
LLM: call getTicketStatus  -> wrapper makes token t-1 -> notebook {t-1: getTicketStatus} -> server -> progress(t-1 ...) -> listener checks notebook -> "getTicketStatus 40%"
LLM: call createTicket     -> wrapper makes token t-2 -> notebook {t-2: createTicket}     -> server -> (no progress, quick) -> result
```

### 4.1 The notebook

```java
@Component
public class ProgressRegistry {

    public record CallInfo(String toolName, Instant startedAt) {}

    private final Map<String, CallInfo> active = new ConcurrentHashMap<>();

    public String newToken(String toolName) {
        String token = UUID.randomUUID().toString();
        active.put(token, new CallInfo(toolName, Instant.now()));
        return token;
    }

    public CallInfo find(String token) {
        return active.get(token);
    }

    public void remove(String token) {
        active.remove(token);
    }
}
```

### 4.2 The wrapper (one per tool)

```java
public class ProgressAwareToolCallback implements ToolCallback {

    private final ToolCallback realTool;
    private final ProgressRegistry registry;

    public ProgressAwareToolCallback(ToolCallback realTool, ProgressRegistry registry) {
        this.realTool = realTool;
        this.registry = registry;
    }

    @Override
    public ToolDefinition getToolDefinition() {
        return realTool.getToolDefinition();            // the LLM sees the same tool as before
    }

    @Override
    public ToolMetadata getToolMetadata() {
        return realTool.getToolMetadata();
    }

    @Override
    public String call(String toolInput) {
        return call(toolInput, null);
    }

    @Override
    public String call(String toolInput, ToolContext toolContext) {
        String toolName = realTool.getToolDefinition().name();
        String token = registry.newToken(toolName);                  // new ticket number for THIS call

        Map<String, Object> context = new HashMap<>();
        if (toolContext != null) {
            context.putAll(toolContext.getContext());
        }
        context.put("progressToken", token);                         // sent to the server as _meta.progressToken

        try {
            return realTool.call(toolInput, new ToolContext(context));
        } finally {
            registry.remove(token);                                  // call finished -> forget the token
        }
    }
}
```

### 4.3 Use the wrapper in the ChatClient

```java
@Configuration
public class ChatConfig {

    @Bean
    ChatClient chatClient(ChatClient.Builder builder,
                          SyncMcpToolCallbackProvider mcpTools,
                          ProgressRegistry registry) {

        ToolCallback[] wrappedTools = Arrays.stream(mcpTools.getToolCallbacks())
                .map(tool -> (ToolCallback) new ProgressAwareToolCallback(tool, registry))
                .toArray(ToolCallback[]::new);

        return builder
                .defaultToolCallbacks(wrappedTools)          // the LLM now uses the wrapped tools
                .build();
    }
}
```

And in `AssistantService` **remove** the fixed `toolContext(Map.of("progressToken", "req-1"))` — the wrapper creates the token now:

```java
public String ask(String question) {
    return chatClient.prompt()
            .user(question)
            .call()
            .content();
}
```

### 4.4 The listener now knows the tool name

```java
@Component
@RequiredArgsConstructor
public class HelpDeskToolProgressListener {

    private static final Logger LOGGER = LoggerFactory.getLogger(HelpDeskToolProgressListener.class);

    private final ProgressRegistry registry;

    @McpProgress(clients = "eazybytes")
    public void onProgress(McpSchema.ProgressNotification notification) {

        String token = String.valueOf(notification.progressToken());
        ProgressRegistry.CallInfo call = registry.find(token);

        if (call == null) {
            return;                              // this call already finished, ignore late message
        }

        double progress = notification.progress();
        Double total = notification.total();

        if (total != null && total > 0) {
            LOGGER.info("[{}] {}% - {}", call.toolName(),
                    Math.round(progress / total * 100), notification.message());
        } else {
            LOGGER.info("[{}] working... - {}", call.toolName(), notification.message());
        }
    }
}
```

### 4.5 What you should see now

```
[getTicketStatus] 10% - Step 1 of 10
[getTicketStatus] 20% - Step 2 of 10
...
[getTicketStatus] 100% - Step 10 of 10
(createTicket is fast and sends no progress, so no lines - that is normal)
```

If the LLM calls the tools in the other order, or calls the same tool twice, each call has its own token, so nothing mixes up. Tools that never send progress simply don't print anything.

### Good to know

- This also works if the tools run at the same time: each call has its own token, and the `ConcurrentHashMap` is thread-safe.
- There is no honest "overall percentage" for the whole chat request, because nobody knows how many tools the LLM will still call. Show progress **per tool**.
- If you use the Tool Search advisor from guide 13, nothing changes: it works with whatever tools you register, and the wrapped tools keep the same definitions.

---

## Step 5 (optional) — Show the progress to the user in the browser

Right now progress goes only to the console. To send it to a web page, use Server-Sent Events (SSE).

### The broadcaster

```java
@Component
public class ProgressBroadcaster {

    private final List<SseEmitter> emitters = new CopyOnWriteArrayList<>();

    public SseEmitter subscribe() {
        SseEmitter emitter = new SseEmitter(0L);
        emitters.add(emitter);
        emitter.onCompletion(() -> emitters.remove(emitter));
        emitter.onTimeout(() -> emitters.remove(emitter));
        emitter.onError(e -> emitters.remove(emitter));
        return emitter;
    }

    public void publish(Map<String, Object> event) {
        for (SseEmitter emitter : emitters) {
            try {
                emitter.send(SseEmitter.event().name("progress").data(event));
            } catch (IOException e) {
                emitter.complete();
            }
        }
    }
}
```

### Call it from the listener (add two lines in `onProgress`)

```java
Double percent = (total != null && total > 0) ? progress / total * 100 : null;
broadcaster.publish(Map.of(
        "tool", call.toolName(),
        "percent", percent == null ? -1 : Math.round(percent),   // -1 = unknown
        "message", String.valueOf(notification.message())));
```

(Inject `ProgressBroadcaster broadcaster` in the listener like `registry`.)

### The endpoint and the browser code

```java
@GetMapping(value = "/api/progress", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public SseEmitter progress() {
    return broadcaster.subscribe();
}
```

```js
const es = new EventSource("/api/progress");
es.addEventListener("progress", (e) => {
  const p = JSON.parse(e.data);
  console.log(p.tool, p.percent === -1 ? "working..." : p.percent + "%", p.message);
});
```

Open this SSE connection **before** sending the chat request.

*(This simple version sends events to everyone connected. For many users, add a `conversationId` to the tool context and the notebook, and send each event only to that user's emitter.)*

---

## Final checklist

**Server (`@McpTool` method)**
- [ ] Add the `McpSyncRequestContext ctx` parameter
- [ ] Read the token with `ctx.request().progressToken()`; if `null`, send nothing
- [ ] Send `progress` + `total` + `message`; `progress` only grows; the last one equals `total`

**Client**
- [ ] Always send a token (Step 3: fixed token for one tool; Step 4: wrapper for many tools)
- [ ] `@McpProgress(clients = "...")` listener; the name must match the yml connection name
- [ ] Percentage = `progress / total * 100`; if `total` is null, show "working…"
- [ ] Raise `request-timeout` if a tool runs longer than 20 seconds

## Sources

- MCP specification — Progress: https://modelcontextprotocol.io/specification/latest/basic/utilities/progress
- Spring AI — MCP Annotations Special Parameters: https://docs.spring.io/spring-ai/reference/api/mcp/mcp-annotations-special-params.html
- Spring AI — MCP Client Boot Starter (tool context becomes MCP request metadata): https://docs.spring.io/spring-ai/reference/api/mcp/mcp-client-boot-starter-docs.html
