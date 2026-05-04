# Postmortem: Unarchived Spruce Messages

**Date**: 2026-04-22
**Incident**: Spruce SMS reminder sent but conversation not archived
**Affected Message**: `69e8c652752417d8e83f2e4c`
**Affected Appointment**: `69e88aaf679cad525f1c0984`
**Patient**: TEST SIX (Bangor, Maine)

---

## What Happened

On 2026-04-22, the daily `appointmentReminder` job created a reminder SMS for patient TEST SIX's appointment on 2026-04-24. The message was successfully sent via Spruce at 07:16 AM EST. However, the Spruce conversation was never archived: the message is missing `spruce.conversationId`, `spruce.appUrl`, and `spruce.archivedAt`.

Other messages sent in the same batch (e.g., `69e8c652752417d8e83f2e70` for Augusta, ME) were archived correctly within ~14 minutes of being sent.

---

## Event Timeline Reference

The table below shows the full lifecycle of an SMS message, from appointment creation to archival. Each row is a discrete step, with the file, trigger type, and function responsible.

| Step | Event | Type | File | Function |
|------|-------|------|------|----------|
| 1 | Appointment created/updated in WeInfuse | Webhook | `pages/api/webhooks/looker_appointments.ts` | `handleLookerAppointments()` |
| 2 | Appointment upserted in MongoDB | Webhook handler | `services/webhooks/handlers/util/webhookUpsertAppointments.ts` | `webhookUpsertAppointments()` |
| 3 | Reminder message created (`status: READY`) | Job | `services/jobs/definitions/appointmentReminder.ts` | `appointmentReminder()` |
| 3a | (queries appointments 2-3 days out, Active/Rescheduled) | Job helper | `services/jobs/definitions/appointmentReminder/pendingAppointments.ts` | `pendingAppointments()` |
| 3b | (creates the Message document) | Service | `services/messages/messageService.ts` | `setAppointmentReminderMessage()` |
| 4 | Message picked up and sent via Spruce API | Job | `services/jobs/definitions/sendMessageJob/sendMessageJob.ts` | `sendMessageJob()` |
| 4a | (validates patient/location, calls Spruce API) | Service | `services/messages/messageService.ts` | `sendSmsMessageByAppointment()` |
| 4b | (posts SMS via Spruce endpoint, returns `requestId`) | Service | `services/messages/messageService.ts` | `sendSpruceSms()` |
| 4c | (saves `requestId` to MongoDB immediately — MLID-2167 fix) | Job | `services/jobs/definitions/sendMessageJob/sendMessageJob.ts` | `sendMessageJob()` (line 386-399) |
| 5 | Spruce fires outbound webhook with `conversationId` | Webhook | `pages/api/webhooks/spruce_conversation.ts` | `handleSpruceConversation()` |
| 5a | (sets `spruce.conversationId` + `spruce.appUrl` on Message) | Webhook handler | `services/webhooks/handlers/spruce_conversation.ts` | `handleSpruceConversation()` (line 322) |
| 5b | (posts "Processed by LISA" note + archives conversation) | Webhook handler | `services/webhooks/handlers/spruce_conversation.ts` | `archiveSmsConversation()` → `noteAndArchiveConversation()` |
| 5c | (stamps `spruce.archivedAt` on Message) | Webhook handler | `services/webhooks/handlers/spruce_conversation.ts` | `archiveSmsConversation()` (line 180) |
| 6 | Safety-net sweep for missed archivals | Job | `services/jobs/definitions/spruceArchiveSweeper.ts` | `runSpruceArchiveSweeper()` |

**Note**: All file paths above are relative to `apps/web/`.

### Where the Bangor message broke

Steps 1-4c completed successfully. **Step 5 failed**: the Spruce webhook either arrived before step 4c saved the `requestId` (race condition), or was never delivered. Because `conversationId` was never set, step 6 (sweeper) also cannot find the message. The message is permanently orphaned.

```
Step 1  [OK]  Appointment created via looker_appointments webhook
Step 2  [OK]  Appointment upserted in appointments collection
Step 3  [OK]  appointmentReminder job created Message { status: READY }
Step 4  [OK]  sendMessageJob sent SMS, Spruce returned requestId, saved to MongoDB
Step 5  [FAIL] Spruce webhook: Message.updateMany({ requestId }) matched 0 documents
               → conversationId never set
               → archival never triggered
Step 6  [SKIP] spruceArchiveSweeper: query requires conversationId to exist → message excluded
```

