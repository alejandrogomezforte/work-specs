# Spruce Communication and the Archive Sweeper Solution (MLID-2167)

> Analysis document derived from a design conversation. Summarizes how the Spruce integration works, which webhooks we receive, the race condition that motivated MLID-2167, and why redesigning the logical sequence does not eliminate the need for the sweeper.

---

## 1. Do we receive a webhook from Spruce when a message is archived?

**No.** Spruce does not send a webhook when a conversation is archived. The webhooks we receive from Spruce are inbound message / conversation-item events (the ones that trigger *our* archive flow). There is no "conversation archived" callback that we subscribe to or handle.

Confirmation that the archive happened comes from the **synchronous HTTP response** to our own `PATCH` against Spruce's `updateConversation` endpoint — not from a callback.

### What happens when the sweeper asks Spruce to archive a text?

`spruceArchiveSweeper.ts` → `noteAndArchiveConversation()` → `archiveSpruceConversation()` (`apps/web/services/integrations/spruce.ts:781`):

1. **Atomic claim** — `findOneAndUpdate` keyed on `spruce.archivedAt: { $exists: false }`. Stamps `archivedAt` upfront so a racing late webhook can't double-process.
2. **Post internal note** — `postSpruceInternalNote(conversationId, 'Processed by LISA')`.
3. **Wait 2 s** so the note lands before the conversation is closed.
4. **Archive call** — `api.updateConversation({ archived: true }, { conversationId })`. Spruce replies inline with the updated conversation object (`response.data.conversation`). That HTTP 2xx **is** our archive confirmation.
5. **On success** → `noteAndArchiveConversation` returns `true`, `archivedAt` stays stamped, sweeper moves on.
6. **On failure (throw)** → sweeper `$unset`s `archivedAt` so the next 5-min tick retries.

The loop with Spruce is fully request/response. The `archivedAt` guard in the inbound-webhook handler exists to make **Spruce redelivering the original inbound message** idempotent — not to consume any "archived" event.

---

## 2. What webhooks do we receive from Spruce, and what triggers them?

The subscription is registered without an event filter (`POST /api/admin/integrations/spruce/subscription` → `createWebhookEndpoint({ name, url })`), so **Spruce delivers every event type to a single endpoint**: `POST /api/webhooks/spruce_conversations`. Our handler `handleSpruceConversation` (`apps/web/services/webhooks/handlers/spruce_conversation.ts:209-391`) then discriminates on `body.type` and only acts on four event types — the rest are silently dropped (`else { return; }`).

### Events we actually process

| `type` | Triggered by (Spruce side) | What we do |
|---|---|---|
| `conversation.created` | A new Spruce conversation is opened (e.g., first SMS exchange with a number) | Upsert into our `Conversation` collection keyed on `conversation_id`, store full payload in `data` |
| `conversation.updated` | Any change to a conversation (assignee, tags, archive flag flipped, etc.) | Same upsert as above — payload overwrites `data` |
| `conversationItem.created` | A new item is added to a conversation: inbound SMS, outbound SMS we sent, inbound fax, internal note, etc. | Upsert into `ConversationItem`. **Two side-effect branches:** ① if `direction='outbound'` + `requestId` → link `spruce.conversationId`/`appUrl` onto the originating `Message`, then run the SMS archive flow; ② if `direction='inbound'` with document attachments → schedule `spruceIntakeJob` to ingest the fax/PDF |
| `conversationItem.updated` | Edits to an existing item (rare; e.g., status changes on outbound delivery) | Same as `created` — runs through the same upsert and side-effect branches |

### Events we do **NOT** process

Anything else Spruce emits (delivery receipts, contact/endpoint changes, team membership, **archive-state changes**, etc.) hits our endpoint, fails the `type` check, and is dropped on the floor. **There is no inbound signal that tells us "Spruce archived this conversation"** — that's why MLID-2167 had to rely on the synchronous response from `updateConversation({ archived: true })` plus the sweeper as a safety net.

### Bonus: legacy endpoint

