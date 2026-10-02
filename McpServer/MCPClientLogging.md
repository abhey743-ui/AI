# MCP Client Logging with `@McpLogging` — Complete Guide

> Your code (the starting point):
>
> ```java
> package com.McpClient.McpClient;
>
> import io.modelcontextprotocol.spec.McpSchema;
> import org.slf4j.Logger;
> import org.slf4j.LoggerFactory;
> import org.springframework.ai.mcp.annotation.McpLogging;
> import org.springframework.stereotype.Component;
>
> @Component
> public class McpLoggingLogic {
>     private static Logger logger = LoggerFactory.getLogger(McpLoggingLogic.class);
>
>     @McpLogging(clients = "eazybytes")
>     public void serviceLogging(McpSchema.LoggingLevel loggingLevel, String source, String message) {
>         logger.info(loggingLevel + " " + source + " " + message);
>     }
> }
> ```
>
> This file explains: what this code does line by line, **why** MCP logging exists, **how and when** the server sends the logs, **what information** is inside each log, and **what to do in production**.
>
> Written for Spring AI 2.0.x (current stable) and the MCP specification.

---

## Table of Contents

1. The problem MCP logging solves
2. Big picture: who sends, who receives
3. Your code, line by line
4. What the server sends (the exact message)
5. The 8 log levels
6. When exactly does the server send logs?
7. The full flow (sequence diagram)
8. How the server side sends a log (Spring AI example)
9. Configuring your client (yml) — the `clients` name must match
10. Controlling how much you receive (`logging/setLevel`)
11. Three ways to write the handler
12. Improving your handler (production-quality version)
13. What to use in production (important)
14. Security rules for logs
15. Troubleshooting: "I don't see any logs"
16. Quick summary

---

## 1. The problem MCP logging solves

In MCP, your Spring Boot app is the **client**. The tools run in a **different program** — the MCP **server** — which may be on another machine, inside Docker, or in another company's cloud.

When the LLM calls a tool, your client sends a request and waits. From the client's point of view, the server is a **black box**:

```
Client:  "run tool: fetch-customer(42)"
          ... waiting ...
Server:  "{ result }"        <-- you only see the final answer
```

If the tool is slow, retries something, hits a warning, or fails in a strange way, **your client sees nothing about what happened inside the server**. You would have to go and open the server's own log files.

**MCP logging** is a standard way for the server to say, while it works:

> "Hey client, here is a log message: level = warning, from = payment-module, text = 'retrying call 2 of 3'."

Your client receives these messages and decides what to do: print them, store them, show them to the user, raise an alert, etc.

Think of it like this: the server is a chef in a closed kitchen. The tool result is the final plate. MCP logging is the chef shouting short updates through a small window: "onions are burning!", "waiting for the oven".

---

## 2. Big picture: who sends, who receives

| Part | Role |
|---|---|
| **MCP Server** | Produces log messages (this is the **sender**). It must announce that it supports logging (the `logging` capability). |
| **MCP Client (your app)** | Receives the messages and handles them (this is the **receiver**). In Spring AI you do that with `@McpLogging`. |
| **Transport** | The channel carrying the messages (STDIO, SSE or Streamable HTTP). |

Direction: **server → client**. The client can only control the *minimum level* it wants (section 10).

Important: this is **not** a replacement for the server's own logging (files, ELK, etc.). It is a small *channel* so the client can see what the server wants to tell it.

---

## 3. Your code, line by line

```java
@Component
```
Makes this class a Spring bean. The MCP client starter scans beans for MCP annotations (the property `spring.ai.mcp.client.annotation-scanner.enabled` is `true` by default).

```java
@McpLogging(clients = "eazybytes")
```
- Marks the method as a **handler for log messages coming from an MCP server**.
- `clients = "eazybytes"` means: *only for the MCP connection named `eazybytes`*. The Spring docs say every MCP client annotation **must** have `clients`, and the value must match the connection name in your configuration (section 9).

```java
public void serviceLogging(McpSchema.LoggingLevel loggingLevel, String source, String message)
```
Spring AI lets you receive the log in two forms. You used the "individual parameters" form: **level, logger, data** (in this order). So:

| Your parameter | MCP field | Meaning |
|---|---|---|
| `loggingLevel` | `level` | Severity (`DEBUG`, `INFO`, `WARNING`, `ERROR`...) |
| `source` | `logger` | Name of the part of the server that wrote the log (e.g. `database`, `payment`). **Optional** in the protocol, so it can be `null`. |
| `message` | `data` | The content of the log |