---

## Lifecycle Diagram

```
                         EXTERNAL SYSTEMS
    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
    │  WeInfuse    │    │   JotForm    │    │  Spruce API  │
    │    EHR       │    │              │    │              │
    └──────┬───────┘    └──────┬───────┘    └──────┬───────┘
           │                   │                   │
           │                   │                   │
═══════════╪═══════════════════╪═══════════════════╪═══════════════════════
           │                   │                   │
    STEP 1-2: APPOINTMENT INGESTION                │
           │                   │                   │
           ▼                   │                   │
    ┌──────────────┐           │                   │
    │  [WEBHOOK]   │           │                   │
    │  looker_     │           │                   │
    │  appointments│           │                   │
    └──────┬───────┘           │                   │
           │                   │                   │
           ▼                   │                   │
    ┌──────────────┐           │                   │
    │ webhookUpsert│           │                   │
    │ Appointments │           │                   │
    └──────┬───────┘           │                   │
           │                   │                   │
           ▼                   │                   │
    ┌──────────────┐           │                   │
    │ appointments │           │                   │
    │ collection   │           │                   │
    └──────┬───────┘           │                   │
           │                   │                   │
═══════════╪═══════════════════╪═══════════════════╪═══════════════════════
           │                   │                   │
    STEP 3: MESSAGE CREATION (3 independent paths) │
           │                   │                   │
           │  ┌────────────────┼───────────────┐   │
           │  │                │               │   │
           ▼  │                ▼               ▼   │
    ┌─────────┴──┐    ┌──────────────┐  ┌─────────┴──┐
    │   [JOB]    │    │    [JOB]     │  │ [WEBHOOK]  │
    │ appointment│    │  jotForm     │  │  looker_   │
    │  Reminder  │    │  LoadSubmit  │  │ appointments│
    │            │    │              │  │ (→Complete) │
    │ daily 9AM  │    │  on submit   │  │            │
    │ 2-3d ahead │    │              │  │            │
    └──────┬─────┘    └──────┬───────┘  └──────┬─────┘
           │                 │                 │
           ▼                 ▼                 ▼
    ┌────────────┐   ┌──────────────┐   ┌────────────┐
    │setAppt     │   │setWhatTo     │   │setSymptom  │
    │Reminder    │   │ExpectMessage │   │Message     │
    │Message     │   │              │   │            │
    └──────┬─────┘   └──────┬───────┘   └──────┬─────┘
           │                │                  │
           │  source:       │  source:         │  source:
           │  reminder      │  what-to-expect  │  symptom
           │                │                  │
           └────────────────┼──────────────────┘
                            │
                            ▼
                     ┌──────────────┐
                     │   messages   │
                     │  collection  │
                     │              │
                     │ status: READY│
                     └──────┬───────┘
                            │
═══════════════════════════╪═══════════════════════════════════════════
                            │
    STEP 4: MESSAGE SENDING │
                            │
                            ▼
                     ┌──────────────┐
                     │    [JOB]     │
                     │ sendMessage  │
                     │    Job       │
                     │              │
                     │ periodic     │
                     │ 9AM-3PM EST  │
                     └──────┬───────┘
                            │
                            ▼
                     ┌──────────────┐          ┌──────────────┐
                     │sendSpruceSms │─────────►│  Spruce API  │
                     │              │  POST    │              │
                     │              │◄─────────│  returns     │
                     └──────┬───────┘requestId │  requestId   │
                            │                  └──────┬───────┘
                            ▼                         │
                     ┌──────────────┐                 │
                     │ message.save │                  │
                     │ (MLID-2167)  │                  │
                     │              │                  │
                     │ persists     │                  │
                     │ requestId    │◄── RACE ────────►│
                     │ immediately  │   CONDITION      │
                     └──────┬───────┘   ZONE           │
                            │                          │
                            ▼                          │
                     ┌──────────────┐                  │
                     │   messages   │                  │
                     │  collection  │                  │
                     │              │                  │
                     │ status: SENT │                  │
                     │ + requestId  │                  │
                     └──────────────┘                  │
                                                       │
═══════════════════════════════════════════════════════╪════════════
                                                       │
    STEP 5: SPRUCE WEBHOOK → ARCHIVE                   │
                                                       │
                                                       ▼
                                                ┌──────────────┐
                                                │  [WEBHOOK]   │
                                                │  spruce_     │
                                                │  conversation│
                                                │              │
                                                │  conversationItem
                                                │  .created    │
                                                └──────┬───────┘
                                                       │
                                                       ▼
                                                ┌──────────────┐
                                                │ updateMany   │
                                                │ by requestId │
                                                │              │
                                                │ sets:        │
                                                │ conversationId
                                                │ appUrl       │
                                                └──────┬───────┘
                                                       │
                                                       ▼
                                                ┌──────────────┐
                                                │archiveSms    │
                                                │Conversation  │
                                                │              │
                                                │ 1. post note │
                                                │ 2. archive   │
                                                │ 3. stamp     │
                                                │  archivedAt  │
                                                └──────┬───────┘
                                                       │
                                                       ▼
                                                ┌──────────────┐
                                                │   messages   │
                                                │  collection  │
                                                │              │
                                                │ + conversationId
                                                │ + appUrl     │
                                                │ + archivedAt │
                                                └──────────────┘

═══════════════════════════════════════════════════════════════════

    STEP 6: SAFETY NET

                     ┌──────────────┐
                     │    [JOB]     │
                     │spruceArchive │       Finds messages with:
                     │  Sweeper     │       - conversationId EXISTS
                     │              │       - archivedAt MISSING
                     │ every ~15min │       - sent within last hour
                     │ 1hr lookback │
                     └──────────────┘
```

