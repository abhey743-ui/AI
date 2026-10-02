# MCP Sampling — Simple Step-by-Step Guide (with your HelpDesk code)

This guide uses **only the sampling parts** of your two files. I removed `createTicket`, `getTicketStatus`, progress and `ctx.info` logging, because they are not related to sampling.

| Part | What you learn |
|---|---|
| 1 | What sampling is, in one story |
| 2 | Why we use it |
| 3 | Real-world examples (where it is used) |
| 4 | Your trimmed code |
| 5 | Server code, step by step |
| 6 | Client code, step by step |
| 7 | The full request flow, step by step (the most important part) |
| 8 | What you will see when you run it |
| 9 | Limits of your current code, and safe improvements |
| 10 | Troubleshooting + cheat sheet |

---

## 1. What is sampling? (one story)

Normally in MCP the **client is the boss** and the **server is a helper**:

```
Client (has the LLM / the brain)  ──asks──►  Server (has tools, database, files)
```

The client asks the server: "run this tool", and the server answers.

But sometimes the server, in the middle of its work, needs a **brain**. For example your tool has loaded 8 tickets from the database and now wants to write a friendly summary. Code can't write that. An LLM can.

The server has **no LLM and no API key**. So it does something unusual — it asks the **client** back:

> "Hey client, you have an LLM. Please run this prompt for me and send me the answer."

That is **sampling**: *the server borrows the client's LLM.*

```
normal:    Client ───── "run tool" ─────► Server
sampling:  Client ◄──── "run this prompt on your LLM" ───── Server
```

Think of a **plumber (server)** at your home (client). The plumber has tools, but needs a very expensive special machine. He doesn't carry his own — he asks: "Can I use yours?" You decide: yes/no, which machine, how long. The plumber gets the result and continues his job.

The word "sampling" just means "generating text from an LLM" (the LLM *samples* the next words).

---

## 2. Why do we use it?

| Reason | Explanation |
|---|---|
| **No API key on the server** | The server never stores an OpenAI/Claude/Gemini key. Only the client has it. Fewer secrets to protect |
| **The client controls cost and model** | The client chooses the model, limits tokens, and pays the bill. The server only *suggests* preferences |
| **The server stays model-independent** | Your server works with any client: today GPT, tomorrow Claude, or a local Ollama model |
| **Privacy / control** | Data goes to the LLM chosen by the client owner (the company that owns the app), not to a provider chosen by some third-party server |
| **Smart tools** | A tool can combine real data (database, API) with LLM reasoning *inside one tool call* |
| **Human in the loop** | The spec says the client **should** let a human review or reject sampling requests |

---

## 3. Real-world examples: where sampling is used

In all examples the **server** knows the data, but the **LLM** is needed to understand or write text.

| Server (MCP tool) | What the code does | Where the LLM is needed (sampling) |
|---|---|---|
| **Help desk (yours)** `summarizeTickets` | Loads a customer's tickets from the DB | Write a friendly summary grouped by status |
| **Log analyzer** `explainErrors` | Reads the last 500 log lines from files | "Explain the likely root cause of these errors in simple words" |
| **Document processor** `extractInvoiceData` | Reads a PDF/text file from storage | Turn messy text into structured fields (vendor, date, total) |
| **Code review server** `reviewPullRequest` | Fetches the diff from Git | "List possible bugs and risky changes in this diff" |
| **Database assistant** `describeQueryResult` | Runs a safe SQL query | Describe the result table in plain English for a manager |
| **Support triage** `classifyTicket` | Receives a new ticket text | "Classify as BILLING / TECHNICAL / OTHER and give urgency" |
| **Translation/moderation** | Gets user-generated text | Translate or check if the text is abusive |

**When NOT to use sampling**
- The task is plain logic (sorting, counting, formatting): use normal Java.
- The server must use a specific model for sure (clients can ignore hints).
- Very high volume or strict latency: each sample is a full LLM call (money + seconds).
- The connection is **stateless** — stateless servers can't send requests back to the client, so no sampling.

---

## 4. Your trimmed code

### 4.1 Server (only the sampling tool)

