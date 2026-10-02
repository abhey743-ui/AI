# MCP (Model Context Protocol) — What It Is, Why Not Just HTTP, and Real Microservices Architecture

## 1. What MCP actually is

**MCP (Model Context Protocol)** is an open standard, introduced by Anthropic, that defines a single, consistent way for an AI application to **discover and invoke external tools, read external data, and use reusable prompt templates** — regardless of which specific system those tools/data belong to, and regardless of which specific AI application is doing the calling.

The simplest honest description: **MCP is a protocol, not a product.** An "MCP server" is just a running program that exposes some of its capabilities (actions it can perform, data it can provide, prompt templates it offers) in a standardized, self-describing format that any MCP-compliant AI application can understand and use — without that application needing custom, hand-written integration code for that specific system.

A commonly used analogy: **MCP is to AI tool integration what USB-C is to physical device connectors.** Before USB-C, every device needed its own specific cable and port. After USB-C, any compliant device plugs into any compliant port, using the same physical and electrical standard. MCP does the same thing for "how does an AI model call a tool or read some data" — one standard interface, instead of one bespoke integration per tool.

---

## 2. MCP Host, MCP Client, and MCP Server — plain definitions first

Before anything else, it's worth pinning down these three terms clearly, in isolation, since they get reused constantly for the rest of this guide and are easy to blur together.

### MCP Host
**The actual AI application itself.** This is the program a human (or another system) is actually using — a chatbot, an IDE assistant, a custom internal orchestrator service, a voice assistant. The host is what runs the language model and decides, based on what the model outputs, that a tool needs to be called, a resource needs to be read, or a prompt template needs to be used. It's also the component responsible for policy — which servers it's allowed to connect to, and what it lets the model see or do. **Think of the host as "the AI app, as a whole."**

### MCP Client
**A connector living inside the host, dedicated to exactly one server.** A client doesn't make decisions and doesn't run any AI logic itself — it's pure plumbing. Its only job is: hold the connection to one specific server, ask that server what it offers (`tools/list`, `resources/list`, `prompts/list`), relay that list up to the host, and forward a `tools/call` down to the server whenever the host/model decides to invoke something. If a host talks to five different servers, it internally runs **five separate client instances**, one per server — never one client shared across several servers. **Think of the client as "the dedicated phone line to one specific server."**

### MCP Server
**An independent program that exposes capabilities — tools, resources, and/or prompts — in MCP's standardized format.** A server doesn't know or care which host or which specific AI application is calling it; it just answers standardized requests the same way regardless of who's asking. In practice, a server is usually a thin wrapper around something that already exists — a database, an internal REST API, a filesystem — translating MCP's standard requests into whatever that underlying system actually needs. **Think of the server as "the thing being connected to," describing itself so any compliant client can use it.**

### How the three relate, in one line each
- **Host** = the AI application, running the model, making decisions.
- **Client** = one dedicated connection, living inside the host, to one server.
- **Server** = the independent capability provider, reachable by any compliant client.

A useful shorthand: **one Host → many Clients → many Servers**, with each Client mapped to exactly one Server, and the Host being the only one actually "thinking."

---

## 3. Why not just use normal HTTP APIs directly? The real problem MCP solves

This is the question worth sitting with properly, because the honest answer isn't "HTTP is bad" — HTTP is still what's *underneath* MCP in most real deployments. The problem MCP solves is about **what the AI model needs in order to use an API correctly**, which plain HTTP doesn't provide on its own.

### 2.1 The N×M integration problem
Imagine you have 5 different AI applications (a chatbot, an IDE assistant, an internal agent, a mobile app, a voice assistant) and 10 different internal systems they each need to call (a CRM, a billing system, a ticketing system, and so on). Without a shared standard, you end up writing **5 × 10 = 50 separate, bespoke integrations** — each AI app needs its own custom code to know how to call each system's specific REST API, parse its specific response shape, and decide when to call it. Every new AI app added, or every new system added, multiplies that integration burden. MCP collapses this to **5 + 10 = 15** — each AI app implements the MCP *client* side once, and each system implements the MCP *server* side once, and any client can now talk to any server.

