# MCP Inspector — The X-Ray Tool for MCP Servers

## 1. What it is, and the problem it solves

**MCP Inspector** is the official, open-source developer tool from the Model Context Protocol project for **testing and debugging an MCP server directly — without needing a full AI application (a host) in front of it at all.**

Think about the problem this solves concretely: in every guide so far in this series, the *only* way to actually see whether your MCP server worked was to wire it into a real `ChatClient`, ask the model a question, and hope it decided to call the right tool. If something was broken — a wrong schema, a bad connection, a tool silently failing — you had no direct way to isolate *where* the failure was: the server itself? The client configuration? The model's tool-selection reasoning? MCP Inspector exists specifically to remove the model and the host from that loop entirely, letting you talk to your server **directly**, call its tools with hand-picked inputs, and watch the raw protocol traffic go by — the same way **Postman** lets you test a REST API directly without a frontend UI in front of it. That Postman comparison is the single best mental model for what this tool is.

The "x-ray" framing is apt: it doesn't change or wrap your server in any way — it just gives you a transparent window directly into exactly what your server advertises and exactly how it responds, at the raw protocol level.

---

## 2. What it's actually made of

MCP Inspector is two cooperating pieces, both maintained in the same open-source repository (`github.com/modelcontextprotocol/inspector`):

- **Inspector Client** — a React-based web UI that runs in your browser, where you actually click around, select tools, fill in test inputs, and read results.
- **Inspector Proxy** — a small Node.js server that sits between the browser UI and your actual MCP server, handling the real protocol connection on the UI's behalf (this exists because a browser can't directly spawn a local `stdio` subprocess or always speak raw MCP transport itself — the proxy does that part and relays results up to the UI over its own connection).

Both pieces start together with a single command, and by default the proxy listens on port `6277`, with the web UI typically served on a port like `6274` (configurable via environment variables if either is already in use).

---

## 3. Installing and running it

No installation step is strictly required — it runs directly through `npx`:

```bash
npx @modelcontextprotocol/inspector
```

This launches the Inspector with no server pre-connected — you'd then enter a server command or URL manually inside the UI. More commonly, you launch it **pointed directly at a server** from the start:

```bash
# Inspect a server you'll run as a local command (stdio)
npx @modelcontextprotocol/inspector node path/to/server/index.js

# Inspect a published npm package server directly
npx -y @modelcontextprotocol/inspector npx -y @modelcontextprotocol/server-filesystem /Users/you/Desktop

# Inspect a remote server already running, over Streamable HTTP
npx @modelcontextprotocol/inspector http://localhost:8086/mcp
```

If the default ports are already taken:

```bash
PORT=8080 PROXY_PORT=8081 npx @modelcontextprotocol/inspector
```

---

## 4. Connecting it to your own Spring AI servers from this series

This is where it becomes immediately practical, using the exact two servers built earlier in this series:

### 4.1 Your Streamable HTTP server (`McpServerRemote`, port 8086)

Since this server is already a running, independently reachable HTTP service (exactly the kind of server Streamable HTTP is built for), you simply point the Inspector at its URL **after starting the Spring Boot application normally**:

```bash
# 1. Start your Spring Boot MCP server as usual
mvn spring-boot:run   # (running McpServerRemote, listening on :8086)

# 2. In a separate terminal, launch the Inspector against it
npx @modelcontextprotocol/inspector http://localhost:8086/mcp
```

The Inspector's UI opens in your browser, already connected — no model, no `ChatClient`, no `ToolCallbackProvider` anywhere in this loop. You're talking to your `library-mcp-server` directly.

### 4.2 Your `stdio` server (`McpServer`)

Since a `stdio` server is normally launched as a child process by whatever's connecting to it, you point the Inspector at the **command** that starts your Spring Boot jar, and the Inspector itself becomes the process that spawns and manages it:

```bash
npx @modelcontextprotocol/inspector java -jar target/mcp-server-0.0.1-SNAPSHOT.jar
```