```java
package com.eazybytes.mcpserverremote.tool;

import io.modelcontextprotocol.spec.McpSchema;
import org.springframework.ai.mcp.annotation.McpTool;
import org.springframework.ai.mcp.annotation.McpToolParam;
import org.springframework.ai.mcp.annotation.context.McpSyncRequestContext;
// + your usual: Component, RequiredArgsConstructor, slf4j, List, Collectors, entity, service

@Component
@RequiredArgsConstructor
public class HelpDeskTools {

    private static final Logger LOGGER = LoggerFactory.getLogger(HelpDeskTools.class);

    private final HelpDeskTicketService service;

    @McpTool(name = "summarizeTickets", description = "Generate a friendly, natural-language summary of all the " +
            "support tickets that belong to a given username")
    String summarizeTickets(@McpToolParam(description = "Username to summarize the help desk tickets for")
                            String username, McpSyncRequestContext ctx) {

        List<HelpDeskTicket> tickets = service.getTicketsByUsername(username);          // (A)

        if (tickets.isEmpty()) {                                                        // (B)
            return "No support tickets were found for user " + username + ".";
        }

        if (!ctx.sampleEnabled()) {                                                     // (C)
            return tickets.toString();
        }

        String ticketData = tickets.stream()                                            // (D)
                .map(t -> "Ticket #" + t.getId() + " | Issue: " + t.getIssue()
                        + " | Status: " + t.getStatus() + " | ETA: " + t.getEta())
                .collect(Collectors.joining("\n"));

        String systemPrompt = """
                You are a friendly help desk assistant. Using ONLY the ticket data provided by the user,
                write a short, warm summary for the customer about the status of their support tickets.
                Mention how many tickets they have in total, group them by status (OPEN, IN_PROGRESS, CLOSED),
                and reassure them about the ones that are still being worked on. Keep it under 120 words and
                do not invent any information that is not present in the ticket data.
                """;                                                                    // (E)

        McpSchema.CreateMessageResult result = ctx.sample(spec -> spec                  // (F)
                .systemPrompt(systemPrompt)
                .message("Here are the support tickets for " + username + ":\n" + ticketData));

        return ((McpSchema.TextContent) result.content()).text();                       // (G)
    }
}
```

### 4.2 Client (the sampling handler)

```java
package com.eazybytes.mcpclient.util;

import io.modelcontextprotocol.spec.McpSchema;
import org.springframework.ai.chat.messages.*;
import org.springframework.ai.chat.model.ChatModel;
import org.springframework.ai.chat.model.ChatResponse;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.mcp.annotation.McpSampling;
// + Component, slf4j, ArrayList, List, Objects, Collectors

@Component
public class HelpDeskSamplingProvider {

    private final ChatModel chatModel;

    public HelpDeskSamplingProvider(ChatModel chatModel) {
        this.chatModel = chatModel;
    }

    @McpSampling(clients = "eazybytes")
    public McpSchema.CreateMessageResult handleSamplingRequest(McpSchema.CreateMessageRequest request) {

        List<Message> messages = new ArrayList<>();                                     // (1)
        if (request.systemPrompt() != null && !request.systemPrompt().isBlank()) {
            messages.add(new SystemMessage(request.systemPrompt()));
        }
        String userText = request.messages().stream()                                   // (2)
                .filter(m -> m.content() instanceof McpSchema.TextContent
                        && m.role().name().equalsIgnoreCase(McpSchema.Role.USER.name()))
                .map(m -> ((McpSchema.TextContent) m.content()).text())
                .collect(Collectors.joining("\n"));
        messages.add(new UserMessage(userText));

        ChatResponse response = chatModel.call(new Prompt(messages));                   // (3)
        if (response.getResult() == null) {
            throw new IllegalStateException("LLM returned no result for the MCP sampling request");
        }
        String generatedText = Objects.requireNonNullElse(response.getResult().getOutput().getText(), "");
        String model = response.getMetadata().getModel();                               // (4)

        return McpSchema.CreateMessageResult.builder(McpSchema.Role.ASSISTANT, generatedText, model)
                .build();                                                               // (5)
    }
}
```

### 4.3 Config (client)

```yaml
spring:
  ai:
    mcp:
      client:
        type: SYNC
        request-timeout: 60s         # sampling waits for an LLM; the default 20s can be too short
        streamable-http:
          connections:
            eazybytes:               # this name = clients = "eazybytes"
              url: http://localhost:8081
```