Parameter *names* don't matter; the **order and types** do.

```java
logger.info(loggingLevel + " " + source + " " + message);
```
Writes the received log into **your client's own log** using SLF4J. Example output:

```
2026-10-03T10:15:42.101  INFO  c.M.McpClient.McpLoggingLogic : WARNING payment-service Retrying call 2 of 3
```

> Notice: the server's `WARNING` is only *text* inside an `INFO` line. Your logging system will treat every server log as INFO. Section 12 fixes this.

---

## 4. What the server sends (the exact message)

On the wire, MCP uses JSON-RPC. A log is a **notification** (a one-way message: no reply expected). Method name: `notifications/message`.

Example from the MCP specification:

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/message",
  "params": {
    "level": "error",
    "logger": "database",
    "data": {
      "error": "Connection failed",
      "details": {
        "host": "localhost",
        "port": 5432
      }
    }
  }
}
```

So each log carries exactly three useful things:

| Field | Required? | What it is |
|---|---|---|
| `level` | Yes | One of the 8 syslog severities |
| `logger` | Optional | Name of the logger/component on the server |
| `data` | Yes | Any JSON-serializable value: a plain string **or** a structured object (like above) |

There is **no** timestamp, **no** request ID, and **no** server name in the message. If you need those, you must add them yourself on the client (e.g. add the connection name, add `now()`), or put them inside `data` on the server.

---

## 5. The 8 log levels

MCP follows the standard syslog levels (RFC 5424). Lowest to highest:

| Level | Meaning | Example (from the spec) |
|---|---|---|
| `debug` | Detailed debugging info | Function entry/exit |
| `info` | General information | Operation progress updates |
| `notice` | Normal but significant | Configuration changes |
| `warning` | Warning condition | Deprecated feature used |
| `error` | Error condition | Operation failed |
| `critical` | Critical condition | A system component failed |
| `alert` | Action must be taken immediately | Data corruption detected |
| `emergency` | System unusable | Complete system failure |

In Java these are the enum `McpSchema.LoggingLevel` (`DEBUG`, `INFO`, `NOTICE`, `WARNING`, `ERROR`, `CRITICAL`, `ALERT`, `EMERGENCY`).

---

## 6. When exactly does the server send logs?

The protocol does **not** fix exact moments. It is the **server author's choice**. In practice, servers send logs when something interesting happens:

| Moment | Typical log |
|---|---|
| **Inside a tool call** (most common) | "Started processing 500 records" (info), "API slow, retrying" (warning), "Payment API failed" (error) |
| **Long-running work** | Progress updates (often together with progress notifications) |
| **Resource access / prompt handling** | "Loaded file X", "Cache miss" |
| **Startup / config changes** | "Config reloaded" (notice) |

Rules from the specification that decide *whether* a log is actually sent:

1. **The server must declare the `logging` capability** during the initial handshake. If it does not, it should not send log notifications.
2. **Level filter:** the client **may** send `logging/setLevel`. After that, the server sends only messages at that level **and above**. The spec's example: set `error` → the server sends only error and higher.
3. If the client never sets a level, the server decides on its own which logs to send. (This is how the spec describes the default; behavior can vary by server implementation.)
4. **Stateful connection needed (practical point):** in Spring AI, a tool method gets the logging ability through the server *exchange* object. The docs describe the stateless mode as giving only a lightweight transport context *without* the full exchange functionality — so do not expect logs from a **stateless** server. Use STDIO, SSE or (stateful) Streamable HTTP.
5. Logs are sent **in the same session** as your client's connection — they come asynchronously, possibly while a tool call is still running.

---

## 7. The full flow (sequence diagram)

```
 Your Spring App (MCP Client)                         MCP Server "eazybytes"
        |                                                     |
        |-- initialize ------------------------------------->|
        |<-- capabilities: { tools:{}, logging:{} } ----------|   [1] server says "I can send logs"
        |                                                     |
        |-- logging/setLevel { level: "info" } ------------->|   [2] (optional) "send info and above"
        |<-- {} (empty result) --------------------------------|
        |                                                     |
        |-- tools/call  fetch-customer(42) ----------------->|   [3] LLM asked for a tool
        |                                                     |
        |<-- notifications/message {info, "customer-svc",     |   [4] server logs WHILE working
        |        "Looking up customer 42"} ------------------|
        |        |-> Spring AI finds @McpLogging(clients="eazybytes")
        |        |-> calls serviceLogging(INFO, "customer-svc", "Looking up customer 42")
        |                                                     |
        |<-- notifications/message {warning, "customer-svc",  |
        |        "DB slow, 2.3 s"} --------------------------|   [5] another log
        |        |-> serviceLogging(WARNING, ...)
        |                                                     |
        |<-- tools/call result { customer data } -------------|   [6] final tool result
        |                                                     |
   (result goes back to the LLM -> final answer -> user)
