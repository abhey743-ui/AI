# Spring AI Message Roles — The Complete Guide

## 1. Why roles exist at all

When you send a prompt to an LLM, you're not sending one blob of text — you're sending a **list of messages**, and each message is tagged with a **role**. The role tells the model (and Spring AI) *who said this* and *what job it's doing* in the conversation.

Without roles, the model can't tell the difference between:
- "You are a strict Java tutor, never write code for the student" (an instruction)
- "Write the code for me" (the actual user asking something)

If both were just plain text with no role, the model has no reliable way to know which one is a rule it must obey and which one is a request it should evaluate. Roles give the model **structure and hierarchy**, which makes responses more predictable, more secure, and easier to reason about.

Spring AI represents this with a single enum:

```java
package org.springframework.ai.chat.messages;

public enum MessageType {
    USER("user"),
    ASSISTANT("assistant"),
    SYSTEM("system"),
    TOOL("tool");
}
```

Each `Message` implementation maps to one of these:

| MessageType | Class | Who "speaks" it |
|---|---|---|
| `SYSTEM` | `SystemMessage` | The developer, before the conversation starts |
| `USER` | `UserMessage` | The end user (or the app on the user's behalf) |
| `ASSISTANT` | `AssistantMessage` | The model's own previous reply |
| `TOOL` | `ToolResponseMessage` | The result of a function/tool call, fed back to the model |

---

## 2. The four roles, explained

### 2.1 System Role — `SystemMessage`

**What it does:** Sets the model's persona, ground rules, tone, output format, and constraints — instructions given *before* the actual conversation begins.

**Why we use it:** It separates "how you should behave" from "what the user is asking." This keeps behavior stable across many different user inputs and avoids mixing instructions with untrusted user text.

```java
String response = chatClient.prompt()
        .system("You are a senior Java architect. Answer only Java questions. Be concise.")
        .user("How does garbage collection work?")
        .call()
        .content();
```

**Important caveat (Spring AI docs are explicit about this):** the system message is a *behavioral nudge*, not a security boundary. A model can still be jailbroken or tricked into ignoring it via prompt injection, so never put secrets or authorization logic inside a system message — enforce that in your application code instead.

### 2.2 User Role — `UserMessage`

**What it does:** Carries the actual question, command, or input — from a human, or from your app acting on the human's behalf. This is the "payload" the model is supposed to respond to. It can also carry multimodal content (images, audio, documents) via the `Media` field.

```java
UserMessage userMessage = new UserMessage("Summarize this document.");
```

### 2.3 Assistant Role — `AssistantMessage`

**What it does:** Represents the model's *own prior response* in a multi-turn conversation. When you resend history to the model (for memory/context), each earlier AI answer goes back in as an `AssistantMessage` so the model "remembers" what it already said.

It can also carry **tool call requests** — when the model decides it wants to call a function instead of answering directly, that request is embedded in an `AssistantMessage`.

```java
List<Message> history = List.of(
    new UserMessage("What is a Java record?"),
    new AssistantMessage("A record is an immutable data carrier class..."),
    new UserMessage("Give me an example")
);
```

### 2.4 Tool Role — `ToolResponseMessage`

**What it does:** Carries the *result* of executing a tool/function that the model previously requested (via an `AssistantMessage` tool call). This closes the loop: model asks for a tool → your code runs it → you feed the result back tagged as `TOOL` → model uses it to produce a final answer.

```java
// Simplified flow:
// 1. Model replies with an AssistantMessage containing a tool call request
// 2. Spring AI (or you) execute the actual Java method / API call
// 3. Result is wrapped in a ToolResponseMessage and sent back
// 4. Model reads it and produces the final natural-language answer
```

---

## 3. Which provider supports which role — the real picture

Here's the catch: Spring AI's four-role model (`SYSTEM`, `USER`, `ASSISTANT`, `TOOL`) is a **common abstraction**. The underlying providers do **not** all support the same roles in the same way at the wire-protocol level.

| Provider | Roles it natively supports (raw API) | Notes |
|---|---|---|
| **OpenAI / Azure OpenAI** | `system`, `user`, `assistant`, `tool` | Full 1:1 match. This is essentially the reference model Spring AI's enum was designed around. |
| **Anthropic (Claude)** | `user`, `assistant` only | **No `system` role message** and **no separate `tool` role** in the messages array. System instructions go in a distinct top-level `system` parameter, outside the messages list. Tool results are sent back as `user` messages containing a special `tool_result` content block. |
| **Google Gemini / Vertex AI** | `user`, `model` | Gemini doesn't even call it "assistant" — it's `model`. System instructions go in a separate `systemInstruction` field, not in the message list at all. Tool responses are sent as a `user` message with a `functionResponse` part. |
| **Ollama (local models)** | `system`, `user`, `assistant`, `tool` | Mirrors OpenAI's shape closely (see `OllamaApi.Message.Role`), though actual behavior depends on whether the specific local model was fine-tuned to respect a system prompt. |
| **Mistral AI** | `system`, `user`, `assistant`, `tool` | OpenAI-compatible shape. |
| **Amazon Bedrock** (Claude, Titan, Llama, etc., via Bedrock Converse API) | Varies by underlying model | Bedrock's Converse API mostly normalizes to `user`/`assistant` plus a separate system field, similar to Anthropic's approach, because many Bedrock-hosted models (including Claude) don't have a native `system` role. |
| **Hugging Face-hosted / other open models** | Often just `user` (no roles at all) | Many raw open-weight chat templates don't distinguish roles beyond a template string; Spring AI has to synthesize structure. |

**The pattern:** OpenAI's shape (4 roles, all in one flat list) became the "lowest common denominator" API design, but Anthropic and Google deliberately pulled the system instruction *out* of the message list into its own field, and don't have a genuine `tool`/`function` role — they fold tool results back in as `user` turns with special content blocks.

---

## 4. How Spring AI handles a role the provider doesn't support

This is the actual point of Spring AI's abstraction: **you always write your code against `SystemMessage` / `UserMessage` / `AssistantMessage` / `ToolResponseMessage`, and each provider's `ChatModel` implementation silently translates that into whatever shape the real API needs.** You don't have to special-case Anthropic vs. OpenAI in your business logic.

Concretely, per role:

### System role, when the provider has no `system` message type
- **Anthropic / Bedrock-Claude:** Spring AI's `AnthropicChatModel` pulls every `SystemMessage` out of the message list and concatenates their text into the request's dedicated top-level `system` field. The messages array sent over the wire only ever contains `user`/`assistant` entries — a raw `system`-role message would actually be *rejected* by Anthropic's API with an invalid-role-sequence error, so Spring AI must strip it out before the call ever goes out.
- **Gemini / Vertex:** similarly hoisted into the `systemInstruction` request field rather than left in the conversation list.
- **Models with no system-prompt concept at all:** Spring AI's general fallback (per the official docs) is that for *"AI models that do not use specific roles, the `UserMessage` implementation acts as a standard category"* — meaning if a model genuinely has nothing resembling a system slot, the framework degrades gracefully by treating that content as a leading user message rather than failing outright.

### Tool role, when the provider has no dedicated `tool` role
- **Anthropic:** there's no `tool` role in the wire format. Spring AI's Anthropic client converts a `ToolResponseMessage` into a `user` message whose content is a `tool_result` content block referencing the original tool-call ID — same information, different envelope.
- **Gemini:** same idea — the tool result becomes a `user` message carrying a `functionResponse` part instead of a bare "tool" role.
- Your Java code never notices this. You still just build a `ToolResponseMessage` and hand it to `ChatClient`/`ChatModel`; the provider-specific `ChatModel` bean is what performs the translation before serializing the HTTP request.

### Assistant role, when the provider names it differently
- **Gemini** literally calls this role `model`, not `assistant`. Spring AI's Vertex/Gemini integration maps `AssistantMessage` → `model` under the hood. Application code still just uses `AssistantMessage`.

### General principle
Spring AI's `ChatModel` abstraction (`OpenAiChatModel`, `AnthropicChatModel`, `VertexAiGeminiChatModel`, `OllamaChatModel`, `MistralAiChatModel`, `BedrockConverseChatModel`, etc.) is exactly where this adaptation logic lives. Each implementation is responsible for:
1. Walking the `List<Message>` in the `Prompt`.
2. Re-bucketing/renaming/relocating messages by `MessageType` into whatever shape that vendor's REST API actually expects (a `system` param, a `model` role, a `tool_result` content block, etc.).
3. Reassembling the provider's raw response back into Spring AI's normalized `ChatResponse` / `AssistantMessage` types.

So the practical takeaway: **you write portable code once, using the 4 Spring AI roles, and swapping the `ChatModel`/`ChatClient` implementation (OpenAI → Anthropic → Gemini → Ollama, etc.) doesn't require you to rewrite your prompt-building logic.** The adaptation for "this provider doesn't have that role" is Spring AI's job, not yours — with the one soft caveat that if a role is *emulated* (like system-as-first-user-message on a truly role-less model), the effectiveness of that instruction depends on how well that specific model was trained to respect instructions in that position.

---

## 5. Quick reference cheat sheet

```
SystemMessage    -> "how to behave"      -> may be hoisted to a separate field, or become a leading user message
UserMessage      -> "what's being asked" -> universally supported everywhere
AssistantMessage -> "what the AI said"   -> may be renamed (e.g. Gemini's "model")
ToolResponseMessage -> "tool result"     -> may be folded into a user message with a special content block
```

If you remember one sentence: **Spring AI's roles are a stable contract for your code; the provider-specific `ChatModel` is the translator that makes that contract work no matter how quirky the underlying vendor's API is.**
