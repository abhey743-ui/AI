# Spring AI MCP Implementation — `stdio` vs Streamable HTTP, End to End

## 1. The core difference, in one picture

```
STDIO                                  STREAMABLE HTTP
─────                                  ────────────────
Client process                         Client process
     │                                       │
     │ spawns as a CHILD PROCESS             │ sends HTTP POST / GET
     ▼                                       ▼
Server process                         Server process (remote, its own process,
(same machine, no network)             its own machine/container, reachable
communicates via stdin/stdout          over the network at a URL)
```

- **`stdio`** — the client **launches the server as a subprocess** on the same machine, and the two processes talk by writing JSON-RPC messages to each other's **standard input and standard output** — literally the same stdin/stdout a terminal command uses. There is no network involved at all; it's inter-process communication on one machine.
- **Streamable HTTP** — the server is a **separately running, independently deployed process** (potentially on a completely different machine), reachable over the network at a URL. The client sends JSON-RPC messages as HTTP POST requests to that URL, with the server able to stream a response back (optionally using Server-Sent Events) rather than being strictly request/response.

This single distinction — "spawned as a local child process" vs. "an independent service reachable over a network" — is what drives every other difference between the two.

---

## 2. Dependencies — client side

```xml
<!-- Standard client: supports stdio, SSE, Streamable-HTTP, and Stateless Streamable-HTTP, all from one starter -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-mcp-client</artifactId>
</dependency>
```

For production deployments specifically using Streamable HTTP/SSE, Spring AI recommends the reactive variant instead:

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-mcp-client-webflux</artifactId>
</dependency>
```

Both starters let a single Spring Boot application connect to **multiple MCP servers simultaneously**, mixing transports freely — some connections over `stdio`, others over Streamable HTTP, all configured declaratively in `application.yml`. This is exactly why your client config has both a `streamable-http` block and a commented-out `stdio` block sitting side by side — they're not mutually exclusive, they're two independent connection types the same client application can maintain at once.

---

## 3. Dependencies — server side

```xml
<!-- For a Streamable HTTP server, built on Spring MVC -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-mcp-server-webmvc</artifactId>
</dependency>
```

A `stdio` server, by contrast, **doesn't need a web dependency at all** — there's no HTTP endpoint to serve, since communication happens over stdin/stdout, not a network socket. This is exactly why your `McpServer` config has:

```yaml
spring:
  main:
    web-application-type: none
```

`web-application-type: none` tells Spring Boot explicitly: **don't start an embedded web server for this application at all.** For a `stdio` MCP server, that's correct and intentional — the whole point is that there's no HTTP server to run; the application just sits there, reading from stdin and writing to stdout, exactly like any ordinary command-line program would.

---

## 4. Your `McpClient` config, explained line by line

```yaml
spring:
  application:
    name: McpClient
  ai:
    mcp:
      client:
        enabled: true
        streamable-http:
          connections:
            abhay:
              url: http://localhost:8086
              endpoint: /mcp
#        stdio:
#          servers-configuration: classpath:mcp-servers.json
```

- **`spring.ai.mcp.client.enabled: true`** — turns on the MCP client auto-configuration at all; without this, none of the connection blocks below would do anything.
- **`streamable-http.connections.abhay`** — `abhay` here is just a **name you chose** for this particular connection — it's how this specific server connection gets identified internally (and in logs) if you have more than one. Under it:
  - **`url: http://localhost:8086`** — the base address of the remote MCP server to connect to.
  - **`endpoint: /mcp`** — the specific path on that server where the MCP protocol endpoint actually lives (the server side needs to expose this same path — see §5).
- **The commented-out `stdio` block** — had it been active, `servers-configuration: classpath:mcp-servers.json` tells the client to read a JSON file (in the exact shape used by Claude Desktop's own config, and other MCP hosts) describing one or more servers to **launch as subprocesses**, rather than connect to over the network.

### 4.1 The `mcp-servers.json` file, explained

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "cmd.exe",
      "args": [
        "/c",
        "npx",
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "C:\\Users\\VICTUS\\OneDrive\\Attachments\\Desktop\\mcp"
      ]
    }
  }
}
```

This defines one named server, `filesystem`, and tells the client **exactly what command to run to start it**:
- **`command: "cmd.exe"`** — on Windows, this spawns the Windows command shell as the actual process.
- **`args`** — the arguments passed to that command, which here chain together to run: `npx -y @modelcontextprotocol/server-filesystem C:\...\mcp`. `npx` downloads (if not already cached) and runs the official, publicly published **Filesystem MCP Server** — a ready-made MCP server (written in Node.js, by the MCP project itself) that exposes tools for reading/writing/listing files inside the one directory you give it (here, a folder on the desktop).
- **The `-y` flag** tells `npx` to auto-confirm installing the package without prompting.

**What actually happens at runtime:** when your Spring Boot `McpClient` application starts up and this `stdio` block is active, Spring AI's MCP client auto-configuration literally runs this exact command as a **child process** of your Java application, then talks to that child process's stdin/stdout using the MCP protocol — discovering its tools (`list_directory`, `read_file`, `write_file`, and so on, depending on that server's implementation) and making them available to your `ChatClient` exactly like any other MCP server's tools.

This is a genuinely important realization: **the server doesn't have to be something you wrote at all.** It can be any pre-built, third-party MCP server published by anyone, and your Java client launches and talks to it as a subprocess with zero code of your own beyond this JSON configuration.

---

## 5. Your `McpServerRemote` config, explained line by line

```yaml
spring:
  application:
    name: McpServerRemote
  ai:
    mcp:
      server:
        protocol: STREAMABLE
        name: library-mcp-server
        version: 1.0.0

