# MCP Elicitation — Letting a Server Ask the User a Question Mid-Task

## 1. The simplest possible explanation

Normally, information flows in one direction: the user asks a question, the server does its job, and sends back an answer. **Elicitation breaks that one-way flow.** It lets an MCP **server**, while it's in the middle of doing something, **pause and ask the human a question** — and actually wait for a real answer before continuing.

Think of it like a human customer support agent on the phone. You say "I want to raise a support ticket." The agent doesn't just create a blank ticket — they ask you back: *"Sure, what priority should I give it, and what's a good number to reach you on?"* You answer, and **only then** do they finish creating the ticket. Elicitation is that exact back-and-forth, except the "agent" here is your MCP server's tool, and the "you" is whoever is using the AI application on the other end.

---

## 2. Why this exists — the real problem it solves

Without elicitation, a tool has only two options when it's missing information it actually needs:
1. **Guess** — fill in a default value, or let the model make something up. Bad, because it can be wrong (the model might invent a phone number or pick a random priority).
2. **Fail** — refuse to proceed and tell the model "missing required field." Annoying, because the model then has to ask the *user* in plain chat, the user types an answer back in plain text, and the whole tool call has to be retried from scratch with that info stuffed awkwardly into a new message.

Elicitation gives a **third, much cleaner option**: the tool itself can directly ask the person a specific, structured question — right in the middle of doing its job — and get back a clean, structured answer, instead of relying on the model to notice something is missing and relay a vague request back and forth in plain English.

---

## 3. Real-world scenarios where this actually matters

- **Booking a flight** — a travel-booking tool needs to know your preferred airline, but your profile doesn't have one saved. Instead of guessing or failing, the tool elicits: *"Which airline do you prefer?"* — and only continues the search once you answer.
- **Opening a support ticket** (exactly your example) — the tool knows *what* issue you have, but not how urgent it is or the best phone number to reach you. It elicits those two specific fields before actually creating the ticket.
- **Age or identity verification** — a tool that looks up age-restricted content asks, right there, "Please confirm you are 18 or older" before proceeding, rather than assuming.
- **Confirming a risky action** — before a tool deletes a file, cancels a subscription, or sends a large payment, it elicits an explicit "yes, I'm sure" from the person, rather than just trusting that the model's decision to call the tool was definitely what the human actually wanted.
- **VIP/premium customer checks** — a product-lookup tool elicits whether the requester is a VIP customer, and changes its behavior (more detail, priority routing) based on the answer, instead of needing that information passed in some other way beforehand.

The common thread in every one of these: **the tool needs a specific, small piece of information that only a human can reliably provide right now**, and elicitation is the mechanism built specifically for stopping and getting it, cleanly.

---

## 4. The two sides of elicitation

Elicitation always has two separate pieces of code, living in two separate applications:

- **The server side — the one that *asks*.** This is your `HelpDeskTools` class, running inside your MCP server. It decides *when* to ask and *what* to ask for.
- **The client side — the one that *answers*.** This is your `HelpDeskElicitationProvider` class, running inside your MCP client application. It decides *how* to respond when asked.

This is a genuinely important thing to notice: **the server never talks directly to a human.** It sends its question to the *client*, and it's the client's job to actually get an answer — whether that's by popping up a real UI form for a real human to fill in, or (as in your example) by simulating an answer programmatically.

---

## 5. Walking through the server-side code — `HelpDeskTools.createTicket(...)`