The Inspector launches your jar as a subprocess exactly the way your real `McpClient` would have, talks to it over stdin/stdout, and — this is a genuinely useful side effect — **immediately tells you if your logging/banner hygiene from the `stdio` guide was actually done correctly.** If anything other than valid MCP JSON-RPC messages leaks onto stdout (a stray `INFO` log line, Spring Boot's banner), the Inspector will show a broken or garbled connection right away, which is a far faster way to catch that class of bug than debugging it through a full client/model round trip.

---

## 5. What the interface actually shows you

### 5.1 Server connection pane
Lets you pick (or confirm) the transport — `stdio`, SSE, or Streamable HTTP — and shows live connection status: is the server actually reachable, did the initial handshake succeed, what protocol version is in use.

### 5.2 Tools tab
Lists every tool your server exposes via `tools/list`, showing each one's **exact schema** — its name, its description (the natural-language text the *model* would read to decide when to use it), and its JSON Schema input definition. Critically, it lets you **invoke any tool directly**, typing in test arguments by hand, and immediately see the raw result the server returns — this is the single most useful feature for verifying "does this tool actually work correctly" completely independently of whether a model would ever choose to call it correctly.

### 5.3 Resources tab
Lists resources the server exposes (recall from the MCP architecture guide: read-only context data, not model-invoked actions), lets you inspect their metadata and actual content, and test subscription behavior if the server supports resource-change notifications.

### 5.4 Prompts tab
Shows any reusable prompt templates the server exposes, including their declared arguments, and lets you test generating a filled-in prompt with sample values — useful for verifying a prompt template renders exactly as intended before a real user ever picks it.

### 5.5 Notifications / logs pane
Shows the raw log messages and notifications the server has sent over the connection — genuinely the closest thing to literally watching the JSON-RPC traffic go by in real time, which is exactly the "x-ray" experience being asked about: nothing hidden, nothing abstracted, just the actual protocol messages as they happen.

---

## 6. The recommended development workflow

The MCP project's own documentation describes a specific iterative loop worth adopting as a habit whenever you're building a server:

```
1. Launch the Inspector against your server.
2. Make a change to your server's code.
3. Rebuild the server.
4. Reconnect the Inspector (it doesn't auto-reload when your server changes).
5. Test the specific feature you just changed.
6. Deliberately test edge cases — invalid inputs, missing required
   arguments — to confirm your server's error handling actually behaves
   the way you expect, not just the happy path.
```

The emphasis on step 6 is worth internalizing: it's far easier (and far faster) to discover that a tool mishandles a missing argument by typing a deliberately broken test case directly into the Inspector's Tools tab, than to discover it later because a model happened to generate a malformed call during a real conversation.

---

## 7. CLI mode — for automation and CI pipelines

Beyond the interactive web UI, the Inspector also has a scriptable CLI mode, letting you validate a server's behavior as part of an automated pipeline rather than only by hand:

```bash
# Validate the server's advertised tool list hasn't broken/changed unexpectedly
npx @modelcontextprotocol/inspector --cli node build/index.js --method tools/list

# Call a specific tool with a specific argument, and assert on the result
npx @modelcontextprotocol/inspector --cli node build/index.js \
    --method tools/call --tool-name get_doctor_availability --tool-arg doctor=Iyer
```

This is exactly the kind of check worth adding to a CI pipeline for any MCP server you maintain: run it on every commit, and catch a breaking change to a tool's schema or behavior automatically, rather than discovering it only once a connected AI application starts behaving strangely in production.

---

## 8. Debugging with it — the practical payoff

Recall the specific risk flagged in the `stdio` guide: any stray output on stdout corrupts the protocol stream. If your real `McpClient` application ever fails to connect to a `stdio` server with a vague, unhelpful error, the Inspector is the fastest way to isolate why — launch the exact same server command directly through the Inspector, and you'll either see a clean connection (meaning the problem is in your client-side configuration) or a garbled/broken one (meaning the problem is genuinely in the server's console output hygiene) — cleanly separating which half of the system is actually at fault, rather than guessing.

More generally, this is the first tool to reach for whenever any of these questions come up:
- **"Is my server even running and reachable at all?"** — the connection pane answers this immediately, independent of any client or model.
- **"Why isn't this tool working?"** — invoke it directly with the Tools tab and read the raw result, instead of inferring what happened from a model's final natural-language answer three layers removed from the actual failure.
- **"Did my schema change break something?"** — the Tools tab shows you the exact JSON Schema being advertised right now, which is the quickest way to catch a typo or an unintended breaking change before any client ever encounters it.

---

## 9. Where it fits relative to other testing approaches

| Approach | What it actually tests | When to use it |
|---|---|---|
| **MCP Inspector** | Protocol-level correctness: does the server advertise the right tools, do they execute correctly for a given input | First stop, during active development of a server — before any AI application is involved at all |
| **A real `ChatClient` + model** | End-to-end behavior: does the model correctly *choose* to call the right tool at the right time, given a natural-language question | After you're confident the server itself works, to validate the model's tool-selection reasoning |
| **CI-integrated Inspector CLI** | Regression protection: did a code change silently break a tool's schema or behavior | Ongoing, on every commit, for any MCP server you maintain long-term |

Note the clean separation of concerns here: the Inspector can tell you a tool is correctly implemented and correctly described; it **cannot** tell you whether a model will reliably choose to call that tool at the right moment in a real conversation — that's a separate, model-reasoning concern the Inspector deliberately doesn't attempt to test.

---

## 10. Recap

MCP Inspector is the official "Postman for MCP" — a browser UI plus a small local proxy that lets you connect directly to an MCP server (over `stdio` or Streamable HTTP, the exact two transports covered earlier in this series), see precisely what tools/resources/prompts it advertises, invoke them by hand with arbitrary test inputs, and watch the raw protocol traffic and logs in real time — all without needing a model or a full host application anywhere in the loop. It's the right first stop whenever you're building or debugging a server, it directly exposes the exact class of bug the `stdio` logging-hygiene guide warned about, and its CLI mode lets the same checks run automatically in a CI pipeline, catching a broken tool schema the moment it happens rather than only once it quietly confuses a model in production.