### Race Condition Detail

```
    HAPPY PATH                              FAILURE PATH (Bangor message)
    ══════════                              ══════════════════════════════

    sendMessageJob        Spruce            sendMessageJob        Spruce
         │                  │                    │                  │
         │── POST SMS ─────►│                    │── POST SMS ─────►│
         │                  │                    │                  │
         │◄─ { requestId } ─│                    │◄─ { requestId } ─│
         │                  │                    │                  │
         │── save() ──► DB  │                    │                  │
         │   requestId      │                    │  ┌───────────────┤
         │   NOW IN DB      │                    │  │ webhook fires │
         │                  │                    │  │ BEFORE save   │
         │  ┌───────────────┤                    │  ▼               │
         │  │ webhook fires │                    │  updateMany()    │
         │  ▼               │                    │  by requestId    │
         │  updateMany()    │                    │  → 0 MATCHES     │
         │  by requestId    │                    │  (not in DB yet) │
         │  → 1 MATCH       │                    │                  │
         │  sets:           │                    │── save() ──► DB  │
         │  conversationId  │                    │   requestId      │
         │  appUrl          │                    │   NOW IN DB      │
         │                  │                    │   (too late)     │
         │  archive         │                    │                  │
         │  conversation    │                    │  conversationId  │
         │                  │                    │  NEVER SET       │
         │  stamp           │                    │                  │
         │  archivedAt      │                    │  sweeper SKIPS   │
         │                  │                    │  (no convId)     │
         ▼                  ▼                    ▼                  ▼
      ARCHIVED                                ORPHANED FOREVER
```

---

## How the SMS Lifecycle Works

There are three independent events that create messages in the `messages` collection. Each creates a document with `status: READY`, which the `sendMessageJob` later picks up and sends via Spruce.

### Event 1: Appointment Reminder (source: `reminder`)

**Trigger**: The `appointmentReminder` background job runs daily at 9 AM EST.

1. Job queries the `appointments` collection for appointments **2-3 days in the future** with `status: Active` or `Rescheduled` and no existing `reminder` flag.
2. For each matching appointment, calls `setAppointmentReminderMessage(appointmentId)`.
3. `setAppointmentReminderMessage` creates a `Message` document:
   - `type: sms`, `source: reminder`, `status: READY`
   - `dateToSend`: today at 9:00 AM EST (13:00 UTC)
   - `appointmentId`: reference to the appointment

### Event 2: What-to-Expect Confirmation (source: `what-to-expect`)

**Trigger**: Patient submits a JotForm intake form.

1. The `jotFormLoadSubmission` job processes the intake.
2. Calls `setWhatToExpectMessage(intakeId)` which creates a `Message` document:
   - `type: sms`, `source: what-to-expect`, `status: READY`
   - `immediately: true` (sent on the next `sendMessageJob` run, not scheduled)
   - `intakeId`: reference to the intake

### Event 3: Symptom/Checkout Follow-up (source: `symptom`)

