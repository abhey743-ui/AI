# Wiring MCP Tools Into `ChatClient` — `ToolCallbackProvider`, Fully Explained

## 1. Why this one small class matters so much

Across the last two guides, we built two separate halves of a system: MCP **servers** exposing tools (a remote Streamable HTTP server, and a locally-spawned `stdio` filesystem server), and an MCP **client** configured in `application.yml` to connect to both. But none of that, by itself, actually lets the **language model** use any of those tools. Configuration alone doesn't wire discovered MCP tools into a `ChatClient`'s tool-calling machinery — something in your actual Java code has to bridge the two.

That bridge is `ToolCallbackProvider`, and the controller you wrote is the smallest possible piece of code that performs that bridging. This guide exists to make sure every line of it — and everything invisible that happens around it — is completely clear.

---

## 2. The fully documented code

```java
package com.EazyBytesSpringAi.EazyBytesSpringAi.Mcp;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.tool.ToolCallbackProvider;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class ClientController {

    // The fully built ChatClient this controller will use for every request.
    // It's built ONCE, in the constructor, and reused for every incoming call —
    // not rebuilt per-request, which would be wasteful and pointless.
    private final ChatClient chatClient;

    /**
     * Spring injects two things here automatically:
     *
     * 1. ChatClient.Builder — the standard auto-configured builder, already wired
     *    to whichever ChatModel your model starter (OpenAI, Anthropic, etc.) provides.
     *
     * 2. ToolCallbackProvider — NOT something you wrote or configured yourself.
     *    This bean is auto-configured by Spring AI's MCP CLIENT starter, the moment
     *    spring.ai.mcp.client.enabled=true is set and at least one MCP server
     *    connection exists in application.yml (stdio, streamable-http, or both).
     *    It represents EVERY tool discovered across EVERY connected MCP server,
     *    already wrapped into Spring AI's internal tool-callback format.
     */
    public ClientController(ChatClient.Builder builder, ToolCallbackProvider toolCallbackProvider) {

        this.chatClient = builder
                // .defaultTools(...) registers tools that are available on EVERY
                // call made through this ChatClient, without needing to repeat
                // tool registration in every single service method.
                //
                // Passing a ToolCallbackProvider here (rather than a plain Java
                // object with @Tool-annotated methods) tells Spring AI: "take
                // whatever tools this provider currently knows about, and make
                // ALL of them available to the model." Since the provider was
                // built from your MCP client connections, this single line is
                // what actually exposes every MCP-discovered tool — from your
                // remote library-mcp-server over Streamable HTTP, AND from your
                // locally-spawned filesystem server over stdio, if both are
                // connected — to this ChatClient.
                .defaultTools(toolCallbackProvider)
                .build();
    }

    /**
     * A single, simple chat endpoint. Nothing about this method itself is
     * MCP-specific — it looks exactly like any other ChatClient call from
     * earlier in this series. That's precisely the point: once tools are
     * registered at the bean-building stage above, EVERY call through this
     * chatClient automatically has access to them. The service method doesn't
     * need to know or care that some of its "tools" are backed by a remote
     * server or a spawned subprocess rather than a local Java method.
     */
    @GetMapping("/chat")
    public String chat(@RequestParam("q") String q) {
        return chatClient.prompt(q)
                .call()
                .content();
    }
}
```