```java
@McpTool(name = "createTicket", description = "Create the Support Ticket")
String createTicket(@McpToolParam(description = "Details to create a Support ticket")
TicketRequest ticketRequest, McpSyncRequestContext ctx) {

    String priority = DEFAULT_PRIORITY;        // "MEDIUM" — the fallback if elicitation isn't possible
    String contactPhone = NO_PHONE_PROVIDED;    // "N/A" — same idea

    if (ctx.elicitEnabled()) {
        // STEP 1: Check first whether the connected client even SUPPORTS elicitation
        // at all. Not every MCP client implements it — some are simple, automated
        // clients with no human on the other end. Asking a question that can never
        // be answered would just hang or fail, so this check comes first.

        ctx.info("Asking you for a few extra details before opening this ticket...");

        // STEP 2: Actually send the elicitation request. The "spec" describes the
        // QUESTION (plain text message shown to the human), and TicketContactInfo.class
        // describes the SHAPE of the answer we expect back — turned automatically into
        // a JSON schema the client can use to build a real form.
        StructuredElicitResult<TicketContactInfo> elicitResult = ctx.elicit(
                spec -> spec.message("Before we open your support ticket, please choose a priority "
                        + "(LOW, MEDIUM, HIGH or URGENT) and share a contact phone number so our "
                        + "team can reach you."),
                TicketContactInfo.class);

        // STEP 3: Check how the human (or the client, on their behalf) responded.
        if (elicitResult.action() == McpSchema.ElicitResult.Action.ACCEPT
                && elicitResult.structuredContent() != null) {

            // They answered! Pull the actual values out of the structured response.
            TicketContactInfo info = elicitResult.structuredContent();
            if (info.priority() != null && !info.priority().isBlank()) {
                priority = info.priority();
            }
            if (info.contactPhone() != null && !info.contactPhone().isBlank()) {
                contactPhone = info.contactPhone();
            }
        } else {
            // They said no (DECLINE) or backed out (CANCEL) — don't crash, don't
            // retry forever. Just proceed gracefully with the safe defaults.
            ctx.info("No extra details provided. Opening the ticket with default priority '"
                    + DEFAULT_PRIORITY + "'.");
        }
    }

    // STEP 4: Whether we got real answers or fell back to defaults, the ticket
    // gets created EITHER WAY — elicitation politely enhances the result, it
    // doesn't block the tool from working at all if it's unavailable or declined.
    HelpDeskTicket savedTicket = service.createTicket(ticketRequest, priority, contactPhone);
    return "Ticket #" + savedTicket.getId() + " created successfully...";
}
```

### What `TicketContactInfo` is doing behind the scenes

```java
record TicketContactInfo(String priority, String contactPhone) {}
```

This plain Java record is the *shape* of the answer being requested. Spring AI automatically converts it into a JSON Schema that gets sent along with the elicitation request — so the client knows exactly what fields it needs to fill in (`priority`, `contactPhone`) and can build an actual form around that shape, rather than receiving a vague free-text question with no structure.

---

## 6. Walking through the client-side code — `HelpDeskElicitationProvider`

```java
@Component
public class HelpDeskElicitationProvider {

    @McpElicitation(clients = "eazybytes")
    public McpSchema.ElicitResult handleElicitationRequest(McpSchema.ElicitRequest request) {

        // This method is automatically called by Spring AI the moment an elicitation
        // request arrives from a server connection named "eazybytes". The "clients"
        // attribute links this handler to one specific connection — in a client
        // talking to multiple MCP servers, you could have a different @McpElicitation
        // handler for each one, each responding differently.

        LOGGER.info("Received MCP elicitation request from server: {}", request.message());
        // request.message() is the plain-text question the server asked —
        // exactly the string passed to spec.message(...) on the server side.

        // In a REAL application, this is where you'd show an actual UI — a popup, a
        // form, a chat prompt — and wait for a real human to type an actual answer.
        // Here, it's simulated: hardcoded values standing in for what a human would
        // have typed into a real form.
        Map<String, Object> userResponse = Map.of(
                "priority", "HIGH",
                "contactPhone", "+1-202-555-0185");

        // The keys here ("priority", "contactPhone") MUST match the field names of
        // the server's requested record (TicketContactInfo) exactly — this is how
        // Spring AI maps the raw answer back into that typed record on the server side.

        return McpSchema.ElicitResult.builder(McpSchema.ElicitResult.Action.ACCEPT)
                .content(userResponse)
                .build();
        // ACCEPT means "here's a real answer." The other two options are DECLINE
        // ("the user chose not to answer") and CANCEL ("the user backed out entirely")
        // — both of which the server-side code already handles gracefully, as seen above.
    }
}
```

### Why `clients = "eazybytes"` matters

This string has to match the **name you gave your MCP server connection** in your client's `application.yml` (the exact same naming pattern as the `abhay` connection name from the earlier stdio/Streamable HTTP guide). It's how Spring AI knows: *"when an elicitation request comes in from this specific server connection, route it to this specific handler method."* If you're connected to several servers, each one can have its own dedicated elicitation handler.

---

## 7. The complete, step-by-step request flow

