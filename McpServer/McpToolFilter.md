# `McpToolFilter` in Spring AI — What It Is, How It Processes Requests, and How to Use It Properly

## 1. What `McpToolFilter` actually is

Recall from the previous guide: the moment you have an MCP client connected to one or more servers, Spring AI's `ToolCallbackProvider` auto-configuration **automatically exposes every single tool those servers advertise** to your model, with zero filtering by default. That's fine for a small demo with one trusted, hand-built server — but it quietly becomes a real problem the moment you connect to:
- a **third-party** MCP server you don't fully control (it could add new tools tomorrow without you knowing),
- a server that exposes **far more tools than are actually relevant** to your application's purpose,
- or **multiple** MCP servers whose tools might **collide in name** with each other or with your own local `@Tool` methods.

`McpToolFilter` is Spring AI's answer to exactly this: a single interface that lets you **decide, tool by tool, which discovered MCP tools are actually allowed to reach the model at all** — before the `ToolCallbackProvider` ever wraps them into something the model can see or call.

```java
package org.springframework.ai.mcp;

@FunctionalInterface
public interface McpToolFilter {
    boolean test(McpConnectionInfo connectionInfo, McpSchema.Tool tool);
}
```

It's a simple predicate: given information about *which connection* a tool came from, and the tool's own definition, return `true` to let it through, or `false` to drop it entirely.

---

## 2. Why this exists — the real problems it's built to solve

### 2.1 Limiting the "blast radius" of an untrusted or chatty server
An MCP server you don't control — a third-party one, or one maintained by a different team — can change what it exposes at any time. Without filtering, the very next time your application starts up and reconnects, the model could suddenly gain access to a brand-new tool you never reviewed, with no code change on your end at all. `McpToolFilter` lets you draw a hard boundary: *only these specific tools from this server are ever allowed through*, regardless of what else that server might later advertise.

### 2.2 Resolving name collisions
Recall from the tool-calling guide: a local `@Tool`-annotated method and a remote MCP tool can end up with the exact same name. Spring AI's `DefaultMcpToolNamePrefixGenerator` automatically disambiguates **duplicate names across different MCP servers**, but it has no awareness of your own local, hand-written `@Tool` methods — if one of those collides with a remote MCP tool's name, you have to resolve it yourself, and dropping the remote one via a filter is the simplest fix.

### 2.3 Enforcing an allow-list instead of trusting a server's full surface
Some servers expose a wide surface of tools, only a handful of which are actually relevant to what your specific application needs. A filter lets you implement a strict allow-list — "only tools whose name starts with `allowed_`" — rather than trusting the server's entire advertised list by default.

### 2.4 Excluding tools by content, not just by name
The filter receives the full `McpSchema.Tool` object, including its description — meaning you can filter on *content*, not just identity. A common real pattern: exclude any tool whose description contains the word "experimental," keeping unstable server capabilities out of production entirely without needing to know their exact names in advance.

### 2.5 Applying different rules per connection
`McpConnectionInfo` tells you exactly *which* client/server connection a given tool came from — meaning one filter can apply **different rules to different servers** in the same application: perhaps a fully trusted internal server gets everything through unfiltered, while an external, less-trusted one is restricted to a narrow, explicitly approved subset.

---

## 3. What information you actually have inside the filter

```java
public boolean test(McpConnectionInfo connectionInfo, McpSchema.Tool tool) {
    // connectionInfo gives you:
    //   - connectionInfo.clientInfo()         -> name/version of the MCP client connection itself
    //   - connectionInfo.clientCapabilities() -> what capabilities this client connection declared
    //   - connectionInfo.initializeResult()   -> the full handshake result from the server,
    //                                            including the SERVER's own reported identity

    // tool gives you:
    //   - tool.name()         -> the tool's exact name, as the model will see it
    //   - tool.description()  -> the natural-language description the model reads
    //   - tool.inputSchema()  -> the JSON Schema defining its expected arguments

    return true; // or false
}
```

This is everything you need to build arbitrarily sophisticated rules: match on which server a tool came from, match on its name, match on keywords in its description, or any combination of the three.

---

## 4. A professional, production-shaped implementation

