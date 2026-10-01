# Spring AI – Tool Calling (Complete Guide)

> Goal of this file: after reading it, you understand **what** tool calling is, **why** we need it,
> **how it works inside Spring AI** (classes, interfaces, data flow, loop), and you can **write your own tools**
> with custom parameters.
>
> Version note: this guide is written for Spring AI **1.0.x / 1.1.x** (`@Tool`, `@ToolParam`, `ToolCallback`,
> `ToolCallingManager`). Older versions (0.8 / 1.0 milestones) used `FunctionCallback` and `.functions(...)`.
> Those names are old. Always check the official docs for your exact version.

---

## Table of Contents

1. [The Problem: LLM Cannot Do Things](#1-the-problem-llm-cannot-do-things)
2. [What Is a Tool?](#2-what-is-a-tool)
3. [Why Do We Use Tools?](#3-why-do-we-use-tools)
4. [When Do We Use Tools (and When Not)?](#4-when-do-we-use-tools-and-when-not)
5. [Benefits](#5-benefits)
6. [Where Tools Are Used in Real Tech](#6-where-tools-are-used-in-real-tech)
7. [What We Can Achieve: AI App + Tools](#7-what-we-can-achieve-ai-app--tools)
8. [Big Picture: Who Does What?](#8-big-picture-who-does-what)
9. [Step-by-Step Flow (Request to Final Answer)](#9-step-by-step-flow-request-to-final-answer)
10. [The State Machine (Loop) Inside Spring AI](#10-the-state-machine-loop-inside-spring-ai)
11. [Core Classes and Interfaces](#11-core-classes-and-interfaces)
12. [How the Model Decides Which Tool to Use](#12-how-the-model-decides-which-tool-to-use)
13. [How Spring AI Runs Your Method](#13-how-spring-ai-runs-your-method)
14. [Your First Tool (Demo)](#14-your-first-tool-demo)
15. [Ways to Add Tools to a Request](#15-ways-to-add-tools-to-a-request)
16. [Tool Parameters: `@ToolParam` and Custom Request Params](#16-tool-parameters-toolparam-and-custom-request-params)
17. [Complex Parameters (Records / Objects / Lists)](#17-complex-parameters-records--objects--lists)
18. [Return Values and `returnDirect`](#18-return-values-and-returndirect)
19. [`ToolContext`: Extra Data the Model Must NOT See](#19-toolcontext-extra-data-the-model-must-not-see)
20. [Other Ways to Define Tools (Function Bean, Programmatic)](#20-other-ways-to-define-tools-function-bean-programmatic)
21. [Error Handling](#21-error-handling)
22. [Manual Tool Execution (You Control the Loop)](#22-manual-tool-execution-you-control-the-loop)
23. [Full Mini Project: Appointment Assistant](#23-full-mini-project-appointment-assistant)
24. [Tools + Other Spring AI Parts (Advisors, Memory, RAG, MCP)](#24-tools--other-spring-ai-parts-advisors-memory-rag-mcp)
25. [Security and Safety Rules](#25-security-and-safety-rules)
26. [Best Practices](#26-best-practices)
27. [Common Mistakes](#27-common-mistakes)
28. [Interview Questions and Answers](#28-interview-questions-and-answers)
29. [Cheat Sheet](#29-cheat-sheet)

---

## 1. The Problem: LLM Cannot Do Things

An LLM (like GPT or Claude) is a **text predictor**. It only has:

- Knowledge from its **training data** (old, frozen in time)
- The **text you send** in the prompt

So an LLM **cannot**:

| You ask | Why the LLM fails |
|---|---|
| "What is the weather in London right now?" | It has no live data |
| "What is my account balance?" | It cannot access your database |
| "Book an appointment for tomorrow 5 PM" | It cannot call your API |
| "What is today's date?" | It has no clock |
| "Send an email to my manager" | It cannot send emails |
| "What is 98234 × 77123?" | It guesses; math is not reliable |

Without tools, the LLM either says "I don't know" or, worse, **makes up an answer** (hallucination).

**Tool calling** solves this.

---

## 2. What Is a Tool?

A **tool** is a normal piece of code (in Spring AI: a **Java method**) that you give to the LLM,
together with a **description** that explains what the method does.

> **Very important idea:**
> The LLM **never runs your code**. The LLM only says: *"Please run tool X with these arguments."*
> **Your Spring application** runs the method and sends the result back to the LLM.

Think of it like a manager and an assistant:

- The **LLM** is a smart manager who cannot touch the computer.
- **Your Java app** is the assistant who has the keys.
- The manager says: *"Call `getWeather` with city = London."*
- The assistant runs it and says: *"Result: 14°C, cloudy."*
- The manager then writes a nice answer for the customer.

Tool calling is also called **function calling** (older name, same idea).

Two types of tools in general:

| Type | Meaning | Example |
|---|---|---|
| **Information retrieval** | Read data, no side effects | get weather, get balance, search documents |
| **Taking action** | Change something in the world | book appointment, send email, create ticket |

---

## 3. Why Do We Use Tools?

1. **Real-time data** – weather, stock price, order status, current time.
2. **Private data** – your database, your company APIs. The LLM was never trained on them.
3. **Actions** – the AI can *do* things, not only talk.
4. **Accuracy** – use code for math, date calculation, validation. Code is exact; LLM is not.
5. **Less hallucination** – the LLM reads real data instead of guessing.
6. **Reuse existing code** – your existing services (`AppointmentService`, `OrderService`) become AI-ready with one annotation.

---

## 4. When Do We Use Tools (and When Not)?

### Use tools when

- The answer needs **data that changes** (prices, status, time).
- The answer needs **your private system** (DB, internal API).
- The user wants the AI to **perform an action**.
- You need **exact calculation or validation**.
- The AI must **choose between many operations** based on what the user says.

### Do NOT use tools when

- The question can be answered from general knowledge ("What is polymorphism?").
- You always need the same data for every request. Then just **put it in the prompt** (or use RAG). It is cheaper and faster than a tool round-trip.
- The operation is dangerous and has no confirmation step (delete all data, transfer money). Add human approval first.
- A fixed, simple workflow is enough. Plain `if/else` code is simpler than an AI deciding.

### Tools vs RAG vs Prompt context (quick comparison)

| Need | Best choice |
|---|---|
| Static text knowledge (docs, PDFs, policies) | **RAG** (vector store) |
| Same small info every time | **Put in system prompt** |
| Live data or actions, decided by the AI | **Tools** |
| Dynamic data AND big documents | **RAG + Tools together** |

---

## 5. Benefits

- **Extends the LLM** from "talker" to "doer".
- **Fresh data** without retraining.
- **Your code stays in control** – the LLM only *requests*, Java *executes*.
- **Less code for you** – no manual JSON parsing. Spring AI generates the schema, parses arguments, calls the method, and converts the result.
- **Model-independent** – same `@Tool` code works with OpenAI, Anthropic, Ollama, Gemini, Mistral, etc. (as long as the model supports tool calling).
- **Composable** – the model can call several tools in one answer (e.g., get date, then check slots, then book).

---

## 6. Where Tools Are Used in Real Tech

| Area | Example tools |
|---|---|
| **Customer support bots** | `getOrderStatus`, `createTicket`, `refundOrder`, `getCustomerPlan` |
| **Banking / fintech assistants** | `getBalance`, `getLastTransactions`, `blockCard` |
| **Booking systems** (hospital, salon, hotel, travel) | `checkAvailableSlots`, `bookAppointment`, `cancelAppointment` |
| **E-commerce** | `searchProducts`, `checkStock`, `addToCart` |
| **DevOps / SRE assistants** | `getPodStatus`, `fetchLogs`, `restartService` |
| **Internal company copilots** | `searchJira`, `readConfluence`, `queryEmployeeDirectory` |
| **Data / BI assistants** | `runSqlQuery` (read-only!), `getSalesReport` |
| **Coding assistants** (Cursor, Copilot agent mode, Claude Code) | `readFile`, `editFile`, `runTests`, `runShell` |
| **Personal assistants** | `getCalendar`, `sendEmail`, `setReminder` |
| **MCP (Model Context Protocol) servers** | Standard way to expose tools to many AI apps |

Almost every **AI agent** you hear about = **LLM + tools + loop**.

---

## 7. What We Can Achieve: AI App + Tools

Without tools your app is a **chatbot**. With tools your app becomes an **AI assistant / agent**.

Example: user types *"Book a dentist visit next Monday morning and tell me the doctor name."*

The AI can:

1. Call `getCurrentDate()` → learns today is Thursday 1 Oct → "next Monday" = 5 Oct.
2. Call `getAvailableSlots(date="2026-10-05", period="MORNING")` → gets slots.
3. Call `bookAppointment(slotId=42, patientId=...)` → booked.
4. Write final answer: *"Done! Booked Monday 5 Oct, 10:00 with Dr. Smith."*

You wrote **three simple Java methods**. The LLM planned and chained them.

---

## 8. Big Picture: Who Does What?

```
 ┌──────────┐      ┌──────────────────────────────┐        ┌────────────┐
 │  User    │      │      Your Spring Boot App     │        │    LLM     │
 │ (client) │      │                               │        │ (OpenAI,   │
 └────┬─────┘      │  ChatClient / ChatModel       │        │  Claude..) │
      │            │  ToolCallingManager           │        └─────┬──────┘
      │  question  │  ToolCallbacks (your methods) │              │
      ├───────────►│                               │              │
      │            │ 1. send prompt + tool list ───┼─────────────►│
      │            │                               │              │ decides
      │            │ 2. ◄── "call tool X(args)" ───┼──────────────┤
      │            │                               │              │
      │            │ 3. run Java method X(args)    │              │
      │            │                               │              │
      │            │ 4. send tool result ──────────┼─────────────►│
      │            │                               │              │
      │            │ 5. ◄── final text answer ─────┼──────────────┤
      │  answer    │                               │              │
      │◄───────────┤                               │              │
```

| Part | Responsibility |
|---|---|
| **You (developer)** | Write tool methods + good descriptions |
| **Spring AI** | Make JSON schema, send tools to model, detect tool request, run method, send result back, repeat |
| **LLM** | Decide *if* a tool is needed, *which* one, and *with what arguments*; then write the final answer |

---

## 9. Step-by-Step Flow (Request to Final Answer)

Let us follow one real example.

**Tool:**

```java
@Tool(description = "Get the current weather for a city")
public String getWeather(@ToolParam(description = "City name") String city) { ... }
```

**User asks:** "Should I take an umbrella in London today?"

### Step 1 – Spring AI builds the request

Spring AI converts every tool into a **tool definition** (name + description + JSON schema of parameters) and sends it **with the prompt**:

```json
{
  "messages": [
    { "role": "user", "content": "Should I take an umbrella in London today?" }
  ],
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "getWeather",
        "description": "Get the current weather for a city",
        "parameters": {
          "type": "object",
          "properties": {
            "city": { "type": "string", "description": "City name" }
          },
          "required": ["city"]
        }
      }
    }
  ]
}
```

### Step 2 – The model answers with a TOOL CALL (not text)

The model reads the question + tool list and thinks: *"I need live weather."*
It returns a special assistant message:

```json
{
  "role": "assistant",
  "content": null,
  "tool_calls": [
    {
      "id": "call_abc123",
      "type": "function",
      "function": {
        "name": "getWeather",
        "arguments": "{\"city\":\"London\"}"
      }
    }
  ]
}
```

The finish reason here is `tool_calls` (not `stop`). This is the **signal** for Spring AI.

### Step 3 – Spring AI executes your Java method

`ToolCallingManager` does:

1. Find tool named `getWeather` (from the callbacks you registered).
2. Convert JSON `{"city":"London"}` → Java argument `"London"`.
3. Call `getWeather("London")` using reflection.
4. Convert the return value to a JSON/String result.

### Step 4 – Spring AI sends the result back

The conversation is now:

```
USER:      Should I take an umbrella in London today?
ASSISTANT: (tool_call) getWeather({"city":"London"})
TOOL:      {"temp":"14C","condition":"Rain"}      <-- tool_call_id = call_abc123
```

This whole history is sent to the model again.

### Step 5 – The model writes the final answer

The model sees the result and now returns normal text, finish reason `stop`:

> "Yes, it is raining in London (14°C). Take an umbrella!"

### Step 6 – You get the answer

`chatClient....call().content()` returns that final text. **All the tool steps happened inside that one call.**
Your controller code does not see the middle steps.

---

## 10. The State Machine (Loop) Inside Spring AI

Tool calling is a **loop**. The model may need **more than one** tool, or the same tool several times.

```
                    ┌──────────────────────────┐
                    │  START: user prompt      │
                    └────────────┬─────────────┘
                                 ▼
                    ┌──────────────────────────┐
              ┌────►│  Call the LLM            │
              │     │  (prompt + tool defs +   │
              │     │   history so far)        │
              │     └────────────┬─────────────┘
              │                  ▼
              │     ┌──────────────────────────┐
              │     │ Response has tool calls? │
              │     └───────┬──────────┬───────┘
              │          YES│          │NO
              │             ▼          ▼
              │  ┌─────────────────┐  ┌──────────────────────┐
              │  │ Execute tool(s) │  │ DONE: return final   │
              │  │ via             │  │ text to caller       │
              │  │ ToolCalling-    │  └──────────────────────┘
              │  │ Manager         │
              │  └────────┬────────┘
              │           ▼
              │  ┌─────────────────────┐   YES   ┌───────────────────────┐
              │  │ returnDirect = true?├────────►│ DONE: return tool     │
              │  └────────┬────────────┘         │ result directly to    │
              │           │NO                    │ caller (skip LLM)     │
              │           ▼                      └───────────────────────┘
              │  ┌─────────────────────┐
              │  │ Add tool result to  │
              └──┤ conversation history│
                 └─────────────────────┘
```

### States in plain words

| State | Meaning |
|---|---|
| **Calling model** | Request sent, waiting for answer |
| **Tool requested** | Model returned `tool_calls` (finish reason = TOOL_CALLS) |
| **Executing tool** | Spring AI is running your Java method(s) |
| **Tool result ready** | Result wrapped in a `ToolResponseMessage` and added to history |
| **Calling model again** | History + tool result sent back |
| **Done** | Model returned plain text (finish reason = STOP), or `returnDirect` stopped the loop |

### Message types in the history

| Spring AI class | Role | Meaning |
|---|---|---|
| `SystemMessage` | system | Rules for the AI |
| `UserMessage` | user | What the user typed |
| `AssistantMessage` | assistant | AI text, **or** AI text + `toolCalls` list |
| `ToolResponseMessage` | tool | Results of tool execution (list of `ToolResponse`) |

So one full tool round-trip adds **two messages** to history:
`AssistantMessage(with toolCalls)` + `ToolResponseMessage(results)`.

> Tip: Because each loop is a **new LLM call**, tool calling costs more tokens and time.
> Two tools in sequence = 3 model calls (1 to pick tool A, 1 to pick tool B, 1 for final text).

### Parallel tool calls

Some models can request **several tools in one response** (e.g., weather for London AND Paris).
Then `toolCalls` has 2 items; Spring AI executes both and returns both results together in one `ToolResponseMessage`.

---

## 11. Core Classes and Interfaces

This is the "inside of the engine".

```
 @Tool method / Function bean
          │ (converted into)
          ▼
   ┌───────────────┐        has        ┌──────────────────┐
   │ ToolCallback  │──────────────────►│ ToolDefinition   │ name, description, inputSchema
   │ (interface)   │──────────────────►│ ToolMetadata     │ returnDirect
   └──────┬────────┘                   └──────────────────┘
          │ implemented by
   ┌──────┴─────────────┬────────────────────┐
   │ MethodToolCallback │ FunctionToolCallback│
   │ (for @Tool methods)│ (for Function/      │
   │                    │  Supplier/Consumer) │
   └────────────────────┴─────────────────────┘

   ToolCallingManager ── executes ──► ToolCallback.call(json)
   ToolCallbackResolver ── finds callback by NAME (when you pass names)
```

### 11.1 `ToolCallback` (the most important interface)

The runtime representation of **one tool**.

```java
public interface ToolCallback {
    ToolDefinition getToolDefinition();                    // name, description, JSON schema
    default ToolMetadata getToolMetadata() { ... }         // returnDirect etc.
    String call(String toolInput);                         // run tool with JSON string input
    String call(String toolInput, ToolContext toolContext);// run tool with extra context
}
```

- Input = **JSON string** of arguments (exactly what the model sent).
- Output = **String** (usually JSON) that goes back to the model.

### 11.2 `ToolDefinition`

What the **model sees** about the tool.

```java
public interface ToolDefinition {
    String name();         // e.g. "getWeather"  (defaults to method name)
    String description();  // tells the model WHEN and HOW to use it
    String inputSchema();  // JSON Schema of parameters
}
```

### 11.3 `ToolMetadata`

Extra behavior settings, mainly `returnDirect()`.

### 11.4 `MethodToolCallback`

The `ToolCallback` for methods annotated with `@Tool`. It stores:

- the `Method` (reflection object)
- the target object (your bean/instance)
- the `ToolDefinition`
- a `ToolCallResultConverter` (to turn the return value into String)

When called, it:
1. Parses the JSON input into Java arguments (Jackson).
2. Invokes the method by reflection.
3. Converts the return value to String.

### 11.5 `FunctionToolCallback`

Same idea but wraps a Java `Function<I,O>`, `Supplier<O>`, `Consumer<I>`, or `BiFunction<I,ToolContext,O>`.

### 11.6 `ToolCallbacks` (helper) and `MethodToolCallbackProvider`

```java
ToolCallback[] callbacks = ToolCallbacks.from(new WeatherTools()); // scan @Tool methods

ToolCallbackProvider provider = MethodToolCallbackProvider.builder()
        .toolObjects(new WeatherTools(), new AppointmentTools())
        .build();                                                  // used a lot with MCP servers
```

### 11.7 `ToolCallingManager` (the engine)

The component that **runs** the tool calls.

```java
public interface ToolCallingManager {
    List<ToolDefinition> resolveToolDefinitions(ToolCallingChatOptions chatOptions);
    ToolExecutionResult executeToolCalls(Prompt prompt, ChatResponse chatResponse);
}
```

- Default implementation: `DefaultToolCallingManager` (auto-configured bean).
- `executeToolCalls(...)`: reads tool calls from the `ChatResponse`, finds the right `ToolCallback`,
  runs it, builds the `ToolResponseMessage`, and returns a `ToolExecutionResult`.

### 11.8 `ToolExecutionResult`

```java
public interface ToolExecutionResult {
    List<Message> conversationHistory(); // history INCLUDING assistant tool-call msg + tool response msg
    boolean returnDirect();              // should we skip the LLM and return result to the user?
}
```

### 11.9 `ToolCallbackResolver`

Finds a `ToolCallback` **by name**. Used when you pass `.toolNames("getWeather")` or when a model response asks for a tool name.
It looks in: static registered callbacks, then **Spring beans** (functions registered as `@Bean`).

### 11.10 `ToolCallResultConverter`

Converts the **Java return value → String** for the model. Default uses **JSON (Jackson)**.
You can plug in your own with `@Tool(resultConverter = MyConverter.class)`.

### 11.11 `ToolContext`

A `Map<String,Object>` of **extra data** passed to the tool but **never sent to the model** (userId, tenantId, JWT...). See section 19.

### 11.12 `ToolCallingChatOptions`

Chat options that carry the tools:

| Property | Meaning |
|---|---|
| `toolCallbacks` | List of `ToolCallback` for this request |
| `toolNames` | Names to resolve through `ToolCallbackResolver` |
| `toolContext` | The context map |
| `internalToolExecutionEnabled` | `true` (default) = Spring AI runs the loop; `false` = you handle it yourself |

### 11.13 `ToolExecutionEligibilityPredicate`

A small predicate used by the `ChatModel` to decide: *"Is this response a tool request that I should execute?"*
Default checks that internal execution is enabled **and** the response actually contains tool calls.

### 11.14 `ToolExecutionExceptionProcessor`

Decides what to do if the tool throws an exception: send the error message to the model, or throw the exception to your app (section 21).

### Summary table

| Class / Interface | One-line job |
|---|---|
| `@Tool` / `@ToolParam` | Mark a method as a tool / describe a parameter |
| `ToolDefinition` | Name + description + JSON schema (model sees this) |
| `ToolCallback` | Executable tool |
| `MethodToolCallback` / `FunctionToolCallback` | Implementations of `ToolCallback` |
| `ToolCallingManager` | Runs the tool calls and builds the next conversation |
| `ToolCallbackResolver` | Finds tools by name |
| `ToolCallResultConverter` | Java result → String |
| `ToolContext` | Hidden extra data for tools |
| `ToolCallingChatOptions` | Carries tools into the request |

---

## 12. How the Model Decides Which Tool to Use

Important: **Spring AI does not decide. Your Java code does not decide. The LLM decides.**

The LLM only has these clues:

1. The **user's message** (and previous conversation).
2. The **tool name**.
3. The **tool description**.
4. The **parameter names and descriptions** (from JSON schema).
5. Your **system prompt** (if you gave rules like "always check availability before booking").

So the model is doing a **language understanding match**:

> "User wants live weather → tool `getWeather` says 'Get the current weather for a city' → matches → call it."

### What this means for you

- **Description quality = tool selection quality.** A bad description gives wrong or no tool calls.
- If two tools have similar descriptions, the model may choose the wrong one.
- If no tool matches, the model just answers normally with text.
- The model may call **zero, one, or many** tools.
- The model may **repeat** a tool with different arguments.
- The model cannot call a tool that was not sent in the request. Only the tools you register are possible.

### Bad vs good description

```java
// BAD - model has no idea when to use this
@Tool(description = "Gets data")
String get(String x) { ... }

// GOOD - tells WHAT it does, WHEN to use, and what comes back
@Tool(description = """
        Get the list of free appointment slots for a given date.
        Use this BEFORE booking, to know which times are available.
        Returns a list of slots with id and start time.
        """)
List<Slot> getAvailableSlots(
        @ToolParam(description = "Date in format yyyy-MM-dd, e.g. 2026-10-05") String date) { ... }
```

### Controlling decisions with the system prompt

```java
chatClient.prompt()
    .system("""
        You are a clinic assistant.
        Rules:
        - Never guess dates. Use getCurrentDate tool first if the user says 'tomorrow' or 'next Monday'.
        - Always call getAvailableSlots before bookAppointment.
        - Ask the user to confirm before booking.
        """)
    .user(userMessage)
    .tools(appointmentTools)
    .call()
    .content();
```

Some providers also let you force or disable tool use (for example `tool_choice` in OpenAI-style APIs) through provider-specific options.
This is not the same for every model, so check your provider's Spring AI page.

---

## 13. How Spring AI Runs Your Method

Here is the exact path inside `MethodToolCallback.call(String toolInput, ToolContext ctx)`:

```
 toolInput (JSON string): {"date":"2026-10-05"}
        │
        ▼
 1. Parse JSON → Map<String,Object>             (Jackson ObjectMapper)
        │
        ▼
 2. Match each JSON field to a method parameter (by parameter NAME / position)
    Convert each value to the parameter's Java type
    (String, int, LocalDate, enum, record, List<...> ...)
        │
        ▼
 3. If a parameter is of type ToolContext → inject it (not from JSON)
        │
        ▼
 4. Method.invoke(toolObject, args...)          (Java reflection)
        │
        ▼
 5. Take the return value → ToolCallResultConverter → String (JSON)
        │
        ▼
 6. Return String → ToolCallingManager wraps into ToolResponse
```

### Which method runs when there are many?

`ToolCallingManager` matches by **tool name**:

- Each tool call from the model has a `name` (e.g. `"getAvailableSlots"`).
- The manager looks for a `ToolCallback` whose `ToolDefinition.name()` equals that string.
- If found → run it. If not found → error (`IllegalStateException: No ToolCallback found for tool name ...`).

So **tool names must be unique** inside one request.
By default the tool name = the **Java method name**. You can change it:

```java
@Tool(name = "check_slots", description = "...")
```

Many providers want names without spaces and with only letters, numbers, `_` and `-`.

### Supported parameter / return types

Most normal Java types work: `String`, numbers, `boolean`, enums, `LocalDate`/`LocalDateTime`, `List`, `Map`, records, POJOs.

**Not supported** as parameters or return types: `Optional`, asynchronous types (`CompletableFuture`), reactive types (`Flux`, `Mono`), and functional types
(`Function`, `Supplier`, `Consumer`).

### Threading note

By default the tool runs on the **same thread** that is processing the request (blocking). Keep tools **fast** and add timeouts for slow external calls.

---

## 14. Your First Tool (Demo)

### 14.1 Dependency (Maven)

Use the starter for your model provider (example: OpenAI) and the Spring AI BOM. Tool calling is **built in**; no extra dependency.

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.ai</groupId>
      <artifactId>spring-ai-bom</artifactId>
      <version>${spring-ai.version}</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>

<dependencies>
  <dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-openai</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
</dependencies>
```

`application.properties`:

```properties
spring.ai.openai.api-key=${OPENAI_API_KEY}
spring.ai.openai.chat.options.model=gpt-4o-mini
```

> Your model **must support tool calling**. Small/old models may not. Check the model's capability table in Spring AI docs.

### 14.2 Write the tool class

```java
package com.example.ai.tools;

import java.time.LocalDate;
import java.time.LocalDateTime;
import org.springframework.ai.tool.annotation.Tool;
import org.springframework.stereotype.Component;

@Component
public class DateTimeTools {

    @Tool(description = "Get the current date and time of the server in ISO format")
    public String getCurrentDateTime() {
        return LocalDateTime.now().toString();
    }

    @Tool(description = "Get today's date in format yyyy-MM-dd")
    public String getToday() {
        return LocalDate.now().toString();
    }
}
```

That is all. `@Tool` + a good `description`.

### 14.3 Use it with `ChatClient`

```java
@RestController
@RequestMapping("/api/ai")
public class AiController {

    private final ChatClient chatClient;
    private final DateTimeTools dateTimeTools;

    public AiController(ChatClient.Builder builder, DateTimeTools dateTimeTools) {
        this.chatClient = builder.build();
        this.dateTimeTools = dateTimeTools;
    }

    @GetMapping("/ask")
    public String ask(@RequestParam String q) {
        return chatClient.prompt()
                .user(q)
                .tools(dateTimeTools)     // <-- register tool object for THIS request
                .call()
                .content();
    }
}
```

Test: `GET /api/ai/ask?q=What day will it be in 10 days?`

What happens:

1. LLM sees `getToday` tool → asks to call it.
2. Spring AI runs `getToday()` → `"2026-10-01"`.
3. LLM calculates +10 days → answers "11 October 2026".

You can see the steps by enabling debug logs:

```properties
logging.level.org.springframework.ai=DEBUG
```

---

## 15. Ways to Add Tools to a Request

| # | How | Code | When to use |
|---|---|---|---|
| 1 | **Tool objects** (`@Tool` methods) | `.tools(obj1, obj2)` | Most common, simplest |
| 2 | **Tool callbacks** (built manually or by `ToolCallbacks.from`) | `.toolCallbacks(cb1, cb2)` | When you build callbacks in code, or use MCP providers |
| 3 | **Tool names** (resolved from Spring beans) | `.toolNames("beanName")` | When the tool is a `Function` bean |
| 4 | **Default tools** (for every request of this client) | `builder.defaultTools(obj)` | Tools every request needs |
| 5 | **ChatModel options** | `ToolCallingChatOptions.builder().toolCallbacks(...)` | When using `ChatModel` directly |

### Per-request (dynamic) tools

```java
chatClient.prompt().user(q).tools(new DateTimeTools()).call().content();
```

Good because you can **choose tools per user or per use case** (admin gets more tools than a normal user).

### Default tools (every request)

```java
ChatClient chatClient = ChatClient.builder(chatModel)
        .defaultTools(new DateTimeTools())
        .build();
```

You can still add more per request with `.tools(...)`. Both sets are sent.

### Using the ChatModel directly

```java
ToolCallback[] callbacks = ToolCallbacks.from(new DateTimeTools());

ChatOptions options = ToolCallingChatOptions.builder()
        .toolCallbacks(callbacks)
        .build();

Prompt prompt = new Prompt("What day is tomorrow?", options);
ChatResponse response = chatModel.call(prompt);
```

### Tool by name (Function bean)

```java
@Bean
@Description("Get the weather for a given city")
public Function<WeatherRequest, WeatherResponse> currentWeather() {
    return request -> new WeatherResponse(request.city(), "14C", "Rain");
}

// usage
chatClient.prompt().user("Weather in Paris?").toolNames("currentWeather").call().content();
```

### Which one should I use?

- Start with **`.tools(object)`** and `@Tool`. It is the cleanest.
- Use **`defaultTools`** for tools needed on every request.
- Use **`toolCallbacks`** for MCP or programmatic cases.
- Use **`toolNames`** if you prefer Spring beans.

---

## 16. Tool Parameters: `@ToolParam` and Custom Request Params

The **parameters of the Java method** become the **parameters of the tool**.
Spring AI reads your method signature and builds a JSON schema automatically.

### 16.1 Basic: add parameters

```java
@Tool(description = "Get free appointment slots for a doctor on a date")
public List<String> getSlots(String doctorName, String date) { ... }
```

Generated schema (simplified):

```json
{
  "type": "object",
  "properties": {
    "doctorName": { "type": "string" },
    "date":       { "type": "string" }
  },
  "required": ["doctorName", "date"]
}
```

This works, but the model has **no hints** about the format. So we add `@ToolParam`.

### 16.2 `@ToolParam`: describe the parameter

```java
@Tool(description = "Get free appointment slots for a doctor on a date")
public List<String> getSlots(
        @ToolParam(description = "Full name of the doctor, e.g. 'Dr. Anna Smith'") String doctorName,
        @ToolParam(description = "Date in format yyyy-MM-dd, e.g. 2026-10-05") String date) {
    ...
}
```

`@ToolParam` attributes:

| Attribute | Meaning | Default |
|---|---|---|
| `description` | Explains the param to the model (format, example, allowed values) | none |
| `required` | Is the param mandatory? | `true` |

### 16.3 Optional parameters

```java
@Tool(description = "Search appointments of a patient")
public List<Appointment> searchAppointments(
        @ToolParam(description = "Patient id") String patientId,
        @ToolParam(description = "Filter by status: BOOKED, CANCELLED or COMPLETED. Leave empty for all",
                   required = false) String status) {

    if (status == null || status.isBlank()) { ... all ... }
    ...
}
```

Rules about `required`:

- A parameter is **required by default**.
- Mark as optional with `@ToolParam(required = false)`.
- Optional parameters may arrive as `null` in your method. **Always null-check them.**
- Tip: use **wrapper types** (`Integer`, not `int`) for optional numbers. A primitive `int` cannot be `null`.

### 16.4 Enum parameters (limit the choices)

Enums are great: the model can only choose valid values. The schema contains the allowed list.

```java
public enum Period { MORNING, AFTERNOON, EVENING }

@Tool(description = "Get free slots for a date and part of the day")
public List<String> getSlotsByPeriod(
        @ToolParam(description = "Date yyyy-MM-dd") String date,
        @ToolParam(description = "Part of the day") Period period) { ... }
```

### 16.5 Date/time parameters

You can use `LocalDate` / `LocalDateTime` directly. Describe the format clearly:

```java
@Tool(description = "Cancel appointments on a given date")
public String cancelByDate(
        @ToolParam(description = "Date in ISO format yyyy-MM-dd") LocalDate date) { ... }
```

If you see conversion errors, switch the type to `String` and parse it yourself. Strings are the most forgiving.

### 16.6 Rename the tool and control its metadata

```java
@Tool(
    name = "book_appointment",            // name seen by the model (default = method name)
    description = "Book an appointment ...",
    returnDirect = false                  // default false
)
public String book(...) { ... }
```

### 16.7 Tips for good parameter design

| Do | Why |
|---|---|
| Give **format + example** in `description` | Model sends correct values |
| Use **enums** for fixed choices | Prevents invalid values |
| Keep parameters **few** (1-5) | Easier for the model |
| Use clear **names** (`patientId`, not `p1`) | The name is also part of the schema |
| Accept **simple types** when you can | Less chance of conversion errors |
| **Validate** inside the method | The model can send wrong values |

> Remember: parameter **names** are read by reflection. Compile with `-parameters` (Spring Boot Maven/Gradle plugin already does this) so real names appear, not `arg0`, `arg1`.

---

## 17. Complex Parameters (Records / Objects / Lists)

### 17.1 Record as a parameter

```java
public record BookingRequest(
        @JsonPropertyDescription("Patient id, e.g. P-1001") String patientId,
        @JsonPropertyDescription("Doctor name") String doctorName,
        @JsonPropertyDescription("Start time in ISO format, e.g. 2026-10-05T10:00") String startTime,
        @JsonPropertyDescription("Reason for the visit") String reason) {}

@Tool(description = "Book a new appointment")
public String bookAppointment(BookingRequest request) {
    ...
}
```

- For fields **inside** a record/class, describe them with Jackson annotations:
  - `@JsonPropertyDescription("...")` – description of a field
  - `@JsonClassDescription("...")` – description of the whole class
  - `@JsonProperty(required = true)` – mark required fields
- `@ToolParam` is for **method parameters**. Jackson annotations are for **fields inside objects**.

Alternatively you can combine both:

```java
@Tool(description = "Book a new appointment")
public String bookAppointment(
        @ToolParam(description = "Booking details") BookingRequest request) { ... }
```

### 17.2 List parameter

```java
@Tool(description = "Check stock for several product ids at once")
public Map<String, Integer> checkStock(
        @ToolParam(description = "List of product ids") List<String> productIds) { ... }
```

### 17.3 Return a record / list

```java
public record Slot(long id, String start, String doctor) {}

@Tool(description = "List free slots for a date")
public List<Slot> freeSlots(@ToolParam(description = "yyyy-MM-dd") String date) {
    return List.of(new Slot(1, "10:00", "Dr. Smith"), new Slot(2, "11:00", "Dr. Smith"));
}
```

Spring AI converts the list to JSON, for example:
`[{"id":1,"start":"10:00","doctor":"Dr. Smith"}, ...]`
The model reads this JSON and explains it in normal language to the user.

---

## 18. Return Values and `returnDirect`

### 18.1 What can a tool return?

Almost any serializable type: `String`, numbers, `boolean`, records, POJOs, `List`, `Map`, `void`.
Spring AI converts the value to **JSON text** and sends it to the model.

Notes:

- `void` returns something like `"Done"` to the model.
- Return a **clear, small** result. Do not return a huge object with 100 fields. It wastes tokens and confuses the model.
- Return **useful error messages as text** when the situation is expected (for example `"No free slots on this date"`), so the model can explain it to the user.

### 18.2 `returnDirect = true`

Normal flow: tool result → back to LLM → LLM writes final answer.
With `returnDirect = true`: tool result → **straight to the caller**, **the LLM is skipped**.

```java
@Tool(description = "Get the full details of an appointment as JSON", returnDirect = true)
public AppointmentDto getAppointment(@ToolParam(description = "Appointment id") long id) { ... }
```

Use it when:

- You want **exact data**, not an LLM re-write (a JSON response for a UI).
- You want to **save one LLM call** (cost + latency).

Do not use it when you want the model to explain or combine results. Remember: the caller receives the **raw tool result**.

---

## 19. `ToolContext`: Extra Data the Model Must NOT See

Sometimes your tool needs data that should **not** go to the LLM: logged-in `userId`, `tenantId`, JWT, request id.
**Never let the model provide the user id.** The model could be tricked (prompt injection) into using another user's id.

Use `ToolContext`:

```java
@Component
public class MyAppointmentTools {

    @Tool(description = "Get the upcoming appointments of the current user")
    public List<Appointment> myAppointments(ToolContext toolContext) {
        String userId = (String) toolContext.getContext().get("userId");   // from the server, safe
        return appointmentService.findUpcoming(userId);
    }
}
```

Passing the context:

```java
chatClient.prompt()
        .user("What appointments do I have?")
        .tools(myAppointmentTools)
        .toolContext(Map.of("userId", currentUser.getId()))   // not sent to the model
        .call()
        .content();
```

Facts:

- `ToolContext` parameter is **not** part of the JSON schema. The model never sees it.
- You can combine it with normal params: `myTool(@ToolParam(...) String date, ToolContext ctx)`.
- Inside, read values with `toolContext.getContext()` (a `Map<String,Object>`).

---

## 20. Other Ways to Define Tools (Function Bean, Programmatic)

### 20.1 `@Tool` on methods (recommended)

Already shown. Simple and readable.

### 20.2 Programmatic: `MethodToolCallback`

Use when you need full control (no annotation, dynamic names).

```java
Method method = ReflectionUtils.findMethod(DateTimeTools.class, "getToday");

ToolCallback toolCallback = MethodToolCallback.builder()
        .toolDefinition(ToolDefinitions.builder(method)
                .name("getToday")
                .description("Get today's date in format yyyy-MM-dd")
                .build())
        .toolMethod(method)
        .toolObject(new DateTimeTools())
        .build();

chatClient.prompt().user("What is today's date?").toolCallbacks(toolCallback).call().content();
```

### 20.3 Programmatic: `FunctionToolCallback`

Wrap a Java `Function`:

```java
public record WeatherRequest(
        @JsonProperty(required = true) @JsonPropertyDescription("City name") String city,
        @JsonPropertyDescription("Unit: C or F") String unit) {}

public record WeatherResponse(String city, double temp, String condition) {}

public class WeatherService implements Function<WeatherRequest, WeatherResponse> {
    @Override
    public WeatherResponse apply(WeatherRequest request) {
        return new WeatherResponse(request.city(), 14.0, "Rain");
    }
}

ToolCallback weatherTool = FunctionToolCallback
        .builder("currentWeather", new WeatherService())
        .description("Get the current weather in a city")
        .inputType(WeatherRequest.class)       // needed so Spring AI can build the JSON schema
        .build();
```

### 20.4 Dynamic: `Function` as a Spring `@Bean`

```java
@Configuration
class ToolConfig {

    @Bean
    @Description("Get the current weather in a city")        // used as the tool description
    Function<WeatherRequest, WeatherResponse> currentWeather() {
        return new WeatherService();
    }
}

// use by bean name
chatClient.prompt().user("Weather in Rome?").toolNames("currentWeather").call().content();
```

### Which style to pick?

| Style | Best for |
|---|---|
| `@Tool` method | 90% of cases: your service methods |
| `MethodToolCallback` | No annotation access, or dynamic creation |
| `FunctionToolCallback` | Lambda / functional style tools |
| `@Bean Function` | Tools picked **by name** at runtime |

---

## 21. Error Handling

What if the tool throws an exception (DB down, bad input)?

Default behavior: `DefaultToolExecutionExceptionProcessor`. By default it **sends the error message back to the model as the tool result**, so the model can say "Sorry, I could not do that" or try again with different arguments.

Config property to change the behavior:

```properties
# true  = throw the exception to your application (the whole call fails)
# false = send the error message to the AI model (default behavior)
spring.ai.tools.throw-exception-on-error=false
```

Custom handling:

```java
@Bean
ToolExecutionExceptionProcessor toolExecutionExceptionProcessor() {
    return exception -> {
        // log it
        log.error("Tool failed: {}", exception.getMessage(), exception);
        // return a SAFE message for the model (do not leak stack traces or secrets)
        return "The tool failed. Tell the user to try again later.";
    };
}
```

Good habits:

- **Validate input** in the tool and return friendly text for expected problems.
- Throw exceptions only for **unexpected** problems.
- Never put secrets, SQL, or stack traces in the message that goes to the model.

---

## 22. Manual Tool Execution (You Control the Loop)

By default Spring AI runs the tool loop **for you** (`internalToolExecutionEnabled = true`).
Sometimes you want control: human approval before a dangerous tool, logging each step, custom persistence, or a custom stop condition.

Turn off the automatic execution and run the loop yourself:

```java
@Service
public class ManualToolService {

    private final ChatModel chatModel;
    private final ToolCallingManager toolCallingManager;

    public ManualToolService(ChatModel chatModel, ToolCallingManager toolCallingManager) {
        this.chatModel = chatModel;
        this.toolCallingManager = toolCallingManager;
    }

    public String ask(String question) {
        ToolCallback[] tools = ToolCallbacks.from(new DateTimeTools());

        ToolCallingChatOptions options = ToolCallingChatOptions.builder()
                .toolCallbacks(tools)
                .internalToolExecutionEnabled(false)    // <-- YOU run the tools
                .build();

        Prompt prompt = new Prompt(question, options);
        ChatResponse response = chatModel.call(prompt);

        while (response.hasToolCalls()) {
            // (optional) ask a human / log / check permissions here

            ToolExecutionResult result = toolCallingManager.executeToolCalls(prompt, response);

            if (result.returnDirect()) {
                // last message of the history is the tool response, return it directly
                return result.conversationHistory().get(result.conversationHistory().size() - 1).getText();
            }

            // new prompt = full history (user + assistant tool call + tool response)
            prompt = new Prompt(result.conversationHistory(), options);
            response = chatModel.call(prompt);
        }

        return response.getResult().getOutput().getText();
    }
}
```

This is exactly what Spring AI does internally in the automatic mode.
The loop shape is the same as the diagram in section 10.

---

## 23. Full Mini Project: Appointment Assistant

An AI assistant that can check slots and book/cancel appointments. This follows a typical booking service idea.

### 23.1 Domain

```java
public record Slot(long id, String doctor, String startTime) {}
public record Appointment(long id, String patientId, String doctor, String startTime, String status) {}
```

### 23.2 Existing service (your normal business code)

```java
@Service
public class AppointmentService {

    private final Map<Long, Appointment> store = new ConcurrentHashMap<>();
    private final AtomicLong ids = new AtomicLong(100);

    public List<Slot> freeSlots(LocalDate date) {
        return List.of(
            new Slot(1, "Dr. Smith", date + "T10:00"),
            new Slot(2, "Dr. Smith", date + "T11:00"),
            new Slot(3, "Dr. Lee",   date + "T14:00"));
    }

    public Appointment book(String patientId, long slotId, String reason) {
        // real code: check slot is still free, save in DB, etc.
        long id = ids.incrementAndGet();
        Appointment a = new Appointment(id, patientId, "Dr. Smith", "2026-10-05T10:00", "BOOKED");
        store.put(id, a);
        return a;
    }

    public List<Appointment> findByPatient(String patientId) {
        return store.values().stream().filter(a -> a.patientId().equals(patientId)).toList();
    }

    public Appointment cancel(long appointmentId, String patientId) {
        Appointment old = store.get(appointmentId);
        if (old == null || !old.patientId().equals(patientId)) {
            throw new IllegalArgumentException("Appointment not found");
        }
        Appointment cancelled = new Appointment(old.id(), old.patientId(), old.doctor(), old.startTime(), "CANCELLED");
        store.put(appointmentId, cancelled);
        return cancelled;
    }
}
```

### 23.3 The tool class (thin wrapper on the service)

```java
@Component
public class AppointmentTools {

    private final AppointmentService service;

    public AppointmentTools(AppointmentService service) {
        this.service = service;
    }

    @Tool(description = "Get today's date in format yyyy-MM-dd. Use it to understand words like 'tomorrow' or 'next Monday'.")
    public String getToday() {
        return LocalDate.now().toString();
    }

    @Tool(description = """
            List free appointment slots for one date.
            Always call this BEFORE booking, so you know valid slot ids.
            """)
    public List<Slot> getFreeSlots(
            @ToolParam(description = "Date in format yyyy-MM-dd, e.g. 2026-10-05") String date) {
        return service.freeSlots(LocalDate.parse(date));
    }

    @Tool(description = """
            Book an appointment for the CURRENT user in a free slot.
            Only call this after the user clearly confirmed the slot.
            """)
    public Appointment bookAppointment(
            @ToolParam(description = "Slot id from getFreeSlots") long slotId,
            @ToolParam(description = "Short reason for the visit", required = false) String reason,
            ToolContext toolContext) {

        String patientId = (String) toolContext.getContext().get("patientId");   // hidden from the model
        return service.book(patientId, slotId, reason == null ? "General" : reason);
    }

    @Tool(description = "List all appointments of the current user")
    public List<Appointment> myAppointments(ToolContext toolContext) {
        return service.findByPatient((String) toolContext.getContext().get("patientId"));
    }

    @Tool(description = "Cancel an appointment of the current user. Only call after user confirmation.")
    public Appointment cancelAppointment(
            @ToolParam(description = "Appointment id") long appointmentId,
            ToolContext toolContext) {
        return service.cancel(appointmentId, (String) toolContext.getContext().get("patientId"));
    }
}
```

### 23.4 Controller

```java
@RestController
@RequestMapping("/api/assistant")
public class AssistantController {

    private final ChatClient chatClient;
    private final AppointmentTools tools;

    public AssistantController(ChatClient.Builder builder, AppointmentTools tools) {
        this.chatClient = builder
                .defaultSystem("""
                    You are a friendly clinic assistant.
                    Rules:
                    1. Use getToday when the user mentions relative dates.
                    2. Always call getFreeSlots before bookAppointment.
                    3. Never book or cancel without the user's explicit confirmation.
                    4. If a tool returns an error, explain it simply and offer another option.
                    """)
                .build();
        this.tools = tools;
    }

    @PostMapping("/chat")
    public String chat(@RequestBody String message, Principal principal) {
        return chatClient.prompt()
                .user(message)
                .tools(tools)
                .toolContext(Map.of("patientId", principal.getName()))   // from login, NOT from the model
                .call()
                .content();
    }
}
```

### 23.5 Example conversation and what happens inside

**User:** "Do you have any slots next Monday?"

```
LLM call #1  → tool_call: getToday()
Java         → "2026-10-01"
LLM call #2  → tool_call: getFreeSlots({"date":"2026-10-05"})
Java         → [{id:1,...10:00},{id:2,...11:00},{id:3,...14:00}]
LLM call #3  → final text: "Yes! Monday 5 Oct: 10:00 and 11:00 with Dr. Smith, 14:00 with Dr. Lee. Want one?"
```

**User:** "Book the 11:00 one for a check-up."

```
LLM call #1  → tool_call: bookAppointment({"slotId":2,"reason":"check-up"})
Java         → Appointment{id:101,status:BOOKED,...}   (patientId from ToolContext)
LLM call #2  → final text: "Your check-up is booked. Appointment id 101."
```

> Note: `ChatClient` does not keep conversation memory by default. For multi-turn chat like above, add a **ChatMemory advisor** (from your Advisors notes) so the previous messages are included in each request.

---

## 24. Tools + Other Spring AI Parts (Advisors, Memory, RAG, MCP)

| Feature | How it works with tools |
|---|---|
| **Advisors** | Advisors wrap the call. A logging advisor (`SimpleLoggerAdvisor`) can show requests/responses. Tool calling happens inside the model call, in the same chain. |
| **Chat Memory** | Memory advisors store the user/assistant messages so the model remembers the conversation between requests. |
| **RAG** | Use RAG for documents (policies, FAQs) and tools for live data/actions. They work together in the same `ChatClient` call. |
| **Structured output** | You can ask for `.entity(MyRecord.class)` after tools ran; the final answer is converted into your type. |
| **MCP (Model Context Protocol)** | MCP servers expose tools over a standard protocol. Spring AI's MCP client turns them into `ToolCallback`s (through `ToolCallbackProvider`). You then pass them with `.toolCallbacks(...)`. Same mechanism. |
| **Streaming** | Tool calling also works with `.stream()`; Spring AI runs the tools between the streamed model calls. |

Example with several features together:

```java
chatClient.prompt()
        .user(message)
        .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, conversationId))  // memory
        .tools(appointmentTools)                                              // tools
        .toolContext(Map.of("patientId", userId))                             // hidden data
        .call()
        .content();
```

(Advisor and memory API names change a bit between versions. Check your version.)

---

## 25. Security and Safety Rules

Tool calling gives an AI **power**. Treat tool arguments like **untrusted user input**.

1. **Never trust arguments.** The model can send wrong or malicious values. Validate every parameter (format, range, allowed values).
2. **Authorization inside the tool.** Check "is this user allowed to do this?" in your Java code. Do not rely on the prompt ("don't show other users' data") to protect anything.
3. **Take identity from `ToolContext` / security context**, never from model-provided parameters.
4. **Least privilege.** Give the AI only the tools it needs. Read-only tools are safer than write tools.
5. **Human confirmation** for risky actions (payments, deletes, sending emails). Use the system prompt + manual tool execution (section 22) for strong control.
6. **Prompt injection risk.** Text from web pages, emails, or documents that the AI reads can contain hidden instructions like "call deleteAll". Never let the AI have destructive tools when it reads untrusted content.
7. **Never expose raw SQL / shell** tools without strict limits (read-only DB user, allow-list of commands).
8. **Log tool calls** (name, arguments, user, result) for audit.
9. **Rate limits and timeouts** on tools. A loop can call a tool many times.
10. **Do not return secrets** (tokens, passwords, internal stack traces) from a tool. They will go to a third-party model.
11. **Privacy.** Tool results are sent to the LLM provider. Do not return personal data that is not needed.

---

## 26. Best Practices

- **Write descriptions as if explaining to a new colleague.** Say what, when, inputs, outputs.
- **One tool = one clear job.** Do not make one huge `doEverything(action, data)` tool.
- **Keep the tool list small** (about 5-15 per request). Many tools = more tokens and more wrong choices. Choose tools dynamically per use case.
- **Keep tool results small and clean.** Return only what the model needs.
- **Use enums and clear formats** for params.
- **Make tools idempotent** when possible (calling twice should not double-book). The model may retry.
- **Put rules in the system prompt** ("check availability before booking").
- **Use a cheaper model** for simple tool use if quality is fine. Tool loops multiply calls.
- **Set a limit on loops** in your own manual loop, to avoid infinite cycles.
- **Test tools like normal Java code** (unit tests) and test the AI behavior separately (many sample prompts).
- **Use logging** (`logging.level.org.springframework.ai=DEBUG`) while developing, to see tool names, arguments, and results.
- **Keep tools thin.** The tool method should call your existing service, not contain the business logic.

---

## 27. Common Mistakes

| Mistake | Result | Fix |
|---|---|---|
| Missing or vague `description` | Model never uses the tool or uses it wrongly | Write a clear, detailed description |
| Model does not support tools | Tools ignored or error | Choose a tool-capable model |
| Parameter names are `arg0`, `arg1` | Model confused | Compile with `-parameters` |
| Two tools with the same name | Wrong tool or error | Use unique names |
| Using `Optional`, `Flux`, `Function` as param/return | Not supported | Use plain types |
| `int` for an optional number | `null` cannot be mapped | Use `Integer` |
| Private/final methods or `static` problems | Reflection issues | Use public instance methods on a Spring bean or normal object |
| Forgot to pass the tools in the request | Model answers without tools | `.tools(...)` or `defaultTools` |
| Tool returns huge data | High cost, confused model | Return summary or limit fields |
| Letting the model provide `userId` | Security hole | Use `ToolContext` |
| Expecting memory between requests | Model forgets previous messages | Add chat memory |
| Dangerous action without confirmation | Accidental actions | Confirmation step / manual execution |
| Slow tool (no timeout) | Request hangs | Add timeouts, keep tools fast |
| Not checking `null` of optional params | `NullPointerException` | Null-check |

---

## 28. Interview Questions and Answers

**Q1. What is tool calling?**
It is a feature where the LLM can ask the application to run a function. The model returns the tool name and arguments; the application runs it and sends the result back; the model then writes the final answer.

**Q2. Does the LLM execute the code?**
No. The LLM only produces a structured request. Spring AI (your app) executes the Java method.

**Q3. How does the model know about the tools?**
Spring AI sends a list of tool definitions (name, description, JSON schema of parameters) together with the prompt.

**Q4. What annotation marks a tool in Spring AI?**
`@Tool` on a method, with `@ToolParam` for parameter descriptions.

**Q5. Which component executes the tools?**
`ToolCallingManager` (default: `DefaultToolCallingManager`) through `ToolCallback` objects.

**Q6. What is `ToolCallback`?**
The interface that represents an executable tool: definition + metadata + `call(String json)`.

**Q7. What is `returnDirect`?**
If `true`, the tool result is returned directly to the caller and not sent back to the LLM.

**Q8. What is `ToolContext`?**
A map of extra data (userId, tenant, etc.) passed to the tool but never sent to the model.

**Q9. How do you stop Spring AI from running tools automatically?**
Set `internalToolExecutionEnabled(false)` in `ToolCallingChatOptions`, then call `ToolCallingManager.executeToolCalls(...)` yourself.

**Q10. Tool calling vs RAG?**
RAG adds relevant **text knowledge** into the prompt (read-only). Tools let the model fetch **live data** or **perform actions**. They are often used together.

**Q11. How many LLM calls does one tool use cause?**
At least two: one that returns the tool request, and one after the tool result that returns the final answer.

**Q12. How do you make the model choose the right tool?**
Clear tool names, strong descriptions, clear parameter descriptions, and a good system prompt. Keep the tool list small.

**Q13. What happens if a tool throws an exception?**
By default the error message is sent back to the model as the tool result; you can configure it to throw to your app instead or provide a custom `ToolExecutionExceptionProcessor`.

**Q14. Security concerns?**
Untrusted arguments, prompt injection, authorization, data leaking to the provider, destructive actions. Validate, authorize in code, use least privilege, and ask for confirmation.

---

## 29. Cheat Sheet

```java
// 1. Define tools
@Component
class MyTools {
    @Tool(description = "What it does, when to use it, what it returns")
    String myTool(@ToolParam(description = "meaning + format + example") String input,
                  @ToolParam(description = "optional thing", required = false) Integer limit,
                  ToolContext ctx) {            // ctx is hidden from the model
        return "result";
    }
}

// 2. Use tools
chatClient.prompt()
    .system("rules for using tools")
    .user(question)
    .tools(myTools)                              // @Tool objects
    // .toolCallbacks(callbacks)                 // ToolCallback objects (MCP, programmatic)
    // .toolNames("beanName")                    // Function beans
    .toolContext(Map.of("userId", id))           // hidden data
    .call()
    .content();

// 3. Default tools for every request
ChatClient.builder(chatModel).defaultTools(myTools).build();

// 4. Manual loop
ToolCallingChatOptions.builder().toolCallbacks(cbs).internalToolExecutionEnabled(false).build();
chatResponse.hasToolCalls();
toolCallingManager.executeToolCalls(prompt, chatResponse);
```

**Remember in one minute:**

1. Tool = Java method + description. The LLM **requests**, your app **executes**.
2. Spring AI builds the JSON schema from the method signature and `@ToolParam`.
3. The flow is a **loop**: model → tool call → run Java → result → model → ... → final text.
4. The **model decides** which tool from names and descriptions. Good descriptions matter most.
5. Use `ToolContext` for hidden data like user id. Validate everything. Confirm risky actions.

---

*End of file.*