**Trigger**: The `looker_appointments` webhook reports an appointment status change to `Complete`.

1. `webhookUpsertAppointments` detects the status change from `Active`/`Rescheduled` to `Complete`.
2. Calls `setSymptomMessage(appointmentId, startDate, patientId)` which creates a `Message` document:
   - `type: sms`, `source: symptom`, `status: READY`
   - `dateToSend`: next day at 1 PM EST

---

## How Messages Get Sent

The `sendMessageJob` runs periodically during work hours (9 AM - 3 PM EST):

1. Queries `messages` collection for documents with `status: READY` where `dateToSend` falls within today's window (or `immediately: true`).
2. Checks feature flags (`APPOINTMENT_REMINDER`, `SEND_SYMPTOM`, `SEND_WHAT_TO_EXPECT`).
3. Checks holiday calendar: REMINDER messages are sent on holidays; SYMPTOM and WHAT_TO_EXPECT are rescheduled to the next business day.
4. For each message (via `Promise.allSettled`):
   - Sets `status: SENDING` and saves.
   - Calls `sendSmsMessageByAppointment()` or `sendSmsMessageByIntake()`.
   - Inside, calls `sendSpruceSms()` which posts the SMS via the Spruce API.
   - Spruce returns a `requestId` (e.g., `asyncRequest_2NE1B99RGG800`).
   - Sets `status: SENT`, `sentDate`, `patientId`, and `spruce: { requestId, txt, phone }`.
   - **Immediately saves** the message to persist `requestId` to MongoDB (MLID-2167 fix).
5. After all messages settle, a deferred save persists final state for each message.

At this point, the message has `spruce.requestId`, `spruce.txt`, and `spruce.phone` — but **no** `conversationId`, `appUrl`, or `archivedAt`. Those come from the Spruce webhook.

---

## How Archival Works

### Primary Path: Spruce Outbound Webhook

After Spruce delivers the SMS to the patient, it fires a `conversationItem.created` webhook back to our system (`spruce_conversation.ts`):

1. Webhook payload includes `requestId`, `conversationId`, `appURL`, and `direction: outbound`.
2. Handler queries `Message.updateMany({ 'spruce.requestId': requestId })` to set:
   - `spruce.conversationId` (Spruce thread ID)
   - `spruce.appUrl` (link to Spruce UI)
3. Immediately calls `archiveSmsConversation(requestId, conversationId)`:
   - Posts an internal note "Processed by LISA" to the Spruce conversation.
   - Waits 2 seconds for Spruce to process the note.
   - Archives the Spruce conversation via `archiveSpruceConversation(conversationId)`.
   - On success, stamps `spruce.archivedAt` on the message document.

### Safety Net: `spruceArchiveSweeper` Job

Runs every few minutes. Catches messages where the webhook set `conversationId` but archival itself failed (Spruce API error, process crash, transient MongoDB error):

```
Query:
  source: [reminder, symptom, what-to-expect, ...]
  spruce.conversationId: { $exists: true }       <-- REQUIRES conversationId
  spruce.archivedAt: { $exists: false }
  sentDate: within last 58-60 minutes             <-- 1-hour lookback window
```

For each candidate, it atomically claims the message (sets `archivedAt`), attempts archival, and releases the claim if it fails.

---

## The Race Condition (MLID-2167)

The `requestId` is the only link between the message in our database and the Spruce webhook. The timeline is:

```
sendMessageJob                              Spruce
     |                                        |
     |-- POST SMS to Spruce API ------------->|
     |                                        |-- delivers SMS to patient
     |<-- response: { requestId } ------------|
     |                                        |-- fires webhook: { requestId, conversationId }
     |-- save requestId to MongoDB            |
     |        (message.save())                |
     |                                        |
     |                              webhook handler runs:
     |                              Message.updateMany({ 'spruce.requestId': requestId })
```

**The problem**: Spruce can fire the webhook **before** our code saves the `requestId` to MongoDB. When the webhook handler runs `Message.updateMany({ 'spruce.requestId': requestId })`, it finds 0 matches because the `requestId` isn't in the database yet.

The MLID-2167 fix (commit `a5d014cb`) moved the `message.save()` to happen immediately after each individual send (instead of in a deferred batch). This narrowed the race window but did not eliminate it — the `save()` still happens **after** the Spruce API response, and the webhook can fire during or before that response.