Rather than one narrow rule, a real filter typically composes several layered checks — a denylist, an allowlist, and per-server scoping — with clear logging so every exclusion is auditable:

```java
package com.EazyBytesSpringAi.EazyBytesSpringAi.Mcp;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.ai.mcp.McpConnectionInfo;
import org.springframework.ai.mcp.McpToolFilter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import io.modelcontextprotocol.spec.McpSchema;

import java.util.Set;

@Configuration
public class McpToolFilterConfig {

    private static final Logger logger = LoggerFactory.getLogger(McpToolFilterConfig.class);

    // Explicitly trusted server names — tools from these pass through unless they
    // also hit a keyword-based denylist rule below.
    private static final Set<String> TRUSTED_SERVERS = Set.of("library-mcp-server");

    // Hard denylist: keywords in a tool's description that should NEVER be exposed
    // to the model, regardless of which server advertised them.
    private static final Set<String> DENIED_KEYWORDS = Set.of("experimental", "deprecated", "internal-only");

    @Bean
    public McpToolFilter mcpToolFilter() {
        return (McpConnectionInfo connectionInfo, McpSchema.Tool tool) -> {

            String serverName = connectionInfo.initializeResult().serverInfo().name();
            String toolName = tool.name();
            String description = tool.description() == null ? "" : tool.description().toLowerCase();

            // Rule 1 — hard denylist by description keyword, applies to EVERY server,
            // no exceptions. This is checked first because it's a safety boundary,
            // not a convenience preference.
            boolean isDenied = DENIED_KEYWORDS.stream().anyMatch(description::contains);
            if (isDenied) {
                logger.warn("Excluding MCP tool '{}' from server '{}' — matched a denylisted keyword.",
                        toolName, serverName);
                return false;
            }

            // Rule 2 — untrusted servers only get through on an explicit allow-list
            // of tool name prefixes; trusted servers skip this check entirely.
            if (!TRUSTED_SERVERS.contains(serverName)) {
                boolean isExplicitlyAllowed = toolName.startsWith("public_");
                if (!isExplicitlyAllowed) {
                    logger.warn("Excluding MCP tool '{}' from UNTRUSTED server '{}' — not on the allow-list.",
                            toolName, serverName);
                    return false;
                }
            }

            // Rule 3 — resolve a known naming collision with a local @Tool method
            // called "book_appointment" by dropping the remote MCP version of it.
            if (toolName.equals("book_appointment") && !serverName.equals("appointment-mcp-server")) {
                logger.warn("Excluding MCP tool '{}' from '{}' — conflicts with a local @Tool method.",
                        toolName, serverName);
                return false;
            }

            // Everything else passes through.
            logger.info("Allowing MCP tool '{}' from server '{}'.", toolName, serverName);
            return true;
        };
    }
}
```

**Why structure it this way:**
- **Denylist checked first, unconditionally** — safety exclusions should never be bypassable by a server simply being on the trusted list; keeping it as the first check makes that guarantee obvious just from reading the code top to bottom.
- **Per-server trust level** — not every connected server deserves the same default posture; an internal, team-owned server can reasonably default to "allow everything not denylisted," while an external one defaults to "deny everything not explicitly allowed."
- **Logging every decision** — when a tool silently doesn't show up for the model to use, you want an audit trail explaining *why*, rather than having to guess or re-derive the filter's logic from memory later.

### 4.1 The one constraint to respect