---

## 5. Server code, step by step

The tool is `summarizeTickets`. The LLM of the **client app** decided to call it (for example, the user asked "summarize john's tickets"). Now the server code runs:

**(A) Load the data.** Normal database work. This is the part only the server can do.

**(B) Empty case.** No tickets → return a plain text. **No LLM is used**, so no cost. Always avoid calling an LLM when code can answer.

**(C) `ctx.sampleEnabled()`** — "Does the connected client support sampling?" During connection setup, the client announces what it can do (its *capabilities*). If the client did not announce `sampling`, calling `ctx.sample` would fail. So we check first and fall back to raw data. This is the graceful fallback.

**(D) Build the data text.** Convert tickets to simple lines like:
```
Ticket #12 | Issue: Cannot log in | Status: OPEN | ETA: 2026-10-05
```
This is the content the LLM will summarize.

**(E) The system prompt.** The instruction for the LLM: tone, grouping, max 120 words, "do not invent anything". The server controls the **instructions**; this is the "prompt engineering" of the tool.

**(F) `ctx.sample(...)`** — the key line.
- It sends a `sampling/createMessage` request to the client, through the **same connection** already open.
- The server thread **waits here** (blocking, since this is `McpSyncRequestContext`) until the client answers.
- `.systemPrompt(...)` = the instruction. `.message(...)` = the user message with the ticket data.
- Optional: you can also pass `modelPreferences` (see section 9) to suggest a cheaper/faster/smarter model.

**(G) Read the answer.** `result.content()` is the generated content. For text it is `TextContent`, and `.text()` is the summary string. This string becomes the **tool result** that goes back to the client app.

---

## 6. Client code, step by step

`@McpSampling(clients = "eazybytes")` means: *"when the server called eazybytes asks for a sample, call this method."* (The name must match the yml connection name.) Because this handler exists, Spring AI's MCP client advertises the sampling capability for you — that is why `ctx.sampleEnabled()` returns `true` on the server.

**Input:** `CreateMessageRequest request` — the server's request (system prompt, messages, and optional preferences like model hints and max tokens).

**(1) Start the Spring AI message list.** If the server sent a system prompt, add it as a `SystemMessage`.

**(2) Convert the MCP messages to text.** MCP messages have a `role` (user/assistant) and `content` (text/image/audio). This code keeps only **text messages from the user role**, and joins them into one string. Then it creates a `UserMessage`.

**(3) Call the LLM:** `chatModel.call(new Prompt(messages))`. This is a plain LLM call with no tools.

> **Why `ChatModel` and not `ChatClient`? (the comment in your code)**
> Your `ChatClient` is configured with the MCP tools. If you used it here, the LLM could decide to call `summarizeTickets` again → that tool asks for sampling again → the client calls the LLM again → … an **infinite loop**.
> `ChatModel` has no tools registered, so the LLM can only answer with text. The loop is impossible.

**(4)** Read the generated text and the **model name** actually used (for example `gpt-4o-mini`). The model name goes back to the server so it knows which model answered.

**(5) Build the result:** `CreateMessageResult` with role `ASSISTANT`, the text, and the model name. This is sent back to the server.

---

## 7. The full request flow, step by step

This is the whole journey. **There are two different LLM calls** — don't mix them:

- **LLM call #1:** the normal chat. The LLM decides *which tool* to call.
- **LLM call #2:** the sampling call. The LLM writes the *summary*.

Both use the **same ChatModel / provider** in your app, but they are separate calls with separate prompts.

### 7.1 The diagram