### 2.2 A plain HTTP API doesn't describe itself to a model
A REST endpoint like `POST /appointments` has a shape a human developer learns by reading documentation — parameter names, required fields, what the response looks like, what error codes mean. A language model has **no inherent way to know any of that** just by being told a URL exists. Someone still has to manually write a description of that endpoint, its parameters, and when to use it, and feed that description to the model as part of its tools — and that manual description-writing step has to be redone, by hand, for every single endpoint, in every single application that wants to let a model use it.

MCP standardizes **exactly that description** into a protocol-level concept: a server responds to a `tools/list` request with a structured list of its tools, each one self-describing — a name, a natural-language description of what it does and when to use it, and a JSON Schema defining its exact input shape. **The model-facing description becomes part of the server itself**, written once by whoever builds that server, rather than re-written by hand inside every single AI application that wants to use it.

### 2.3 Dynamic discovery vs. hardcoded integration
With a plain HTTP integration, an AI application's set of available "tools" is whatever its developers manually coded and hardcoded into it — adding a new capability means a code change and a redeployment of the *AI application itself*. With MCP, an AI application can connect to a server at runtime and call `tools/list` to discover what that server currently offers — if the server's capabilities change (a new tool added, an old one removed), **every connected client sees the update automatically**, with zero change needed to the AI application's own code.

### 2.4 Reusability across completely different AI applications
A REST API integration written specifically for, say, a custom internal chatbot is of no use at all to a different AI application (an IDE assistant, say) — someone has to write an entirely separate integration for that second application too, even though it's calling the exact same underlying system. An MCP server, once built, is usable by **any** MCP-compliant host — Claude, an IDE, a custom agent, a different company's product entirely — with zero additional integration work on the server side.

### 2.5 Standardized primitives beyond just "calling a function"
Plain HTTP gives you requests and responses — nothing more. MCP defines three distinct, purpose-built primitives on top of that: **Tools** (actions the model can decide to invoke), **Resources** (read-only context data the *application*, not the model, decides to pull in), and **Prompts** (reusable prompt templates a *user* can select) — each with a different, deliberate authorization/control boundary (§5), which a raw HTTP API has no standard way to express at all.

**The honest summary:** MCP doesn't replace HTTP — a huge share of real MCP servers are thin wrappers that *call* an existing HTTP API internally once a tool is invoked. What MCP actually replaces is the **ad-hoc, hand-written, one-off layer of "how does an AI model know this tool exists, what it does, and how to call it correctly"** that you'd otherwise have to build fresh for every API, for every AI application, every single time.

---

## 4. The architecture diagram — Host, Client, and Server, visually

MCP's architecture has three distinct participants, and understanding the separation between them is the key to understanding why "the client" is centralized while "the servers" are distributed:

```
┌──────────────────────────────────────────────┐
│                  MCP HOST                      │
│   (the actual AI application — a chatbot,      │
│    an IDE, an internal orchestrator service)    │
│                                                 │
│   Runs the LLM. Decides when to call a tool.    │
│                                                 │
│   ┌───────────┐  ┌───────────┐  ┌───────────┐ │
│   │MCP Client A│  │MCP Client B│  │MCP Client C│ │
│   └─────┬─────┘  └─────┬─────┘  └─────┬─────┘ │
└─────────┼──────────────┼──────────────┼───────┘
          │              │              │
          ▼              ▼              ▼
   ┌─────────────┐┌─────────────┐┌─────────────┐
   │ MCP Server 1 ││ MCP Server 2 ││ MCP Server 3 │
   │ (e.g. Patient││ (e.g. Billing││ (e.g. Pharmacy│
   │  Records svc)││   service)   ││   service)   │
   └─────────────┘└─────────────┘└─────────────┘
```

- **MCP Host** — the actual AI-powered application. It's the thing running the language model, and it's the thing deciding, based on the model's output, that a tool call needs to happen. The host is also the component that enforces consent and access-control policy — it decides *which* servers it's willing to connect to, and can mediate what the model is allowed to see or do.
- **MCP Client** — a component living *inside* the host, maintaining a **dedicated, one-to-one connection to exactly one server**. The host creates one new client instance for every server it connects to — connect to 5 servers, and the host is running 5 client instances internally, each one scoped to its own server connection. The client's job is purely plumbing: send `tools/list` to discover what the server offers, relay that list up to the host/model, and send `tools/call` when the model decides to invoke something.
- **MCP Server** — an independent program exposing tools, resources, and/or prompts. A server knows nothing about which host or which model is calling it — it just responds to standardized JSON-RPC requests (`tools/list`, `tools/call`, etc.) the same way regardless of who's asking.

