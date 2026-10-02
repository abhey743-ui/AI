# Call-Level MCP Tool Filtering — Choosing Tools Per Request, Not Globally

## 1. The difference from the last guide, in one sentence

`McpToolFilter` (the previous guide) decides, **once, at application startup**, which tools exist *at all* for the whole application — every request sees the exact same filtered set. What you've built here is different: a way to decide, **fresh, on every individual request**, exactly which tools *this specific call* gets to use — meaning two different endpoints in the same application can hand the model two completely different, independently chosen sets of tools, drawn live from whatever your connected MCP servers currently expose.

```
McpToolFilter (global, discovery-time)        Call-level filtering (this guide)
───────────────────────────────────           ──────────────────────────────────
Runs ONCE, at startup                         Runs EVERY TIME a request comes in
Same filtered tool set for EVERY request      Each endpoint/call can pick its own subset
Built into ToolCallbackProvider automatically Built manually, by YOUR code, per call
Applied via .defaultTools(toolCallbackProvider) Applied via .tools(toolArray) per request
```

---

## 2. The fully explained code

### 2.1 `ToolUtil` — manually building a filtered `ToolCallback[]`

```java
package com.McpClient.McpClient.McpClientController;

import io.modelcontextprotocol.client.McpSyncClient;
import org.springframework.ai.mcp.SyncMcpToolCallback;
import org.springframework.ai.tool.ToolCallback;

import java.util.List;

public class ToolUtil {

    /**
     * Builds a filtered array of ToolCallbacks, drawn live from the given MCP
     * clients, restricted to a specific server and/or a specific tool name hint.
     *
     * Pass null (or a blank string) for either argument to mean "no restriction
     * on this dimension" — e.g. serverName="Book", toolName=null means
     * "every tool from any server whose name contains 'Book', regardless of
     * the tool's own name."
     */
    public static ToolCallback[] selectToolsFor(List<McpSyncClient> mcpClients,
                                                  String serverName, String toolName) {
        return mcpClients.stream()
                // For EVERY connected MCP client, ask it RIGHT NOW what tools
                // its server currently exposes — this is a live tools/list
                // call happening at the moment this method runs, not a cached
                // result from application startup.
                .flatMap(client -> client.listTools().tools().stream()
                        // Keep only tools whose SERVER name matches the
                        // serverName hint, AND whose own TOOL name matches
                        // the toolName hint. getServerInfo().name() is the
                        // exact same value McpConnectionInfo exposes to a
                        // global McpToolFilter — this is deliberately the
                        // same identity information, just read manually here
                        // instead of through that interface.
                        .filter(tool -> matches(client.getServerInfo().name(), serverName)
                                && matches(tool.name(), toolName))
                        // Wrap each surviving Tool definition into an actual,
                        // invokable Spring AI ToolCallback, bound to the
                        // specific McpSyncClient it came from — this is the
                        // manual equivalent of what ToolCallbackProvider does
                        // automatically for the WHOLE discovered set.
                        .map(tool -> (ToolCallback) SyncMcpToolCallback.builder()
                                .mcpClient(client)
                                .tool(tool)
                                .build()))
                .toArray(ToolCallback[]::new);
    }

    // A simple, forgiving substring match: a null or blank hint means "match
    // anything" (no restriction on this dimension at all); otherwise it's a
    // case-insensitive "does the actual value contain the hint" check.
    private static boolean matches(String actual, String hint) {
        return hint == null || hint.isBlank()
                || actual.toLowerCase().contains(hint.toLowerCase());
    }
}
```

### 2.2 `ClientController` — using it per endpoint