> **Only one `McpToolFilter` bean is allowed in the application context.** If you need several independent sets of rules, combine them inside a single bean (as shown above — three "rules" living inside one filter's `test` method), rather than trying to register several `McpToolFilter` beans and hoping Spring merges them. It won't — it expects exactly one.

---

## 5. How the request is actually processed — the full flow

This is the part worth understanding precisely, because the filter runs at a very specific, easy-to-misjudge point in the pipeline — **at tool *discovery* time, not at tool *invocation* time.**

```
1. Spring Boot application starts.
   spring.ai.mcp.client.enabled=true triggers MCP client auto-configuration.

2. For each configured MCP server connection (stdio or Streamable HTTP),
   a McpSyncClient (or McpAsyncClient) is created and connects to that server.

3. Each client sends tools/list to its connected server, receiving back the
   server's full, unfiltered list of McpSchema.Tool definitions — exactly what
   the server itself advertises, with nothing removed yet.

4. The McpToolCallbackAutoConfiguration machinery builds a
   SyncMcpToolCallbackProvider (or AsyncMcpToolCallbackProvider), passing in:
     - every connected McpSyncClient/McpAsyncClient
     - YOUR McpToolFilter bean, via .toolFilter(...) on the provider's builder
       (if you didn't define one, a permissive default — "always true" — is
       used instead, meaning everything passes through unfiltered)

5. For EVERY tool returned by EVERY connected server's tools/list response,
   the provider calls:
       yourFilter.test(connectionInfoForThatServer, thatSpecificTool)

   - returns true  → the tool is wrapped into a Spring AI ToolCallback and
                      included in the provider's final array
   - returns false → the tool is silently dropped — it is NEVER wrapped,
                      NEVER included, and critically, NEVER sent to the model
                      as part of its available tools at all

6. The resulting ToolCallbackProvider bean — already containing ONLY the
   tools that survived filtering — is what your ClientController injects
   and registers via .defaultTools(toolCallbackProvider).

7. From this point on, every single chatClient.prompt(...).call() request
   sends the model a tool list that was ALREADY filtered at step 5 — the
   model literally has no knowledge that an excluded tool ever existed. It
   cannot ask about it, cannot attempt to call it by guessing its name,
   because it was never in the list of tools offered to it in the first
   place.
```

### 5.1 Why this distinction (discovery-time vs. invocation-time) matters

This is fundamentally different from something like a `SafeGuardAdvisor` from the advisors guide, which intercepts and can block a call **after** the model has already decided to attempt it. `McpToolFilter` works one layer earlier — it controls **what the model is even allowed to know exists.** This is a meaningfully stronger form of restriction: there's no scenario where the model "tries and gets refused" for a filtered-out tool, because from the model's perspective, that tool simply isn't part of reality for this conversation at all.

```
Advisor-based blocking (e.g. SafeGuardAdvisor):
   Model sees the tool → model decides to call it → advisor intercepts and
   blocks the call → model is informed the action failed

McpToolFilter:
   Model never sees the tool in the first place → there is nothing to decide
   to call → no failure to handle, because no attempt was ever possible
```

### 5.2 When filtering actually re-runs

The filtering happens whenever the `ToolCallbackProvider`'s tool list is built/refreshed from the underlying MCP client connections — in the common case, this is effectively once, at application startup, when the clients first connect and discover their servers' tools. If a server's tool list changes later and your setup supports picking that up dynamically, the filter runs again against the newly discovered list at that point — but it is never invoked per individual chat request; it's a discovery-time gate, not a per-call gate.

---

## 6. A focused example — resolving the exact name-collision scenario

Tying this back directly to §2.2, here's the minimal version of a filter whose *only* job is preventing a specific local/remote name collision, if that's genuinely all you need right now:

```java
@Bean
public McpToolFilter mcpToolFilter() {
    // Drop any MCP-discovered tool literally named "book_appointment" — a local
    // @Tool method with that exact name already exists in this application,
    // and we want OUR local implementation to be the only one the model sees.
    return (connectionInfo, tool) -> !tool.name().equals("book_appointment");
}
```

Simple, single-purpose filters like this are perfectly fine — not every real use needs the full, multi-rule version from §4. Build the complexity your actual situation calls for, nothing more.

---

## 7. Recap

`McpToolFilter` is a single-method predicate — `test(McpConnectionInfo, McpSchema.Tool) -> boolean` — that Spring AI's MCP client auto-configuration applies to **every tool discovered from every connected server**, at the moment those tools are wrapped into a `ToolCallbackProvider`. A tool that fails the filter is dropped entirely before it's ever presented to the model — not blocked after an attempted call, but genuinely never visible to the model's reasoning in the first place. This makes it the right tool for limiting the blast radius of untrusted or chatty MCP servers, resolving name collisions between local and remote tools, and enforcing an explicit allow-list instead of trusting a server's full advertised surface — all implemented as exactly one bean in your application context, since Spring AI only permits a single `McpToolFilter` to be registered at a time.