### 3.1 Why "one client per server" instead of one client juggling many servers

This separation exists specifically so that **each server connection is isolated** — a problem with one server (a crash, a slow response, a different protocol version) doesn't bleed into the client logic talking to a different server. It also cleanly maps to how capabilities get merged: the host collects the tool lists from every one of its clients, merges them into one combined set the model can see, and when the model picks a tool, the host routes that specific call through the one client connected to the server that actually owns it. The *client* layer is intentionally dumb and mechanical; all the interesting application logic (which servers to trust, how to merge/present their capabilities, how to enforce permissions) lives in the host, one layer up.

---

## 5. Why centralize "the client" architecturally, instead of every service talking to the model directly

This is the architectural decision your question is really getting at, and it's worth being explicit about, because it's exactly the pattern real enterprise systems use.

**The anti-pattern:** every microservice independently embeds its own LLM-calling logic, its own prompt engineering, its own API key, and talks to the model provider directly whenever it needs "AI" to do something. This duplicates model-calling logic across every service, scatters API keys and cost tracking across the whole system, makes consistent behavior (safety, logging, rate limiting) impossible to enforce centrally, and means every service has to independently stay current with model provider API changes.

**The MCP-centered pattern instead:** build **one centralized AI orchestrator** (the MCP Host) that's the *only* component actually talking to the language model. Every other microservice instead exposes its own capabilities as an **MCP server** — a thin layer describing what that service can do, in MCP's standard shape — and the central orchestrator's MCP clients connect out to each of them. The language model itself only ever exists in one place; every other service just answers standardized `tools/list`/`tools/call` requests, which is a far smaller, far more stable surface to maintain than "embed and maintain an LLM integration."

This gives you, architecturally:
- **One place to enforce safety, logging, auth, and rate limiting** — the orchestrator, not scattered across every microservice.
- **One place that holds the actual model API key/credentials** — not duplicated and separately secured in every service.
- **Independent evolution of each microservice's capabilities** — a service can add, remove, or change its exposed tools without the orchestrator (or any other service) needing a code change, since discovery happens dynamically via `tools/list`.
- **Reuse across multiple AI applications** — if the organization later builds a second AI application (a mobile assistant, say), it can reuse the exact same MCP servers that the first orchestrator already connects to, without any of the underlying microservices needing to change at all.

---

## 6. The three primitives — Tools, Resources, Prompts

| Primitive | Who decides to use it | What it's for | Example |
|---|---|---|---|
| **Tools** | The **model** — it autonomously decides, based on the conversation, when to call one | Actions/functions the model can invoke, each described by a name, a description, and a JSON Schema for its inputs | `book_appointment`, `get_patient_vitals`, `charge_invoice` |
| **Resources** | The **application** (the host), not the model — the host decides what context to load and when | Read-only data the application pulls in as context, not something the model autonomously triggers | A patient's medical record file, a hospital's current bed-availability report |
| **Prompts** | The **user** — selected explicitly, like a slash command or a template picker | Reusable, pre-written prompt templates exposed by the server for a user to invoke directly | A "discharge summary template" a doctor explicitly picks from a menu |

This three-way split isn't just organizational — it maps directly to **who is actually in control of each kind of action**, which matters enormously for safety and auditability: you generally want a human (or at least deterministic application logic) deciding what read-only context gets loaded, want the model empowered to autonomously decide when an action genuinely needs to happen, and want a user explicitly opting into a canned prompt template rather than it firing on its own.

---

## 7. Transport — how the client and server actually talk

MCP defines two transport mechanisms, and picking the right one is a real architectural decision for a microservices deployment:

- **stdio** — the client and server run as two processes on the *same machine*, communicating over standard input/output. Simple, zero network configuration, but inherently local-only — not usable for a server living in a different microservice, on a different machine, behind a network boundary.
- **Streamable HTTP** — the server is exposed over the network as an HTTP endpoint; the client sends JSON-RPC messages as HTTP POST requests, with the server able to stream responses back. This is the transport that matters for a real microservices architecture — it works with standard HTTP infrastructure (load balancers, API gateways, reverse proxies), scales horizontally (modern revisions of the spec are explicitly stateless, making it a natural fit for multiple instances behind a load balancer), and is the recommended choice whenever a server needs to be reachable remotely, by potentially many different clients. *(An earlier transport, HTTP+SSE, has been deprecated in favor of Streamable HTTP — new servers should target Streamable HTTP exclusively.)*

For a hospital's distributed microservices, **every MCP server exposed by a backend service uses Streamable HTTP** — stdio simply isn't an option once the client (the central orchestrator) and the server (an individual microservice) are different deployed services, likely on different machines or containers entirely.

---

## 8. Real-world enterprise pattern — putting a gateway in front

In a serious production deployment, you don't expose each microservice's MCP server directly to the internet or even directly to the orchestrator without any intermediary. The common professional pattern layers in:

- **An API gateway in front of the MCP servers** — handling authentication (commonly OAuth 2.1, which the MCP spec explicitly recommends for Streamable HTTP deployments), rate limiting, and routing, exactly like a gateway would for any other set of internal microservices.
- **An MCP registry/catalog** — a central, discoverable directory of which MCP servers exist, what they expose, and their ownership/SLA — so that as the number of internal MCP servers grows across an organization, there's one place to find and vet them, rather than tribal knowledge of "ask the billing team what their server's URL is."
- **Centralized observability** — logging every `tools/call` through the orchestrator (and/or the gateway) gives you a single, auditable trail of every action an AI system took across every underlying microservice — critical in a regulated domain like healthcare.

---

## 9. Full real-world example — a hospital management system

### 8.1 The microservices, each exposing its own MCP server

Imagine a hospital's backend is already built as microservices (a completely ordinary, pre-AI architecture):

- **Appointment Service** — manages doctor schedules and bookings.
- **Patient Records Service** — holds medical history, allergies, prescriptions.
- **Billing Service** — handles invoices, insurance claims, payments.
- **Pharmacy Service** — manages medication inventory and prescription fulfillment.
- **Lab Results Service** — stores and retrieves diagnostic test results.

Each of these services, which already exist as ordinary REST microservices for the hospital's own internal web/mobile apps, now **additionally** exposes a thin MCP server layer alongside its existing API — this MCP server doesn't replace the service's REST API, it *wraps* it, translating MCP's standardized `tools/call` requests into the service's already-existing internal API calls:

```
Appointment Service  → exposes tools: book_appointment, cancel_appointment, get_doctor_availability
Patient Records Svc  → exposes tools: get_patient_history, get_allergies
                       exposes resource: patient-record://{patientId}
Billing Service      → exposes tools: generate_invoice, check_insurance_coverage
Pharmacy Service      → exposes tools: check_medication_stock, fulfill_prescription
Lab Results Service   → exposes tools: get_lab_results, order_lab_test
```

### 8.2 The centralized AI orchestrator (the MCP Host)

One service — call it the **Hospital AI Assistant Orchestrator** — is the single place in the whole architecture that actually runs the language model. It maintains **five separate MCP clients**, one connected (via Streamable HTTP) to each of the five services' MCP servers above, sitting behind the organization's API gateway for authentication.

### 8.3 The end-to-end flow for a real request

A hospital staff member, through a chat interface backed by the orchestrator, types:

> *"Book Mrs. Sharma an appointment with Dr. Iyer next Tuesday afternoon, and check if her insurance covers a cardiology consult."*