---

## Why This Specific Message Was Not Archived

**Message `69e8c652752417d8e83f2e4c`** (Bangor, ME — TEST SIX):

| Field | Value |
|---|---|
| `spruce.requestId` | `asyncRequest_2NE1B99RGG800` |
| `spruce.conversationId` | **MISSING** |
| `spruce.appUrl` | **MISSING** |
| `spruce.archivedAt` | **MISSING** |

**Compared to a working message** `69e8c652752417d8e83f2e70` (Augusta, ME):

| Field | Value |
|---|---|
| `spruce.requestId` | `asyncRequest_2NE1B9NM0HG00` |
| `spruce.conversationId` | `t_2GUJ2OR71KG00` |
| `spruce.appUrl` | `https://app.sprucehealth.com/...` |
| `spruce.archivedAt` | `2026-04-22T13:30:00.949Z` |

**What happened**: The Spruce `conversationItem.created` webhook for `requestId: asyncRequest_2NE1B99RGG800` either:
- Arrived before `requestId` was persisted to MongoDB (race condition), so `Message.updateMany` matched 0 documents and `conversationId` was never set, OR
- Was never delivered by Spruce at all (network issue, Spruce internal error).

**Why the sweeper didn't catch it**: The `spruceArchiveSweeper` requires `spruce.conversationId: { $exists: true }`. Since the webhook never set `conversationId`, the sweeper's query excludes this message entirely. Additionally, the sweeper only looks back 1 hour — the message aged out of the window long ago.

**Both safety nets failed**:
1. Primary path (webhook) — couldn't match `requestId` in MongoDB.
2. Safety net (sweeper) — filters on `conversationId` existing, which it doesn't.

---

## Gap in Current Design

The sweeper was designed for the failure mode: "webhook arrived and set `conversationId`, but the archive API call failed." It does **not** cover the failure mode: "webhook missed entirely, so `conversationId` was never set."

Messages that lose the race condition end up in a permanent orphan state:
- They have `spruce.requestId` but no `spruce.conversationId`.
- No existing job or webhook will ever set `conversationId` retroactively.
- The sweeper ignores them.
- They appear in the UI as "Sent" without the arrow link to Spruce.

---

## The `conversationitems` Overwrite Problem

### Initial hypothesis: recover from `conversationitems`

A natural fix would be to have the sweeper look up `requestId` in the `conversationitems` collection to recover the `conversationId`. The Spruce webhook handler saves every webhook payload to `conversationitems` at line 268 of `spruce_conversation.ts` — this happens **before** the bridging logic that links to `messages` (line 322). So even when the bridging fails (race condition), the raw Spruce data should still be in `conversationitems`.

### What we found: `requestId` is being overwritten to empty string

Querying outbound items in `conversationitems`:

| requestId | Count |
|---|---|
| `""` (empty string) | **317,215** |
| non-empty (e.g., `asyncRequest_XXX`) | **49,454** |

**86% of outbound conversation items have lost their `requestId`.**

### Root cause: `$set: { data }` replaces the entire payload

The culprit is the `findOneAndUpdate` at `spruce_conversation.ts` line 268:

```typescript
await ConversationItem.findOneAndUpdate(
  { conversation_item_id: conversationItemId },
  {
    $set: { data },           // ← REPLACES entire data field
    $setOnInsert: {
      conversation_id: conversationId,
    },
  },
  { upsert: true, new: true }
);
```

The handler processes both `conversationItem.created` and `conversationItem.updated` events (line 250-253). Spruce fires **two** webhooks for each outbound SMS, both sharing the same `conversationItemId`:

```
Webhook 1: conversationItem.created
  → requestId: "asyncRequest_2NE1B99RGG800"    ← has the value
  → findOneAndUpdate: inserts new doc with requestId

Webhook 2: conversationItem.updated
  → requestId: ""                                ← empty string
  → findOneAndUpdate: $set: { data } overwrites the entire data field
  → requestId is now ""
```

Because `$set: { data }` replaces the entire `data` field, the second webhook (`.updated`) blanks out the `requestId` that the first webhook (`.created`) had saved.

### Why archival still works most of the time

The primary archival path (lines 282-339) runs during the **first** webhook (`conversationItem.created`), which **does** have the `requestId`. The archival logic runs inline, before the `.updated` webhook arrives and blanks the `requestId`. So for the happy path, archival completes before the overwrite happens.