```

Two different things travel from server to client:
- **Logs** (`notifications/message`) = "what is happening".
- **Tool result** = "the actual answer for the LLM".

Logs **do not go to the LLM** by default. They go only to your `@McpLogging` handler.

---

## 8. How the server side sends a log (Spring AI example)

You asked "how is the server sending the log". On a Spring AI MCP server, a tool method can receive an `McpSyncServerExchange` and call `loggingNotification(...)`:

```java
@Component
public class CustomerTools {

    @McpTool(name = "fetch-customer", description = "Fetch a customer by id")
    public String fetchCustomer(McpSyncServerExchange exchange,
                                @McpToolParam(description = "Customer id") long id) {

        exchange.loggingNotification(LoggingMessageNotification.builder()
                .level(McpSchema.LoggingLevel.INFO)
                .logger("customer-svc")
                .data("Looking up customer " + id)
                .build());

        // ... do the real work ...

        if (slow) {
            exchange.loggingNotification(LoggingMessageNotification.builder()
                    .level(McpSchema.LoggingLevel.WARNING)
                    .logger("customer-svc")
                    .data("DB slow: 2.3 s")
                    .build());
        }
        return "Customer " + id + ": ...";
    }
}
```

Notes:
- The exchange parameter is injected by the framework and is **not** part of the tool's JSON schema (the LLM never sees it).
- The Spring AI docs now also describe a unified *request context* object that replaces direct use of the exchange in newer versions (the exchange type is marked deprecated in the annotations project). The idea is the same; check the *Special Parameters* page of your Spring AI version for the exact helper names.
- A plain `System.out.println` or normal `log.info()` on the server does **not** reach the client. Only `loggingNotification(...)` creates an MCP log message.
- For STDIO servers, never print random text to standard output — stdout is the protocol channel. That is exactly why MCP has its own log notification.

---

## 9. Configuring your client (yml) — the `clients` name must match

In your annotation you wrote `clients = "eazybytes"`. That string must be **the connection name** in `application.yml`:

```yaml
spring:
  ai:
    mcp:
      client:
        type: SYNC                    # your handler is a normal (sync) method
        annotation-scanner:
          enabled: true               # default, shown for clarity
        streamable-http:
          connections:
            eazybytes:                # <-- THIS name = clients = "eazybytes"
              url: http://localhost:8081
              endpoint: /mcp
```

Same idea for SSE or STDIO:

```yaml
        sse:
          connections:
            eazybytes:
              url: http://localhost:8081
```
```yaml
        stdio:
          connections:
            eazybytes:
              command: java
              args:
                - -jar
                - /path/to/server.jar
```

If the name doesn't match (`eazybytes` vs `eazybyte`, case differences...), the handler is simply **never called** and you get no error. This is the #1 reason for "no logs".

Sync vs async:
- `type: SYNC` → use a normal `void` method like yours.
- `type: ASYNC` → use reactive methods (return `Mono<Void>`). The docs note that a SYNC client registers only synchronous annotated methods and an ASYNC client only asynchronous ones.

---

## 10. Controlling how much you receive (`logging/setLevel`)

Without a filter, a chatty server can flood you with `debug` logs. The client can ask the server for a **minimum level**:

```json
{ "jsonrpc": "2.0", "id": 1, "method": "logging/setLevel", "params": { "level": "info" } }
```

The server replies with an empty result. After that it should send only `info` and above.

In the Java MCP SDK the client object has a method for it (check the exact name in your SDK version; it is `setLoggingLevel(...)` in the versions I know):

```java
@Component
public class McpLogLevelInitializer {