`apps/web/pages/api/webhooks/spruce.ts` exists but is effectively dead code — it verifies the signature, then returns a hard-coded `{ success: true, jobId: '123' }` without dispatching anything. All real traffic goes through `spruce_conversations.ts`.

---

## 3. How Spruce communication works — high level

Imagine **Spruce is like WhatsApp but for clinics**: it's the tool the team uses to send and receive text messages (SMS) with patients. Our app (LISA) talks to Spruce all day to automate tasks that a person used to have to do by hand.

The communication is **two-way**, like a phone conversation.

### 3.1 We talk to Spruce (outbound requests)

When LISA wants Spruce to do something, it sends an HTTP request — like making a phone call:

- **"Send this SMS to the patient"** → when an appointment is coming up, LISA asks Spruce to send the reminder.
- **"Drop this internal note in the conversation"** → so the team sees that LISA already processed the message (the famous *"Processed by LISA"* note).
- **"Archive this conversation"** → so it disappears from the team's inbox and they don't have to touch it.

In each case Spruce responds immediately with *"done"* or *"failed because X"*. That response is our only confirmation — there is no second messenger that comes later to confirm.

### 3.2 Spruce talks to us (webhooks)

This is where the **webhooks** come in. A webhook is basically a doorbell: every time something happens in Spruce, Spruce rings LISA's doorbell by sending a small packet of information.

We have **a single doorbell installed** (`/api/webhooks/spruce_conversations`), and Spruce notifies us about **everything** that happens. But LISA only pays attention to four kinds of notifications (see table in section 2).

### 3.3 The "new message" notification does all the heavy lifting

When the "new message" notification arrives, LISA looks at what kind it is:

- **Is it an SMS we sent?** → Spruce hands us back the ID of the conversation that was just created. We save it on our message and then ask Spruce to drop the *"Processed by LISA"* note and archive the conversation. All automatic.
- **Is it a fax or document that came from the patient?** → We kick off a background job (`spruceIntakeJob`) that downloads the PDF, processes it, and lands it in the patient's record.

### 3.4 What Spruce does **NOT** notify us about

Spruce does not notify us when a conversation is archived. That is: when we tell it *"archive this"*, the immediate response (*"OK, archived"*) is the only signal we get. There is no second messenger that comes later to confirm.

That's why in MLID-2167 we had to add the **sweeper** (a process that every 5 minutes checks if there are messages that didn't get archived) — because if the immediate response is lost for any reason, we have no way to find out via webhook. We have to ask Spruce ourselves.

### One-line summary

**LISA talks to Spruce over HTTP (requests with immediate responses), and Spruce talks to LISA over webhooks (notifications that arrive on their own when things happen)** — but the webhooks only cover new messages and conversations, not events like "it was archived."

---

## 4. The original problem: the MLID-2167 race condition

### 4.1 The "happy path" (how it *should* work)

When LISA sends an SMS reminder to a patient, the following happens in order:

1. **LISA → Spruce:** *"Send this SMS. Take this `requestId` so we can identify it later."*
2. **LISA saves to its database:** *"Message sent, requestId = ABC123."*
3. **Spruce sends the SMS to the patient.**
4. **Spruce → LISA (webhook):** *"Hey, I just created conversation XYZ and the `requestId` is ABC123."*
5. **LISA receives the webhook**, looks up the message with `requestId = ABC123` in its database, stamps `conversationId = XYZ` on it, and then tells Spruce: *"Drop the 'Processed by LISA' note and archive conversation XYZ."*

All beautiful. The patient gets the SMS, the team doesn't see the message in their inbox because it's already archived, and nobody has to do anything manually.

### 4.2 The problem: steps 2 and 4 race each other

Here's the trap. Imagine that step 2 (*"LISA saves to its database"*) **is slow**. Maybe the database is busy, maybe there's an external call beforehand, maybe the save was "deferred" for later.

And it turns out Spruce is **really fast**. Sometimes the webhook from step 4 arrives **before** LISA finishes step 2.