*(One small fix from your original: `.defaultAdvisors()` was being called with no arguments at all — a valid but completely no-op call that registers zero advisors. It's been removed here since it did nothing; if you do want memory, logging, or caching advisors from the earlier guides, add them explicitly inside that call, e.g. `.defaultAdvisors(MessageChatMemoryAdvisor.builder(chatMemory).build())`.)*

---

## 3. What `ToolCallbackProvider` actually is, in depth

### 3.1 The interface

```java
public interface ToolCallbackProvider {
    ToolCallback[] getToolCallbacks();
}
```

It's deliberately simple — a provider that, when asked, hands back an array of `ToolCallback` objects. A `ToolCallback` is Spring AI's internal, unified representation of "something the model can invoke" — regardless of where that something actually comes from.

### 3.2 Where the one you're injecting actually comes from

This is the part that's easy to miss, because you never explicitly construct it: **Spring AI's MCP client auto-configuration builds and registers a `ToolCallbackProvider` bean for you automatically**, as a direct consequence of your `application.yml` configuration from the previous guide. Concretely, at application startup:

1. The MCP client auto-configuration reads `spring.ai.mcp.client.*` and establishes a connection to every configured server — your Streamable HTTP connection named `abhay`, and (if active) the `stdio`-spawned filesystem server.
2. For **each** connected server, it sends a `tools/list` request — exactly the protocol-level operation described in the MCP architecture guide — and receives back that server's self-describing list of tools (names, descriptions, JSON Schemas).
3. Every tool from every connected server gets wrapped into Spring AI's own `ToolCallback` representation — the same internal shape a locally-defined `@Tool`-annotated method produces, meaning from this point on, **an MCP-discovered tool and a hand-written `@Tool` method are indistinguishable to the rest of Spring AI's tool-calling machinery.**
4. All of those wrapped callbacks, across every connected server, are collected into **one single `ToolCallbackProvider` bean** — this is the exact bean your constructor receives.

### 3.3 Why this matters architecturally

This is the same "it's just a `ToolCallback`, Spring AI doesn't care where it came from" idea that made custom `DocumentRetriever`s and custom `Advisor`s pluggable earlier in this series — MCP tools aren't a special case bolted awkwardly onto the tool-calling system; they're first-class participants in the exact same mechanism `@Tool`-annotated methods use. This is *why* a single `.defaultTools(toolCallbackProvider)` call is enough to expose potentially dozens of tools, spanning multiple remote servers and local subprocesses, with no per-tool registration code anywhere.

---

## 4. `.defaultTools(...)` — the two different shapes it accepts

It's worth contrasting this against the `@Tool` pattern from the architecture guide, because the same method accepts both:

```java
// Shape 1 — a plain Java object whose methods are annotated with @Tool.
// Spring AI reflects over it to find and register each annotated method.
.defaultTools(orderTools)

// Shape 2 — a ToolCallbackProvider, which already contains a whole ARRAY of
// pre-built ToolCallbacks (potentially from several different MCP servers).
// Spring AI simply registers every callback the provider returns.
.defaultTools(toolCallbackProvider)
```

Both end up producing the exact same internal thing — a set of `ToolCallback`s the model can invoke — just arrived at via two different starting points: reflection over your own annotated code, versus tools discovered dynamically from one or more external MCP servers. You can even pass **both** at once, mixing your own local `@Tool` methods with MCP-discovered ones in the same `ChatClient`:

```java
this.chatClient = builder
        .defaultTools(orderTools)           // your own hand-written @Tool methods
        .defaultTools(toolCallbackProvider)  // every tool discovered from MCP servers
        .build();
```

---

## 5. The complete, end-to-end flow for a single request

```
1. HTTP GET /chat?q=List the files in my mcp folder

2. ClientController.chat("List the files in my mcp folder") runs,
   calling chatClient.prompt(q).call().content()

3. ChatClient sends the prompt to the model, ALONGSIDE the full list of
   registered tools — which, thanks to .defaultTools(toolCallbackProvider),
   includes every tool discovered from every connected MCP server:
     - Tools from the remote library-mcp-server (Streamable HTTP, port 8086)
     - Tools from the locally-spawned filesystem server (stdio), if configured
       e.g. list_directory, read_file, write_file

4. The model reads the question, recognizes it needs the filesystem server's
   list_directory tool, and responds with a tool-call request rather than
   a direct answer.

5. Spring AI's tool-calling machinery (the exact same advisor-driven loop from
   the architecture guide — intercept, execute, feed the result back) sees
   that this particular ToolCallback is backed by an MCP connection, and
   routes the actual invocation through the correct MCP CLIENT instance
   (recall: one client per server) — in this case, the one managing the
   stdio-spawned filesystem process.

6. That MCP client writes a tools/call JSON-RPC request to the filesystem
   server process's stdin. The filesystem server performs the real
   directory listing and writes the result back over its stdout.

7. The result flows back up through the MCP client, back into Spring AI's
   tool-calling loop, and gets fed back to the model as a tool result.

8. The model reads that result and generates its final, natural-language
   answer — e.g. "Your mcp folder contains: notes.txt, report.pdf, ..."

9. That final text is what chatClient.prompt(q).call().content() returns,
   and what the controller sends back as the plain HTTP response body.
```

**The crucial realization:** none of steps 3 through 8 are visible anywhere in your controller's code. The entire discovery-to-invocation pipeline — connecting to MCP servers, listing their tools, deciding which one to call, routing the call to the right client, executing it, and feeding the result back — is handled entirely by Spring AI's auto-configuration and tool-calling machinery, triggered by nothing more than the presence of a `ToolCallbackProvider` bean and one `.defaultTools(...)` call.

---

## 6. Testing it

```bash
curl "http://localhost:8080/chat?q=What tools do you have access to?"
curl "http://localhost:8080/chat?q=List the files in my mcp folder"
```

The first query is a good sanity check before anything else — if the MCP connections are wired up correctly, the model's answer should actually mention the real tools it was given (file-listing, file-reading, whatever your connected servers expose), confirming the whole discovery pipeline from §3.2 genuinely worked end to end, not just that the application started without errors.

---

## 7. A security note worth taking seriously

The moment you register a `ToolCallbackProvider` built from something like the filesystem MCP server, **every request to this `/chat` endpoint potentially has the ability to read and write real files on the machine running the application** — because that's literally what the filesystem server's tools do, and the model is the one deciding when to invoke them, not you, on a per-request basis. A few concrete implications:

- **This endpoint is, functionally, a remote-code-adjacent capability**, not just a chatbot — anyone who can reach `/chat` can potentially get the model to read or write files within whatever directory the filesystem server was scoped to (`C:\...\Desktop\mcp` in your `mcp-servers.json`).
- **Scope the underlying MCP server as narrowly as possible.** The filesystem server already takes a directory argument specifically so it can't touch the whole disk — make sure that directory genuinely contains nothing sensitive.
- **Don't expose a controller wired this way to the public internet without authentication/authorization in front of it.** This is exactly the kind of endpoint that needs to sit behind whatever auth layer the rest of your application already uses — an anonymous `GET /chat` with filesystem tools attached is a real risk, not a toy.
- **Consider a `SafeGuardAdvisor` or a custom guardrail advisor (from the advisors guide) in front of tool-enabled `ChatClient`s** if the connected MCP servers can perform any action with real-world consequences — the model deciding autonomously *when* to call a powerful tool is exactly the scenario those guardrails exist for.

---

## 8. Recap

`ToolCallbackProvider` is the auto-configured bean that turns "I have some MCP servers connected" into "the model can actually use their tools" — built automatically by discovering every connected server's `tools/list` response and wrapping each tool into Spring AI's own `ToolCallback` format, the exact same internal shape a hand-written `@Tool` method produces. `.defaultTools(toolCallbackProvider)` is the one line that takes that entire discovered tool set and makes it available to every call through a given `ChatClient` — after which, your actual service/controller code looks completely ordinary, with zero MCP-specific logic anywhere in it, because the whole discovery-and-invocation pipeline lives entirely inside Spring AI's auto-configuration and tool-calling machinery, invisible to the code you actually write.