    public McpLogLevelInitializer(List<McpSyncClient> clients) {
        clients.forEach(c -> c.setLoggingLevel(McpSchema.LoggingLevel.WARNING));
    }
}
```

The Spring AI client docs also say that clients can control logging verbosity by setting minimum log levels.

Typical choice:
- **Development:** `debug` or `info`
- **Production:** `warning` (or `info` for important servers)

Errors from the spec: invalid level → JSON-RPC error `-32602`.

---

## 11. Three ways to write the handler

**A. Whole notification object** (best when `data` can be structured JSON):
```java
@McpLogging(clients = "eazybytes")
public void handle(McpSchema.LoggingMessageNotification n) {
    // n.level(), n.logger(), n.data()
}
```

**B. Individual parameters** (what you use): `(LoggingLevel level, String logger, String data)`.
```java
@McpLogging(clients = "eazybytes")
public void handle(McpSchema.LoggingLevel level, String logger, String data) { ... }
```

**C. Async (when `type: ASYNC`)**: same signatures but returning `Mono<Void>`.

Which one?
- The spec says `data` can be **any JSON value** (like the `{ "error": ..., "details": {...} }` example). With form B, `data` is a `String`. If your server sends structured objects, test what arrives, or use form A and convert `n.data()` yourself (for example to JSON with Jackson).
- `logger` can be `null`. Handle that.

---

## 12. Improving your handler (production-quality version)

Problems with the current code:

1. Every server log becomes **INFO** (an `ERROR` from the server will not trigger your error alerts).
2. You don't know **which server** it came from (you know it here, but a hard-coded name is not scalable with many servers).
3. No protection against `null`.
4. String concatenation each time.

Better version:

```java
@Component
public class McpLoggingLogic {

    private static final Logger log = LoggerFactory.getLogger("mcp.server.eazybytes");

    @McpLogging(clients = "eazybytes")
    public void serviceLogging(McpSchema.LoggingLevel level, String source, String data) {

        String origin = (source == null || source.isBlank()) ? "unknown" : source;

        // optional: attach context so you can search logs later
        MDC.put("mcpServer", "eazybytes");
        MDC.put("mcpLogger", origin);
        try {
            switch (level) {
                case DEBUG                           -> log.debug("[{}] {}", origin, data);
                case INFO, NOTICE                    -> log.info("[{}] {}", origin, data);
                case WARNING                         -> log.warn("[{}] {}", origin, data);
                case ERROR, CRITICAL, ALERT, EMERGENCY -> log.error("[{}] {}", origin, data);
            }
        } finally {
            MDC.remove("mcpServer");
            MDC.remove("mcpLogger");
        }
    }
}
```

What this gives you:
- Server `WARNING` → your `WARN`, server `ERROR/CRITICAL/ALERT/EMERGENCY` → your `ERROR`. Your normal alerting rules now work.
- A dedicated logger name (`mcp.server.eazybytes`), so you can turn it up or down in `application.yml`:
  ```yaml
  logging:
    level:
      mcp.server.eazybytes: INFO
  ```
- MDC fields so JSON logging tools can filter by server.
- Use `{}` placeholders instead of string concatenation.

One handler for many servers (an annotation takes several names):
```java
@McpLogging(clients = {"eazybytes", "billing", "inventory"})
```
*(Check that your Spring AI version accepts several names in `clients`, then, if you need the server name inside the method, use one small handler class per server that delegates to a shared helper.)*

---

## 13. What to use in production (important)

Short honest answer: **MCP logging is only the delivery pipe from server to client.** In production you still use your normal logging and monitoring stack. A good setup looks like this:

```
 MCP Server  --(notifications/message)-->  @McpLogging handler (your client app)
                                                     |
                                                     v
                                  SLF4J / Logback (JSON format, MDC fields)
                                                     |
                                                     v
                         Log collector (Filebeat / Fluent Bit / OpenTelemetry Collector)
                                                     |
                                                     v
                     ELK / Grafana Loki / Datadog / Splunk / CloudWatch  (search + alerts)