The overwrite only matters for **recovery** — if the primary archival fails (race condition, process crash), the `conversationitems` collection can no longer be used as a fallback because the `requestId` has been blanked.

### Verification: the Bangor message

The Bangor message (`asyncRequest_2NE1B99RGG800`) is one of the rare cases where `requestId` was **not** overwritten in `conversationitems`:

```
conversationitems document:
  conversation_id: "t_2GUJ4SUTDD800"
  data.data.object.requestId: "asyncRequest_2NE1B99RGG800"
  data.data.object.conversationId: "t_2GUJ4SUTDD800"
  data.data.object.appURL: "https://app.sprucehealth.com/org/entity_1RJ8VSP7ND800/thread/t_2GUJ4SUTDD800/message/ti_2NE1B9AI0BO02"
```

This data is exactly what the `messages` document is missing. The `.updated` webhook either hasn't arrived yet for this item, or it was a different `conversationItemId`. But this is the **exception** — for most messages, this recovery data is already gone.

---

## Summary of Problems

There are **three layered problems** that together cause messages to become permanently orphaned:

| # | Problem | Location | Effect |
|---|---------|----------|--------|
| 1 | **Race condition**: Spruce webhook arrives before `requestId` is saved to `messages` | `sendMessageJob.ts` (line 386) vs `spruce_conversation.ts` (line 322) | `Message.updateMany` matches 0 docs → `conversationId` never set on message |
| 2 | **Sweeper blind spot**: requires `conversationId` to exist | `spruceArchiveSweeper.ts` (line 69) | Messages without `conversationId` are invisible to the sweeper |
| 3 | **`conversationitems` overwrite**: `$set: { data }` blanks `requestId` on `.updated` webhook | `spruce_conversation.ts` (line 268) | Destroys the recovery data that a sweeper fallback would need |

Problem 1 creates the orphan. Problem 2 means no existing job can rescue it. Problem 3 means the backup data (in `conversationitems`) that could be used for recovery is destroyed 86% of the time.

---

## Fix Plan

**Branch**: `fix/MLID-2167-spruce-archive-requestid-recovery`
**Full plan**: `docs/agomez/plans/MLID-2167-requestid-recovery-plan.md`

Two fixes, applied in order. Fix 1 stops the data loss so that Fix 2 can use `conversationitems` as a reliable recovery source.

### Fix 1: Preserve `requestId` in `conversationitems`

**File**: `apps/web/services/webhooks/handlers/spruce_conversation.ts` (line 264-276)

**Problem**: `$set: { data }` replaces the entire `data` field. The `.updated` webhook (with `requestId: ""`) overwrites the `requestId` that the `.created` webhook saved. 86% of outbound items have lost their `requestId`.

**Fix**: When the incoming webhook has an empty `requestId`, preserve the existing value instead of overwriting it. This ensures the `conversationitems` collection always retains the `requestId` for future recovery.

### Fix 2: Expand sweeper with `conversationitems` fallback (two-pass)

**File**: `apps/web/services/jobs/definitions/spruceArchiveSweeper.ts`

**Problem**: The sweeper only finds messages with `spruce.conversationId` already set. Orphaned messages (have `requestId` but no `conversationId`) are invisible.

**Fix**: Add a second pass to the sweeper:

```
Pass 1 (existing, unchanged):
  Find messages with conversationId but no archivedAt → archive them

Pass 2 (new):
  Find orphaned messages: have spruce.requestId, no spruce.conversationId
    │
    ▼
  Look up requestId in conversationitems collection
    │
    ▼
  If found: backfill conversationId + appUrl on the message
    │
    ▼
  Proceed with archival (noteAndArchiveConversation)
    │
    ▼
  Stamp spruce.archivedAt
```

Pass 2 uses a longer lookback window (`ORPHAN_LOOKBACK_MS`, e.g., 24 hours) since these messages were missed entirely and could be older than the 1-hour window used by Pass 1.

### What we're NOT changing

- **`sendMessageJob.ts`**: The MLID-2167 immediate-save fix (commit `a5d014cb`) is already in place. The race condition window is narrow but can't be fully eliminated at this layer.
- **Webhook retry logic**: Initially considered scheduling a short-delay retry when `Message.updateMany` returns `modifiedCount: 0`, but Fix 1 + Fix 2 together make this unnecessary. The sweeper serves as the reliable safety net.