```
User      Client app (ChatClient)       LLM          MCP Client            MCP Server (summarizeTickets)
 |               |                       |               |                          |
 | "Summarize    |                       |               |                          |
 |  john's       |                       |               |                          |
 |  tickets" --->|                       |               |                          |
 |               |-- (1) prompt + tool list -->|          |                          |
 |               |<-- (2) LLM #1: "call summarizeTickets(john)" --|                  |
 |               |-- (3) execute tool ----------------->|  |                          |
 |               |                       |               |-- (4) tools/call ------->|
 |               |                       |               |                          |-- (5) load tickets from DB
 |               |                       |               |                          |-- (6) sampleEnabled()? yes
 |               |                       |               |                          |-- (7) build prompt
 |               |                       |               |<-- (8) sampling/createMessage --|
 |               |                       |               |    [server tool is WAITING]     |
 |               |   (9) @McpSampling handler runs      |                          |
 |               |<-- ChatModel.call(prompt) ------------|                          |
 |               |-- LLM #2: writes the summary ------->|                          |
 |               |<-- summary text ----------------------|                          |
 |               |                       |               |-- (10) sampling result ->|
 |               |                       |               |                          |-- (11) tool returns summary
 |               |                       |               |<-- (12) tools/call result ------|
 |               |<-- (13) tool result ---|               |                          |
 |               |-- (14) LLM #1 again: tool result in context -->|                  |
 |               |<-- (15) final answer text ------------|               |                          |
 |<-- answer ----|                       |               |                          |
```

### 7.2 The same flow in words (track it step by step)

| # | Who | What happens |
|---|---|---|
| 1 | Client app | `ChatClient` sends the user question + the list of MCP tools to the LLM |
| 2 | LLM #1 | Decides: "I need `summarizeTickets` with `username = john`" |
| 3 | Client app | Spring AI's tool loop runs that tool through the MCP client |
| 4 | MCP client → server | Sends the JSON-RPC request `tools/call` |
| 5 | Server tool | Loads john's tickets from the database (section 5, step A) |
| 6 | Server tool | `ctx.sampleEnabled()` — client announced sampling at connection time, so `true` |
| 7 | Server tool | Builds the system prompt and the ticket text |
| 8 | Server → client | `ctx.sample(...)` sends `sampling/createMessage` **back over the same connection**. The server tool now **waits** |
| 9 | Client | Spring AI finds the `@McpSampling` method for `eazybytes` and calls it |
| 9b | Client | The handler calls `ChatModel` → **LLM call #2** → summary text |
| 10 | Client → server | Returns `CreateMessageResult` (assistant, text, model) |
| 11 | Server tool | Wakes up, reads the text, returns it as the tool result |
| 12 | Server → client | Sends the `tools/call` result |
| 13 | Client app | The tool result is added to the conversation |
| 14 | LLM #1 | Called again, now with the summary in the context |
| 15 | LLM #1 | Writes the final answer for the user |

Two things to notice:
1. **Nesting.** Step 8–10 happens *inside* step 4–12. The tool call is still open while the sampling request travels back. The client's `request-timeout` covers the **whole** tool call, including the sampling LLM call, so set it high enough.
2. **Direction.** Steps 4 and 12 go client → server and back (normal). Steps 8 and 10 are server → client and back (the reverse request).

### 7.3 The real messages on the wire

**Step 8 — what your server sends (`sampling/createMessage`):**

```json
{
  "jsonrpc": "2.0",
  "id": 7,
  "method": "sampling/createMessage",
  "params": {
    "systemPrompt": "You are a friendly help desk assistant. Using ONLY the ticket data ...",
    "messages": [
      {
        "role": "user",
        "content": {
          "type": "text",
          "text": "Here are the support tickets for john:\nTicket #12 | Issue: Cannot log in | Status: OPEN | ETA: ...\nTicket #15 | ..."
        }
      }
    ]
  }
}
```

**Step 10 — what your client returns:**

```json
{
  "jsonrpc": "2.0",
  "id": 7,
  "result": {
    "role": "assistant",
    "content": {
      "type": "text",
      "text": "Hi John! You have 3 tickets in total. One is still OPEN ..."
    },
    "model": "gpt-4o-mini"
  }
}
```

Matching the code: `request.systemPrompt()` and `request.messages()` in the client are these exact fields; `CreateMessageResult.builder(ASSISTANT, text, model)` creates the `result`.

---

## 8. What you will see when you run it

Ask the chat: *"Summarize the tickets for john"*.

**Server log (roughly):**
```
Generating ticket summary for user: john
Requesting LLM completion from the MCP client via sampling...
Sampling response received. Model used by client: gpt-4o-mini
```

**Client log:**
```
Received MCP sampling request from server. System prompt: You are a friendly help desk assistant...
LLM produced sampling response using model 'gpt-4o-mini': Hi John! You have 3 tickets ...
```