```java
package com.McpClient.McpClient.McpClientController;

import io.modelcontextprotocol.client.McpSyncClient;
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.tool.ToolCallback;
import org.springframework.ai.tool.ToolCallbackProvider;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;

@RestController
public class ClientController {

    private final ChatClient chatClient;
    // Injected directly: EVERY McpSyncClient instance your application
    // currently has connected — one per configured MCP server connection,
    // exactly as described in the stdio/Streamable HTTP guide. This gives
    // you low-level, direct access to each connection, bypassing the
    // higher-level ToolCallbackProvider abstraction entirely.
    private final List<McpSyncClient> mcpSyncClientList;

    public ClientController(List<McpSyncClient> mcpSyncClientList,
                             ChatClient.Builder builder,
                             ToolCallbackProvider toolCallbackProvider) {

        this.chatClient = builder
                // .defaultTools(toolCallbackProvider) is commented OUT here —
                // meaning this ChatClient registers NO default tools at all.
                // This is deliberate for this controller's purpose: every
                // tool this chatClient ever uses will be supplied explicitly,
                // per call, rather than being available globally by default.
                .build();

        this.mcpSyncClientList = mcpSyncClientList;
    }

    // An ordinary chat endpoint with NO tools available at all, since none
    // were registered as defaults above. This exists mainly as a contrast —
    // notice it has no access to any MCP tool whatsoever.
    @GetMapping("/chat")
    public String chat(@RequestParam("q") String q) {
        return chatClient.prompt(q).call().content();
    }

    // The actual call-level filtering endpoint.
    @GetMapping("/filteredChat")
    public String filteredChat(String q) {

        // Built FRESH for this one request: only tools from a server whose
        // name contains "Book" (e.g. your appointment-booking MCP server),
        // with no restriction on the specific tool name (null = any tool).
        ToolCallback[] tools = ToolUtil.selectToolsFor(mcpSyncClientList, "Book", null);

        // .tools(tools) registers this array for THIS SINGLE CALL ONLY —
        // it does not persist, does not affect any other endpoint, and does
        // not add to whatever (if anything) was registered as a default.
        return chatClient.prompt(q).tools(tools).call().content();
    }
}
```

---

## 3. `McpSyncClient` — what you're actually working with directly

`McpSyncClient` is the **official MCP Java SDK's own client type** — the same low-level object Spring AI's auto-configuration creates one instance of for every configured server connection (stdio or Streamable HTTP, per the earlier transport guide), and the same thing `ToolCallbackProvider` uses internally to build its own tool list. By injecting `List<McpSyncClient>` directly into your controller, **you're bypassing Spring AI's higher-level tool-callback abstraction entirely** and working with the raw, underlying client connections yourself.

- **`client.getServerInfo().name()`** — the identity the *server* reported during the initial MCP handshake; this is exactly the same value a global `McpToolFilter` would read via `connectionInfo.initializeResult().serverInfo().name()` from the previous guide — same underlying data, just accessed through a different, lower-level path here.
- **`client.listTools().tools()`** — sends a live `tools/list` request to that specific server, right now, and returns its current tool list. This is a genuinely important detail covered in §5.

---

## 4. `.tools(tools)` vs `.defaultTools(...)` — the per-call override

This is the exact mechanism that makes "different endpoints, different tool sets" possible in one application:

```java
// Bean-level default — applies to EVERY call through this ChatClient,
// unless a specific call overrides it:
builder.defaultTools(toolCallbackProvider)

// Per-call — applies ONLY to this one request, on top of (or instead of,
// depending on the exact API semantics) whatever defaults exist:
chatClient.prompt(q).tools(tools).call()
```

This mirrors exactly the same "bean-level default vs. service-level override" pattern already covered for system messages, advisors, and chat options earlier in this series — tools are no exception to that general rule. Here, because `.defaultTools(toolCallbackProvider)` is commented out entirely, `/chat` has access to nothing, while `/filteredChat` explicitly supplies its own hand-picked array for just that one call.

---

## 5. A critical operational detail — `listTools()` is a LIVE call, every time

This is the single most important thing to understand about this approach versus the global `McpToolFilter`:

> **`client.listTools()` inside `selectToolsFor(...)` sends an actual `tools/list` request to the real MCP server, at the moment `filteredChat` is invoked — on every single request, for every single call to this endpoint.**

Contrast this with `ToolCallbackProvider`, which (per the previous guide) discovers and filters tools **once**, typically at application startup, and then reuses that already-built, cached array for every subsequent request. The call-level approach trades that caching away in exchange for always reflecting the server's **current, live** state — if a server's available tools change between requests, `/filteredChat` sees the update on its very next call, with zero restart required; `/chat`'s pre-built `ToolCallbackProvider`, by contrast, would still be working off whatever it discovered at startup.

**The real cost:** every call to `/filteredChat` now pays the latency of an extra round trip to the MCP server (for `listTools()`) *before* the actual chat request to the model even begins. For a server reachable instantly on `localhost`, this is negligible; for a genuinely remote server over a real network, this adds measurable, repeated overhead to every single request that didn't exist in the startup-cached approach.

---

## 6. The full request flow for `/filteredChat`