```

Production checklist:

| # | What | Why |
|---|---|---|
| 1 | **Map MCP levels to SLF4J levels** (section 12) | So alerts on `ERROR` work |
| 2 | **Set a minimum level** with `logging/setLevel` (`warning` or `info`) | Avoid noise, cost and load |
| 3 | **JSON logging** (e.g. Logback with a JSON encoder, or Spring Boot structured logging) | Easy to search in ELK / Loki / Datadog |
| 4 | **Add context with MDC**: server name, logger, and your own request / conversation ID | The MCP message has no request ID, so you can't link a server log to a user request otherwise |
| 5 | **Never block inside the handler** | Handler runs on the client's MCP thread; slow code (DB writes, HTTP calls) delays other messages. For heavy work push to an async queue / executor |
| 6 | **Rate limit / sample noisy logs** | The spec says servers *should* rate limit, but you should protect yourself too |
| 7 | **Redact secrets** | See section 14 |
| 8 | **Alert only on high levels** (`ERROR` and up) | `critical/alert/emergency` should page someone |
| 9 | **Keep server-side logs too** | The server's own files/ELK remain the source of truth; MCP logs are a convenience copy |
| 10 | **Add metrics and tracing separately** | Use Micrometer / OpenTelemetry (Spring AI has an observability module) for counts, latency, traces. Logs alone are not enough |

Related handlers worth adding together with logging (same `clients` rule):
- `@McpProgress(clients = "eazybytes")` — progress of long operations.
- `@McpToolListChanged(clients = "eazybytes")` — the server changed its tool list (useful with the Tool Search advisor, because the tool index is built per session).

---

## 14. Security rules for logs

The MCP specification says log messages **MUST NOT** contain:

- credentials or secrets,
- personal identifying information,
- internal system details that could help an attacker.

And implementations **SHOULD**: rate limit messages, validate all data fields, control log access, and monitor for sensitive content.

On the **client** side this means:
- Treat server logs as **untrusted input**. A third-party MCP server could send misleading or huge text.
- **Do not execute** or blindly pass log text into prompts to the LLM. (A malicious server could try prompt injection through log text.)
- Truncate very long messages before saving.
- Control who can read these logs in your log platform.

---

## 15. Troubleshooting: "I don't see any logs"

| Check | Detail |
|---|---|
| **Name mismatch** | `clients = "eazybytes"` must equal the connection name under `streamable-http` / `sse` / `stdio` in the yml |
| **Annotation scanner** | `spring.ai.mcp.client.annotation-scanner.enabled` must not be `false` |
| **Is it a bean?** | The class needs `@Component` (you have it) and must be inside your component scan package |
| **Server never sends logs** | The server must declare the `logging` capability and actually call `loggingNotification(...)` |
| **Minimum level too high** | If you set `error`, you will never see `info` |
| **Stateless server** | Stateless mode has no full exchange, so logging notifications are not expected |
| **SYNC / ASYNC mismatch** | A SYNC client ignores async (Mono) handler methods and the other way round |
| **Your own logger level** | If `McpLoggingLogic` logger is set to `WARN`, your `logger.info` lines are hidden |
| **Wrong parameter order** | Must be `(LoggingLevel, String logger, String data)` |
| **Import** | `org.springframework.ai.mcp.annotation.McpLogging` (as you have) |

Quick test: temporarily change the handler to the whole-notification form and `System.out.println(n)` to see exactly what arrives.

---

## 16. Quick summary

- **What:** MCP logging = the server sends `notifications/message` (level + optional logger + data) to the client.
- **Why:** you can see what happens inside a remote server without opening its log files.
- **Who handles it in Spring AI:** a bean method with `@McpLogging(clients = "<connection name>")`.
- **Your method:** receives `(LoggingLevel level, String logger, String data)`; it logs everything at INFO (works, but loses severity).
- **When does the server send:** whenever the server author calls `loggingNotification(...)`, typically during tool calls; only if it declared the `logging` capability; filtered by the minimum level set with `logging/setLevel`; not in stateless mode.
- **Production:** map levels properly, set a minimum level, JSON logs + MDC, send to ELK/Loki/Datadog, never block in the handler, redact secrets, add metrics/tracing with Micrometer/OpenTelemetry, keep server-side logs too.

---

## Sources

- MCP specification — Logging: https://modelcontextprotocol.io/specification/2025-11-25/server/utilities/logging.md
- Spring AI reference — MCP Client Annotations: https://docs.spring.io/spring-ai/reference/api/mcp/mcp-annotations-client.html
- Spring AI reference — MCP Client Boot Starter: https://docs.spring.io/spring-ai/reference/api/mcp/mcp-client-boot-starter-docs.html
- Spring AI reference — MCP Annotations Special Parameters: https://docs.spring.io/spring-ai/reference/api/mcp/mcp-annotations-special-params.html
- MCP annotations project (server-side `loggingNotification` example): https://github.com/spring-ai-community/mcp-annotations