```
1. Orchestrator (MCP Host) receives the message, passes it to the LLM along with
   the MERGED list of every tool discovered across all 5 connected MCP clients
   (collected earlier via tools/list on each connection).

2. The model reasons about the request and determines it needs to:
   a. Call get_doctor_availability (Appointment Service) to find an open Tuesday-
      afternoon slot for Dr. Iyer.
   b. Call check_insurance_coverage (Billing Service) for a cardiology consult,
      for this specific patient.

3. The orchestrator's Appointment-Service MCP client sends tools/call for
   get_doctor_availability. The Appointment Service's MCP server receives it,
   internally calls its own existing scheduling logic/database, and returns
   available slots.

4. In parallel (or next), the orchestrator's Billing-Service MCP client sends
   tools/call for check_insurance_coverage. The Billing Service's MCP server
   checks the patient's actual insurance record and returns coverage details.

5. Both tool results flow back up through their respective clients to the host,
   which feeds them back into the model as tool results.

6. The model, now informed by real, live data from two completely separate
   microservices, decides to call book_appointment (Appointment Service) with
   the confirmed slot.

7. The Appointment-Service MCP client sends that tools/call; the appointment
   is actually created in that service's real database.

8. The model generates a final natural-language summary for the staff member:
   "Booked Mrs. Sharma with Dr. Iyer, Tuesday 3:00 PM. Her insurance covers
    cardiology consults with a $20 copay."
```

Notice what **never** had to happen anywhere in this flow: the Appointment Service never needed to know anything about language models, prompt engineering, or the Billing Service's internals. The Billing Service never needed to know an AI system existed at all beyond responding to a standard `tools/call`. Each microservice kept doing exactly what it already did — the MCP layer is a thin, additive adapter on top, and the *only* place that actually understands "AI" is the one centralized orchestrator.

### 8.4 Why this specific architecture, for a hospital specifically

- **Auditability** — every tool call (booking an appointment, checking insurance, pulling patient history) flows through one orchestrator and can be logged centrally, which matters enormously for a regulated healthcare environment needing a clear audit trail of every AI-initiated action.
- **Least-privilege access per service** — the Patient-Records MCP server only exposes the specific tools/resources it chooses to (e.g. `get_patient_history`, not "run arbitrary SQL against our database") — each service controls its own exposed surface precisely, rather than the orchestrator having broad direct database access.
- **Independent ownership** — the team that owns the Pharmacy Service can add, change, or deprecate the tools their MCP server exposes without coordinating a redeploy of the central orchestrator — the orchestrator just sees an updated `tools/list` the next time it asks.
- **Reuse beyond one chat interface** — if the hospital later builds a separate mobile app for doctors with its own AI assistant, that second host can connect to the exact same five MCP servers, with zero changes needed on the microservices' side.

---

## 10. Direct HTTP integration vs. MCP — side by side

| | Direct, hand-written HTTP integration per AI app | MCP |
|---|---|---|
| **Model-facing description of what an endpoint does** | Manually written, once per AI application, per endpoint | Written once, by the server itself, discoverable by any client via `tools/list` |
| **Adding a new capability** | Requires a code change + redeploy of the AI application | Requires only a change to the server; connected clients discover it automatically |
| **Reuse across multiple AI applications** | None — each application's integration is bespoke | Full — any MCP-compliant host can use any MCP server |
| **Where the "AI-calling" logic lives** | Potentially duplicated across every service that wants AI features | Centralized in one host; every other service just answers a standard protocol |
| **Standardized read-only context vs. autonomous actions vs. user-triggered templates** | No standard distinction — it's all just "an API call" | Explicit, protocol-level separation: Resources, Tools, Prompts |
| **Scaling to many services × many AI apps** | N×M bespoke integrations | N + M — each side implements the protocol once |

---

## 11. Recap

MCP is a standard protocol — not a replacement for HTTP, but a standardized, self-describing, discoverable layer on top of (usually) HTTP-backed systems — specifically solving the problem of how an AI model finds out a tool exists, understands what it does, and invokes it correctly, without that description being manually, repeatedly hand-written for every tool in every AI application. Its architecture deliberately separates a **host** (the one place actually running the model and deciding when to act), many **clients** (one dedicated connection per server, living inside the host), and many **servers** (each one an independent, reusable, self-describing capability provider) — and real enterprise systems lean into exactly this separation by centralizing the "AI-aware" orchestrator in one place while letting every underlying microservice stay AI-agnostic, exposing only a thin MCP server wrapper around capabilities it already has. The hospital example above is the general pattern any real microservices organization follows: existing services stay exactly as they are, each gains a thin MCP server adapter, and one centralized orchestrator is the only component that ever actually talks to the language model.