Order check: server "Requesting…" → client "Received…" → client "LLM produced…" → server "Sampling response received…". If this order appears, sampling works end to end.

---

## 9. Limits of your current code, and safe improvements

Your code works for the simple case. These are things to know:

| Topic | What your code does now | What to know / improve |
|---|---|---|
| Messages | Keeps only **user-role text** messages | Assistant messages (earlier turns), images and audio are ignored. Fine for one-shot prompts; not enough for multi-turn sampling |
| `maxTokens`, `temperature`, `stopSequences` | Ignored | The server may send them; the client decides. You can pass them to the model through Spring AI `ChatOptions` (check the accessor names of `CreateMessageRequest` in your version) |
| `modelPreferences` | Ignored | The server can suggest model hints and priorities (`costPriority`, `speedPriority`, `intelligencePriority`, each 0–1). The client may map or ignore them |
| `stopReason` | Not set | Optional in the result |
| Result cast on the server | `(TextContent) result.content()` | Throws `ClassCastException` if the client ever returns non-text. Check `instanceof` first |
| Errors | `IllegalStateException` if the LLM returns nothing | Fine; the error goes back to the server as a failed request |
| Human approval | None | The spec says clients **should** let a human review/deny sampling in sensitive apps |

### Suggesting a model from the server (optional)

The Spring AI docs show the server can pass preferences like this:

```java
ctx.sample(spec -> spec
        .systemPrompt(systemPrompt)
        .message("...")
        .modelPreferences(pref -> pref.modelHints("gpt-4")));
```

The spec calls hints *advisory*: the client makes the final model choice. Your handler currently ignores them and uses its own `ChatModel`.

### Security rules for sampling

- **The server writes the prompt.** An untrusted server could try prompt injection or ask for something expensive. Only connect to servers you trust; keep a human or a policy check for sensitive cases.
- **Data goes to your LLM provider.** Ticket data in the prompt is sent to the client's provider. Don't put secrets in sampling prompts.
- **Rate limit** sampling requests on the client (each one costs money), and consider a max token limit.
- **Never use the tool-enabled `ChatClient`** inside the handler (infinite loop; see section 6).

---

## 10. Troubleshooting + cheat sheet

| Symptom | Likely cause | Fix |
|---|---|---|
| Server always returns raw tickets | `ctx.sampleEnabled()` is `false` | Client has no `@McpSampling` for that connection, or `clients` name ≠ yml connection name |
| Handler never called | Name mismatch | `@McpSampling(clients = "eazybytes")` must equal the yml connection name |
| Timeout during the tool call | `request-timeout` (default 20s) is shorter than tool + LLM time | Raise it (e.g. 60s) |
| Infinite loop / repeated calls | Handler uses a `ChatClient` with MCP tools | Use `ChatModel` |
| Sampling never works | Server is **stateless** | Use a stateful transport (stdio, SSE, stateful streamable HTTP) |
| `ClassCastException` on server | Result content is not text | Check `instanceof TextContent` |
| Summary ignores data | System prompt too weak / data missing | Print the prompt on the server; keep "use ONLY the data provided" |

**Cheat sheet**

- **Sampling** = the server asks the client to run an LLM prompt (`sampling/createMessage`), and the client answers with the generated text.
- **Server side:** `ctx.sampleEnabled()` → build the prompt → `ctx.sample(...)` (waits) → read `result.content()`.
- **Client side:** `@McpSampling(clients = "...")` method → convert MCP messages to a Spring AI `Prompt` → `ChatModel.call(...)` → return `CreateMessageResult`.
- **Use `ChatModel`, not `ChatClient`**, in the handler.
- **Two LLM calls** in one user question: #1 chooses the tool, #2 (sampling) writes the text.
- **Client is in control:** model, cost, limits, and human approval.

---

## Sources

- MCP specification — Sampling: https://modelcontextprotocol.io/specification/2025-06-18/client/sampling.md
- Spring AI — MCP Client Annotations (`@McpSampling`): https://docs.spring.io/spring-ai/reference/api/mcp/mcp-annotations-client.html
- Spring AI — MCP Annotations Special Parameters (`McpSyncRequestContext`, `sampleEnabled`, `sample`): https://docs.spring.io/spring-ai/reference/api/mcp/mcp-annotations-special-params.html