So what actually happens is:

1. LISA → Spruce: *"send the SMS, requestId = ABC123."*
2. (LISA starts saving, but hasn't finished yet)
3. Spruce sends the SMS.
4. **Spruce → LISA (webhook): *"requestId = ABC123, conversationId = XYZ."***
5. LISA looks up a message with `requestId = ABC123` in its database…
6. **Finds nothing.** Because step 2 hasn't finished yet.
7. The webhook leaves without doing anything. The conversation never gets archived.
8. (A moment later) LISA finally saves the message. But nobody is going to come look for it anymore — the webhook already passed.

**Result:** the conversation stays unarchived forever. The team sees it in their inbox, assumes they have to do something, wastes time. Multiply that by hundreds of messages a day and you have an inbox flooded with junk.

This is a classic **race condition**: two things running in parallel where the order of arrival is not guaranteed, and the system only works if they arrive in the "correct" order.

### 4.3 The first fix: save immediately (commit `a5d014cb`)

The first solution was obvious: **save the `requestId` to the database before sending anything to Spruce.** No more race because by the time the webhook arrives, the message always already exists.

This closes **about 99% of the cases**. But three smaller holes remained.

### 4.4 The remaining holes

**Hole 1: The microwindow between "Spruce returned OK" and "I saved to my DB"**

Even saving "before," there is a millisecond blink where Spruce has already responded but the `INSERT` in Mongo hasn't finished yet. If the webhook lands in that blink, same problem.

**Hole 2: Spruce redelivers the same webhook**

Spruce, like any well-behaved system, sometimes sends the same webhook twice (in case the first one got lost). Without protection, LISA would process it twice — it would post **two** *"Processed by LISA"* notes and try to archive an already-archived conversation twice.

**Hole 3: Something simply crashes**

- The LISA process crashes right after sending the SMS.
- Mongo has a transient error right when we save.
- Spruce has a delivery failure and the webhook never arrives.

In any of these cases, **there is no way to recover**. The message is orphaned forever.

### 4.5 Why we need the sweeper

The webhook is the "real-time" path — it works almost always, it's fast, and it should remain the primary one. **But "almost always" is not enough** for a healthcare system where a dirty inbox translates to wasted team time and missed messages.

The **archive sweeper** is the **safety net**. It's a background job that runs **every 5 minutes** and asks Mongo a very simple question:

> *"Are there outbound messages from the last hour that ALREADY have a `conversationId` (meaning the webhook did arrive and identified them) but do NOT have an archived stamp? If so, archive them."*

With that we cover the three holes:

- **Hole 1 (microwindow):** if the webhook is lost due to timing, the sweeper picks it up on the next pass (5 minutes worst case).
- **Hole 2 (redelivery):** we now stamp `spruce.archivedAt` when we archive. If the webhook comes twice, the second attempt sees the stamp and bails out. The same protection ensures the sweeper and a late webhook can never collide.
- **Hole 3 (crashes):** the sweeper does not depend on the webhook or the original process. Even if LISA goes down, when it comes back up the sweeper will scan the recent window and fix anything that was left pending.

### 4.6 The philosophy of the fix

The pattern is very classic in distributed systems:

- **Real-time primary path (webhook):** fast, covers 99%.
- **Background reconciler (sweeper):** slow, covers the remaining 1%.
- **"Already done" stamp (`archivedAt`):** prevents the two paths from doing the work twice.

It's not that the webhook is "bad" — it remains the primary path. The sweeper exists because in production there will **always** be some edge case the happy path doesn't cover, and in healthcare we cannot afford "well, if it gets lost, tough luck." The sweeper guarantees that **eventually everything gets archived**, no matter what breaks along the way.

---

## 5. What if we redesign the logical sequence? Do we still need the sweeper?

### Redesign proposal

1. LISA creates a Message record with `requestId`, representing a potential outbound message. At this point a record already exists in the DB.
2. LISA makes the request to Spruce to send the message with the existing `requestId`. If Spruce responds "OK" then LISA updates the record as sent.
3. Spruce sends the SMS.
4. Spruce executes the webhook responding with the conversation ID. By then the record must already exist, because we are using `async/await` in the code — so steps 2, 3, and 4 cannot have run until the DB save succeeded.
5. We continue with the rest of the steps.

### Can we do this? Yes — and in fact we already did

This proposal is **exactly** what the first commit of the ticket implemented (`a5d014cb` — *"persist requestId immediately to prevent archive race condition"*). Before that commit the code saved the message **after** calling Spruce. Changing it to save **before** closes the main race — about 99% of cases.

And you're right that `async/await` guarantees the order: if `await message.save()` runs before `await spruce.send()`, the record **definitely** exists by the time the webhook arrives.

**But the sweeper is still necessary.** And the reason is not the race condition. It's something else.

### The problems the redesign does **NOT** fix

#### 1. Spruce redelivers webhooks on its own

This **is not a race condition on our side** — it's Spruce behavior. When Spruce sends a webhook and for any reason doesn't get a quick `200 OK` (timeout, network glitch, whatever), it **sends it again**. Sometimes twice, sometimes more.

Without protection, LISA would process the same webhook multiple times:
- It would post **two** *"Processed by LISA"* notes in the conversation.
- It would try to archive twice (the second one would fail ugly).

No matter how perfect your save ordering is, **this still happens**. That's why we had to add the `spruce.archivedAt` stamp as a *"this is already done, don't do it again"* flag.

#### 2. The webhook simply never arrives

This is the big one. There are several scenarios where the webhook **does not arrive at all**:

- **Spruce has a delivery failure** — their webhook system goes down, their queue gets stuck, there's a network problem between them and us.
- **LISA is down** when the webhook lands — Spruce retries a couple of times and gives up.
- **The LISA process crashes mid-way** through processing the webhook — we receive the notification but die before archiving.
- **Mongo has a transient error** right when we go to archive.

In **all of these cases**, your perfect redesign doesn't change anything. The message is well saved, everything is well ordered, but **nobody ever told us that it was time to archive**. The conversation stays hanging in the team's inbox forever.

#### 3. The microwindow still exists (though smaller)

Even with your ordering, there is a millisecond blink between Spruce responding *"OK sent"* and LISA finishing updating the message to "sent state." If the webhook arrives right in that blink, it could see the message in an intermediate state. It's a tiny risk, but it exists.

### Why a sweeper is the only real solution

Problems 1 and 3 we can mitigate with code (the `archivedAt` flag and saving correctly). **But problem 2 is structural:** *"what happens when an external system doesn't notify us about something we expected."*

There are only two ways to solve that:

**Option A — Trust and pray:** assume the webhook always arrives. When it doesn't, we lose the message and someone has to clean it up by hand. Bad option in healthcare.

**Option B — Periodic reconciliation (the sweeper):** every so often, **we** go and ask our own database: *"is there any message that was left unarchived?"* If so, we fix it. We don't depend on being told.

This is a very common pattern in distributed systems and it's called **eventual consistency** with a *reconciler*. The philosophy is:

> *"Webhooks are optimistic — they work almost always and they're fast. Reconciliation is pessimistic — it assumes something was lost and goes looking for it."*

### Final summary

| Problem | Does the redesign fix it? |
|---|---|
| Race condition between `save()` and webhook | **Yes** ✅ (this is what we did in `a5d014cb`) |
| Spruce redelivers the same webhook | **No** — we need the `archivedAt` flag |
| The webhook never arrives (crashes, network failures, Spruce-side) | **No** — we need the sweeper |
| Millisecond microwindow | **Almost** — the `archivedAt` flag closes it completely |

**The golden rule:** *any system that depends on an external webhook needs a reconciliation mechanism, because sooner or later the webhook gets lost.* It's not a matter of **if** it happens, it's a matter of **when**.

The proposed redesign is the correct foundation — and it is in fact already in the code. The sweeper is the safety net that protects us from what we **don't control** (Spruce, the network, our own crashes).