server:
  port: 8086
```

- **`spring.ai.mcp.server.protocol: STREAMABLE`** — this is the setting that tells Spring AI's server auto-configuration to expose the MCP endpoint over **Streamable HTTP**, rather than over `stdio`. With this set (and the `spring-ai-starter-mcp-server-webmvc` dependency from §3 present), Spring Boot starts an actual embedded web server and registers an MCP-protocol-handling endpoint on it.
- **`name` / `version`** — identity information this server reports to any connecting client during the initial handshake — shown to the client/host as "which server, which version" it's talking to, useful for logging/debugging on the client side.
- **`server.port: 8086`** — the ordinary Spring Boot embedded-server port property — nothing MCP-specific here, it's the same property that configures the port for any Spring web application. This is exactly why your client's `url: http://localhost:8086` in §4 points at this same port — that's the actual network address this server is listening on.
- **The MCP endpoint path** defaults to `/mcp` (matching your client's `endpoint: /mcp`) unless overridden via `spring.ai.mcp.server.streamable-http.mcp-endpoint`.

This server, unlike the filesystem example in §4, is one **you write yourself** — a normal Spring Boot application where you define `@Tool`-annotated methods (exactly like the tool-calling pattern from the architecture guide earlier in this series), and Spring AI's MCP server auto-configuration automatically exposes those methods as MCP tools, reachable by any connecting client over this Streamable HTTP endpoint.

---

## 6. Your `McpServer` (stdio) config, explained line by line

```yaml
spring:
  application:
    name: McpServer
  main:
    web-application-type: none
  banner-mode: off

logging:
  level:
    root: ERROR
```

- **`web-application-type: none`** — already covered in §3: no embedded web server starts at all, since `stdio` needs none.
- **`banner-mode: off`** — turns off Spring Boot's ASCII-art startup banner. This matters specifically *because* of how `stdio` works: that banner is normally printed straight to **standard output** — the exact same channel the MCP protocol is using to send JSON-RPC messages to the client. If the banner (or any other stray console output) got mixed into stdout, the client would receive corrupted, non-JSON-RPC garbage on that channel and the connection would break.
- **`logging.level.root: ERROR`** — this is the single most important, easy-to-miss detail about building a `stdio` server correctly, and it's for the exact same reason as the banner setting: **Spring Boot's default logging configuration writes log output to the console (stdout) by default.** On a `stdio` MCP server, stdout is a sacred, protocol-only channel — every single byte written to it needs to be a valid MCP JSON-RPC message, nothing else. A normal `INFO`-level Spring Boot startup (bean creation logs, "Started Application in 2.3 seconds," etc.) would all land on stdout and corrupt the protocol stream. Setting the root log level all the way up to `ERROR` is a blunt but effective way to suppress essentially all of that routine startup noise, leaving stdout clean for MCP traffic only. *(A more surgical alternative some setups use is redirecting the logging output to a file instead of the console entirely, rather than just raising the level — but raising the level, as you've done, is the simpler, commonly used fix.)*

**This is worth internalizing as a general rule:** any time you build a `stdio` MCP server, you must be deliberate about keeping stdout clean of anything that isn't a protocol message — logging configuration isn't an afterthought here, it's a correctness requirement.

---

## 7. The full request flow — Streamable HTTP

```
1. McpClient app starts, spring.ai.mcp.client.enabled=true triggers auto-configuration.
2. For the "abhay" connection, the client opens an HTTP connection to
   http://localhost:8086/mcp (your McpServerRemote, already running separately).
3. The client sends an initial handshake / tools-list request as an HTTP POST
   to that endpoint.
4. McpServerRemote (a normal Spring Boot web app, listening on port 8086)
   receives it through its MCP endpoint, and responds with its exposed tools
   (whatever @Tool-annotated methods exist in that application), as JSON.
5. The client relays that tool list up to whatever ChatClient/host application
   is using it — those tools now appear alongside any others in the model's
   available tool set.
6. When the model decides to call one of those tools, the client sends another
   HTTP POST (a tools/call request) to the same endpoint; the server executes
   the real underlying logic and responds with the result, again over HTTP.
```

Two genuinely separate operating-system processes the whole time, communicating purely over the network — the server could just as easily be on a different machine entirely, in a different data center, with zero change to this flow.

---

## 8. The full request flow — stdio

```
1. McpClient app starts, with the stdio block active and pointing at
   mcp-servers.json.
2. Spring AI's client auto-configuration reads that JSON file and, for the
   "filesystem" entry, literally executes:
       cmd.exe /c npx -y @modelcontextprotocol/server-filesystem C:\...\mcp
   as a CHILD PROCESS of the running Java application.
3. The client writes JSON-RPC requests to that child process's STANDARD INPUT,
   and reads JSON-RPC responses from that child process's STANDARD OUTPUT —
   there is no network socket, no URL, no port involved anywhere in this flow.
4. The spawned filesystem server process responds (over its own stdout) with
   its exposed tools — e.g. tools for listing, reading, and writing files
   within the one directory it was told to operate in.
5. The client relays that tool list up to the host/ChatClient, exactly as in
   the Streamable HTTP flow — from the model's perspective, it makes no
   difference at all which transport actually delivered these tools.
6. When the model calls one of those tools, the client writes that request to
   the child process's stdin; the filesystem server performs the real file
   operation and writes the result back over its stdout.
7. If/when the McpClient Java application shuts down, the child process it
   spawned is normally terminated along with it — the two are tied together
   for the lifetime of the parent application.
```

The critical thing to notice: **the model-facing experience is identical regardless of transport.** `ChatClient`/the host never has to know or care whether a given tool came from a local subprocess over `stdio` or a remote service over Streamable HTTP — the transport is entirely an implementation detail of how the client and server physically exchange bytes, completely invisible above that layer.

---

## 9. `stdio` vs Streamable HTTP — side by side

| | `stdio` | Streamable HTTP |
|---|---|---|
| **Where the server runs** | Spawned as a child process on the *same machine* as the client | An independent process, potentially on a *different machine* entirely |
| **Communication channel** | Standard input / standard output of the spawned process | HTTP requests (POST, optionally streamed via SSE) over the network |
| **Needs a web server/port?** | No — `web-application-type: none` | Yes — an embedded web server, listening on a port |
| **Can multiple clients share one server instance?** | No — typically one client per spawned process | Yes — a single remote server can serve many different clients concurrently |
| **Scales horizontally (load balancer, multiple instances)?** | Not applicable — it's a local subprocess, not a deployed service | Yes — this is exactly why modern MCP revisions made Streamable HTTP stateless, specifically to support this |
| **Logging/console output risk** | High — any stray stdout output corrupts the protocol; logging must be carefully suppressed/redirected | None — logging happens on its own normal console/file as usual, since the MCP protocol travels over HTTP, not shared stdout |
| **Typical use case** | Local developer tools, CLI-style integrations, wrapping an existing local command-line tool (like the filesystem server) | Any server you want reachable remotely — a real microservice, something used by multiple different AI applications, anything behind a gateway |
| **Setup complexity** | Simple for a single local tool — no networking, no ports, no TLS | Requires actual deployment — a running service, a reachable URL, usually behind proper auth in production |

---

## 10. Which one should you actually use, and when

- **Use `stdio`** when the "server" is really a **local tool** you want an AI application to use on the same machine — wrapping an existing CLI tool, giving a desktop AI assistant access to the local filesystem (exactly like your `mcp-servers.json` example), or during early local development of a server you're actively writing and want to test quickly without deploying anything.
- **Use Streamable HTTP** for literally anything resembling the hospital/microservices architecture from the earlier MCP guide — any server that needs to be reachable by more than one client, deployed as its own service, scaled independently, sat behind a gateway, or used by an AI application running on a completely different machine than the server itself. This is the transport that actually fits a real microservices/enterprise deployment.

A single AI application is free to use **both at once** — exactly as your `McpClient` config demonstrates, with a `streamable-http` connection to a remote server and a (commented-out) `stdio` connection to a locally spawned one, both discoverable and usable by the same `ChatClient` simultaneously, with the model never needing to know the difference.

## 11. Recap

`stdio` and Streamable HTTP are two different answers to "how do the client and server physically exchange bytes" — `stdio` spawns the server as a local child process and talks over its stdin/stdout (fast, simple, but strictly local, and demanding careful logging hygiene since stdout is a protocol-only channel), while Streamable HTTP treats the server as an independently deployed network service reached over ordinary HTTP (the transport that actually fits remote, multi-client, horizontally-scaled real-world deployments). Spring AI's client starter supports configuring both kinds of connections side by side in the same `application.yml`, and above the transport layer, everything — tool discovery, tool invocation, how the model sees and uses what's offered — behaves identically regardless of which one is carrying the bytes underneath.