```
GET /filteredChat?q=Book me an appointment with Dr. Iyer

1. ClientController.filteredChat("Book me an appointment with Dr. Iyer") runs.

2. ToolUtil.selectToolsFor(mcpSyncClientList, "Book", null) executes:
   a. For EVERY connected McpSyncClient, send a live tools/list request.
   b. Keep only tools from a server whose name contains "Book"
      (e.g. "booking-mcp-server"), with any tool name accepted.
   c. Wrap each surviving tool into a SyncMcpToolCallback bound to the
      specific client it came from.
   d. Return the resulting ToolCallback[] — potentially from ONE server only,
      even if several are connected overall.

3. chatClient.prompt(q).tools(tools).call().content() sends the question to
   the model, offering ONLY this hand-picked, request-specific tool array —
   not the globally registered defaults (there are none here), and not every
   tool from every connected server.

4. The model can only ever call a tool that was included in THIS specific
   array — anything from a server whose name doesn't contain "Book" is as
   invisible to the model on this call as if it had been excluded by a
   global McpToolFilter, just decided dynamically, per request, instead of
   once at startup.

5. The model's final answer, grounded only in whatever this narrow tool set
   allowed it to do, is returned as the HTTP response.
```

---

## 7. Two things worth fixing in this code

### 7.1 `.defaultAdvisors()` called with no arguments
Exactly the same issue flagged in the previous `ClientController` guide — this is a valid but entirely no-op call, registering zero advisors. If intentional as a placeholder, fine; otherwise, safe to delete, or fill in with whatever advisors (memory, logging, guardrails) this application actually needs.

### 7.2 `filteredChat(String q)` is missing `@RequestParam`
Compare the two endpoints:

```java
public String chat(@RequestParam("q") String q) { ... }        // explicit
public String filteredChat(String q) { ... }                    // implicit
```

Spring MVC *can* bind a plain method parameter to a request parameter of the same name without an explicit `@RequestParam`, but **only if the application was compiled with the `-parameters` javac flag** (so parameter names are preserved in the bytecode) — if that flag isn't set in your build, this will fail at runtime with a missing-parameter-name error the moment `/filteredChat` is actually called. Adding `@RequestParam("q")` explicitly, matching the other endpoint, removes this fragile dependency on a specific compiler flag:

```java
@GetMapping("/filteredChat")
public String filteredChat(@RequestParam("q") String q) {
    ...
}
```

---

## 8. `McpToolFilter` vs. call-level filtering — side by side

| | `McpToolFilter` (previous guide) | Call-level filtering (this guide) |
|---|---|---|
| **When it runs** | Once, at tool discovery/startup | Every single request |
| **Scope** | Global — same filtered set for every call through the app | Per-call — each endpoint can choose its own subset |
| **Reflects live server state?** | No — reflects whatever was discovered at startup | Yes — `listTools()` is called fresh every time |
| **Performance cost per request** | None — the filtered list is already built and cached | An extra `tools/list` round trip per call, per connected client checked |
| **Where tools come from** | Spring AI's auto-built `ToolCallbackProvider` | Manually built via your own `ToolUtil`, working directly with `McpSyncClient` |
| **Registration mechanism** | `.defaultTools(toolCallbackProvider)` | `.tools(toolArray)` per request |
| **Best for** | Application-wide security/scoping rules that should never vary by endpoint | Different endpoints/features genuinely needing different, independently chosen tool subsets, or needing always-current server state |

---

## 9. When to actually reach for this approach

- **Different endpoints, genuinely different jobs.** A `/book` endpoint that should only ever touch appointment-booking tools, and a completely separate `/billing` endpoint that should only ever touch billing tools — call-level filtering lets one `ChatClient` serve both without either one ever seeing the other's tools.
- **Freshness matters more than performance.** If a connected server's tool set can change at runtime and your application needs to reflect that immediately rather than after a restart, the live `listTools()` call here guarantees current state in a way a startup-time `ToolCallbackProvider` cannot.
- **You want both at once.** Nothing stops combining a global `McpToolFilter` (as a baseline safety net — denylisted keywords, untrusted-server restrictions) *with* call-level filtering on top (narrowing further, per endpoint) — the global filter still runs during `ToolCallbackProvider` construction for anything using `.defaultTools(toolCallbackProvider)`, while endpoints using the manual `ToolUtil` approach bypass that provider entirely and talk to the raw clients directly, so the two mechanisms don't automatically compose — worth being deliberate about which protection layer a given endpoint is actually relying on.

## 10. Recap

Where `McpToolFilter` decides once, globally, which tools exist for the whole application, this pattern decides fresh, per request, exactly which tools *this specific call* gets — by injecting the raw `List<McpSyncClient>` connections directly, calling `listTools()` live, filtering by server/tool name with a simple substring match, manually wrapping survivors into `SyncMcpToolCallback`s, and handing that array to `.tools(...)` for one call only via `ChatClient`'s per-request tool override. The trade-off is explicit: always-current server state and per-endpoint flexibility, at the cost of a live round trip to the MCP server on every single request that uses it, instead of the one-time cost a cached, startup-built `ToolCallbackProvider` pays.