This is the part worth slowing down on, because it crosses between two completely separate applications (the server and the client) and back again:

```
1. A user, through your AI chat application, types something like:
   "I'm having trouble logging in, can you open a support ticket for me?"

2. The model decides to call the createTicket MCP tool, running on your
   MCP SERVER (a separate application from the client/host).

3. Inside createTicket(...), the code checks ctx.elicitEnabled() — true,
   since the connected client declared it supports elicitation.

4. The server calls ctx.elicit(...), which sends an ElicitRequest message
   OVER THE MCP CONNECTION, back to the CLIENT application — this is a
   real network/protocol message, not a local method call. The server's
   execution of createTicket(...) PAUSES right here, waiting for a reply.

5. On the CLIENT side, Spring AI sees the incoming ElicitRequest, matches
   it to the "eazybytes" server connection, and automatically invokes
   HelpDeskElicitationProvider.handleElicitationRequest(request) — your
   client-side handler — to actually produce an answer.

6. In a real app, this is where the client would show a real form to a
   real human and wait for them to fill it in and submit. (In your demo
   code, this step is skipped — hardcoded values stand in for it.)

7. The handler returns an ElicitResult — ACCEPT, with the priority and
   contactPhone values filled in — which Spring AI sends back OVER THE
   SAME MCP CONNECTION to the waiting server.

8. Back on the SERVER, ctx.elicit(...) FINALLY returns, with the
   StructuredElicitResult now holding that real data. createTicket(...)
   resumes exactly where it paused, reads the priority and phone number
   out of the result, and finishes creating the actual ticket.

9. The tool returns its final text result ("Ticket #123 created...") back
   through the normal MCP tools/call response path, which flows back up
   to the model, which uses it to generate its final natural-language
   reply to the user.
```

**The key thing to really absorb:** step 4 through step 7 is a genuine **round trip across the network**, between two separate running applications, in the *middle* of a single tool call. The server doesn't just send a request and move on — it actually **blocks and waits** for the client's answer before it can continue, exactly like a phone call being put on hold while the agent checks something, rather than a one-shot fire-and-forget message.

---

## 8. A quick but important contrast — elicitation vs. "sampling"

Your `HelpDeskTools` class also has a `summarizeTickets` method using `ctx.sample(...)`, which is a **different** MCP feature, easy to confuse with elicitation since both involve the server reaching out to the client mid-task:

- **Elicitation** — the server asks the **human** a direct question, and needs a **human-provided** answer (a priority, a phone number, a yes/no confirmation).
- **Sampling** — the server asks the **client's own LLM** to generate text on the server's behalf (e.g. "please summarize this ticket data for me"), with no human directly answering anything — the client's AI model does the work instead.

Both follow the same "pause, ask the client, wait for the client's response, resume" shape — they're just asking two fundamentally different things of the client: one wants a human's input, the other wants the client's model's output.

---

## 9. Why the graceful fallback matters so much

Notice that **every single use of elicitation in your code checks for failure and has a safe default**:
- `ctx.elicitEnabled()` is checked *before* even attempting to elicit — some clients genuinely don't support this feature at all.
- `DECLINE` or `CANCEL` from the user is handled by quietly falling back to `DEFAULT_PRIORITY` and `NO_PHONE_PROVIDED`, not by throwing an error or refusing to create the ticket.

This is a deliberate, important design principle: **elicitation should enhance a tool's behavior, never be a hard requirement it can't function without.** A support ticket should still get created even if the user ignores the extra questions, or if the specific client they're using happens not to support elicitation at all — just with less detail than it would have had otherwise.

---

## 10. Recap

Elicitation is the mechanism that lets an MCP server **pause mid-task and directly ask the human using the application a specific, structured question**, instead of guessing, failing, or relying on the model to relay a vague request back and forth in plain chat. The server side (`ctx.elicit(...)`, inside a `@McpTool` method) decides what to ask and what shape the answer should take; the client side (`@McpElicitation`, matched to a specific server connection by name) decides how that question actually gets answered — in a real app, by showing an actual form to a real person. The whole thing is a genuine, blocking round trip across the connection — the server's tool execution truly pauses until the client replies — and good implementations, like yours, always treat the human's answer as an *enhancement*, falling back gracefully to sensible defaults if elicitation isn't supported, is declined, or is cancelled.
