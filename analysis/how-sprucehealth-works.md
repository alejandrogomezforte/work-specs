# How Spruce Health Works in li-call-processor

## Overview

Spruce Health is integrated for three purposes: **SMS messaging** to patients, **fax document intake**, and **conversation tracking**. The integration uses an OpenAPI-generated SDK client and manages credentials through Azure Key Vault.

At a high level, we talk to Spruce in two directions:

- **Outbound** — scheduled background jobs dispatch SMS through the Spruce API (appointment reminders, symptom check-ins, "what to expect", Skyrizi campaigns, reverification).
- **Inbound** — Spruce calls back into our app via webhooks whenever conversations or conversation items change (SMS deliveries, patient replies, inbound fax documents).

After each outbound SMS we also clean up the Spruce inbox: we post a "Processed by LISA" internal note and archive the conversation so the Spruce app doesn't fill up with noise. That cleanup runs in two places — the webhook does it in real time (fast path), and a recurring sweeper job catches anything the webhook missed (safety net).

---

## High-Level Architecture

```
                                     Patient
                                        ^
                                        | SMS (inbound + outbound)
                                        v
                          ┌──────────────────────────┐
                          │    SPRUCE HEALTH CLOUD    │
                          │  (phone numbers, inboxes) │
                          └───────────┬──────────────┘
                            ^         |          ^
                  REST API  |         | Webhooks | REST API
                  (SMS,     |         | (events) | (archive, note,
                   archive, |         |          |  intakes)
                   note)    |         v          |
                ┌───────────┴────────────────────┴──────────────┐
                │             li-call-processor                 │
                │  ┌──────────────────────────────────────────┐ │
                │  │         Pulse scheduler (worker.ts)      │ │
                │  │                                          │ │
                │  │  • appointmentReminder   (9 AM ET daily) │ │
                │  │  • sendMessageProcessing (every 2 min)   │ │
                │  │  • skyriziObiTraining    (6 PM ET Mon-Fr)│ │
                │  │  • skyriziCompletePromo  (6 PM ET Mon-Fr)│ │
                │  │  • reverificationCampaign                │ │
                │  │  • spruceArchiveSweeper  (every 15 min)  │ │
                │  │  • spruceIntake / spruceFax              │ │
                │  └──────────────────────────────────────────┘ │
                │                   │          ^                │
                │                   v          │                │
                │  ┌──────────────────────────────────────────┐ │
                │  │             MongoDB                      │ │
                │  │  Message • Conversation • ConversationItem│ │
                │  │  Intake  • Patient • Appointment         │ │
                │  └──────────────────────────────────────────┘ │
                │                   ^                           │
                │                   │                           │
                │  ┌──────────────────────────────────────────┐ │
                │  │  Webhook endpoint                        │ │
                │  │  /api/webhooks/spruce_conversations      │ │
                │  │  (HMAC verify -> upsert -> dispatch)     │ │
                │  └──────────────────────────────────────────┘ │
                │                   │                           │
                │                   v                           │
                │            Azure Blob Storage                 │
                │   (incoming faxes, intake PDFs, webhook logs) │
                └────────────────────────────────────────────────┘
```

**How to read it:**

1. Our **Pulse jobs** decide when to send SMS (reminders, campaigns) and call the Spruce API.
2. Spruce delivers the SMS and **calls our webhook** for every state change.
3. Our webhook handler updates Mongo, links the delivery back to the original `Message` doc, and (for outbound SMS) posts a note + archives the Spruce conversation.
4. If the webhook miss (process crash, transient error, Spruce delivery failure), **`spruceArchiveSweeper`** reconciles Mongo vs Spruce every 15 minutes.
5. For inbound fax attachments, the webhook schedules an **intake job** that downloads the PDF and pushes it to Azure Blob Storage.

---

## 1. Outbound SMS Messaging (App -> Spruce)

### Flow

```
Scheduled Job (sendMessageJob)
  -> messageService.sendSpruceSms()
    -> spruceApi.postMessageFromEndpoint()
      -> POST /v1/internalendpoints/{id}/messages
```

### How it works

1. A background job fires (e.g., appointment reminder, "what to expect", symptom check-in)
2. `sendSpruceSms()` formats phone numbers in E.164 format (`+1{number}`)
3. Fetches the organization's internal endpoints via `spruceApi.internalEndpoints()`
4. Matches the location's `callback_number` to find the right Spruce phone endpoint
5. Sends SMS via `spruceApi.postMessageFromEndpoint()` with:
   - `destination: { smsOrEmailEndpoint: targetPhoneNumber }`
   - `message: { body: [{ type: 'text', value: text }] }`
6. Stores the returned `requestId` on the `Message` document for webhook linking

### Message types sent via Spruce

All message sources are defined in `apps/web/types/messageDTO.ts` (`MessageSource` enum). Each source represents one outbound SMS campaign; all of them funnel through the same `sendSpruceSms()` dispatcher, but they differ in what triggers them, what content they send, and which feature flag gates them.

| Source | Feature Flag | Triggered by | Description |
|--------|-------------|--------------|-------------|
| `REMINDER` | `APPOINTMENT_REMINDER` | `appointmentReminder` job (9 AM ET daily) + `sendMessageProcessing` (every 2 min) | Appointment reminder SMS sent 2 days before the appointment. Sends even on holidays (per MLID-1540). |
| `WHAT_TO_EXPECT` | `SEND_WHAT_TO_EXPECT` | `sendMessageProcessing` (every 2 min) | Post-intake SMS explaining what to expect at the upcoming appointment. |
| `SYMPTOM` | `SEND_SYMPTOM` | `sendMessageProcessing` (every 2 min) | Pre-appointment symptom check-in (sent 1 day before). Display label: `checkout`. |
| `SKYRIZI_OBI_TRAINING` | `SEND_SKYRIZI_OBI_TRAINING` | `skyriziObiTraining` job (6 PM ET weekdays) | Skyrizi on-body injector (OBI) training materials sent after the patient's **2nd** Skyrizi appointment. |
| `SKYRIZI_COMPLETE_PROMOTION` | `SEND_SKYRIZI_COMPLETE_PROMOTION` | `skyriziCompletePromotion` job (6 PM ET weekdays) | "Skyrizi Complete" patient-support program promotion sent after the patient's **1st** Skyrizi appointment. |
| `REVERIFICATION` | `REVERIFICATION_CAMPAIGN` | `reverificationCampaign` job | Insurance reverification outreach. Also uses `sendSpruceSms()`, but it is **not** in the archivable set (see §4 / §5). |

### Skyrizi campaigns (drug-program SMS)

Both Skyrizi campaigns live under `apps/web/services/messages/skyrizi/` and share a common base (`skyriziBaseService.ts`):

- **OBI training** (`skyriziObiTrainingService.ts`): fires for appointments where `first_appt_on_order_series: false` and `first_listed_primary_med: 'Skyrizi'`. Message body links to AbbVie's official on-body injector instructions.
- **Complete promotion** (`skyriziCompletePromotionService.ts`): fires for appointments where `first_appt_on_order_series: true`, `first_listed_primary_med: 'Skyrizi'`, and appointment status `Complete`. Message body promotes the Skyrizi Complete patient-support program.

Both campaigns:
- Run at 6 PM ET **weekdays only** (cron `0 18 * * 1-5`).
- Target the **previous business day**'s appointments (handles weekend/holiday gaps via `isBusinessDay`).
- Deduplicate (skip appointments already messaged today), write `DECLINED` `Message` records for skipped patients for reporting.
- Are gated by the `APPOINTMENT_REMINDER_TARGET_CRITERIA` env whitelist (same safety gate as reminders — see §12).

### Key files

- `apps/web/services/messages/messageService.ts` -> `sendSpruceSms()` — single dispatcher used by all sources
- `apps/web/services/messages/skyrizi/skyriziBaseService.ts` — shared campaign logic
- `apps/web/services/messages/skyrizi/skyriziObiTrainingService.ts` — OBI training campaign
- `apps/web/services/messages/skyrizi/skyriziCompletePromotionService.ts` — Complete program promotion campaign
- `apps/web/services/messages/reverificationService.ts` — reverification outreach campaign

---

## 2. Inbound Webhooks (Spruce -> App)

### Endpoint

`pages/api/webhooks/spruce_conversations.ts` (Pages Router - active)

There is also a legacy/placeholder at `app/api/webhooks/spruce/route.ts` that returns a stub response and does not call the actual handler.

### Signature verification

- Header: `x-spruce-signature` (Base64-encoded HMAC-SHA256)
- Secret stored in MongoDB integration settings (`signingSecret.value`)
- Time-safe comparison to prevent timing attacks

### Event types handled

| Event | Action |
|-------|--------|
| `conversation.created` | Upserts `Conversation` doc in MongoDB |
| `conversation.updated` | Upserts `Conversation` doc in MongoDB |
| `conversationItem.created` | Upserts `ConversationItem` doc + triggers downstream logic |
| `conversationItem.updated` | Upserts `ConversationItem` doc |

### conversationItem.created downstream logic

**For outbound items** (direction=outbound with requestId):
- Links the webhook back to the original `Message` doc via `spruce.requestId`
- Updates `Message` with `spruce.conversationId` and `spruce.appUrl`
- If message source is REMINDER:
  - Optionally posts internal note (flag: `SPRUCE_SMS_REMINDER_INTERNAL_NOTE`)
  - Optionally archives conversation (flag: `SPRUCE_SMS_REMINDER_ARCHIVE`)

**For inbound items with attachments** (fax documents):
- Validates URL expiry (signed URLs have TTL)
- Schedules a `spruceIntakeJob` for each document attachment

### Key files

- Webhook endpoint: `apps/web/pages/api/webhooks/spruce_conversations.ts`
- Webhook handler: `apps/web/services/webhooks/handlers/spruce_conversation.ts`

---

## 3. Fax Document Intake (Spruce -> App -> Azure)

### Flow

```
Spruce fax arrives
  -> Webhook fires conversationItem.created with attachments
    -> spruceIntakeJob scheduled
      -> Downloads PDF from signed URL
      -> Finds location by fax number (internal endpoint rawValue)
      -> Creates Intake document (type: 'spruceFax', status: 'pending')
      -> Uploads PDF to Azure Blob Storage
      -> Schedules processIntakeJob (if INTAKE_PROCESSING flag enabled)
```

### Azure Blob Storage path

```
documents/intakes/{intake._id}/{timestamp}/{filename}
```

### spruceIntakeJob retry strategy

| Setting | Value |
|---------|-------|
| Attempts | 5 |
| Backoff | Fixed 30 minutes |
| Concurrency | 3 |
| Lock lifetime | 20 minutes |
| Priority | High |

### Alternate fax job: processSpruceFax

There is also a `processSpruceFax` job (`services/jobs/definitions/spruceFax.ts`) that:
1. Fetches conversation item details from Spruce API
2. Downloads each attachment
3. Uploads to `incoming-faxes` Azure container with metadata
4. Stores conversation metadata as JSON blob

### Key file

`apps/web/services/jobs/definitions/spruceIntake/spruceIntakeJob.ts`

---

## 4. Post-Send Actions (Internal Notes & Archiving)

After an outbound SMS is accepted by Spruce, we want to keep the Spruce inbox tidy: every conversation we open with a patient via an automated message should get a "Processed by LISA" internal note and then be archived, so staff only see conversations that actually need human attention.

This happens via a **two-layer** design:

```
   ┌──────────────────────────────┐      ┌────────────────────────────────┐
   │  FAST PATH — webhook-driven  │      │  SAFETY NET — sweeper-driven   │
   │                              │      │                                │
   │  Spruce sends SMS →          │      │  spruceArchiveSweeper cron      │
   │  conversationItem.created    │      │  runs every 15 min              │
   │  webhook fires (ms)          │      │                                │
   │       ↓                      │      │  Finds SENT messages with      │
   │  Handler links requestId →   │      │  spruce.conversationId but no  │
   │  Message, posts note,        │      │  spruce.archivedAt, and runs   │
   │  archives conversation,      │      │  the same note + archive flow  │
   │  stamps spruce.archivedAt    │      │  (§5)                          │
   └──────────────────────────────┘      └────────────────────────────────┘
             ≈ 99%+ of cases                      residual / recovery
```

The fast path handles nearly every message the moment Spruce confirms it. The sweeper exists for the residual cases where the webhook doesn't run (process crash, transient Mongo error, Spruce failing to deliver the webhook). Both paths share the same primitives in `spruce_conversation.ts`:

- `noteAndArchiveConversation(conversationId, logPrefix)` — posts the note, waits for Spruce to commit it, then archives.
- `ARCHIVABLE_SOURCE_LOG_PREFIXES` — the single source of truth for which `MessageSource` values are archivable. Today: `REMINDER`, `SYMPTOM`, `WHAT_TO_EXPECT`, `SKYRIZI_OBI_TRAINING`, `SKYRIZI_COMPLETE_PROMOTION`. `REVERIFICATION` is intentionally **not** in this set.
- `spruce.archivedAt` timestamp on the `Message` doc — the idempotency key shared by both paths. Once stamped, any later webhook or sweeper run will skip the message.

### Internal notes

Uses `POST /v1/conversations/{conversationId}/messages` with `internal: true`:
- SDK method: `spruceApi.postConversationMessage()`
- Posts "Processed by LISA" note to the conversation
- Gated by `SPRUCE_SMS_REMINDER_INTERNAL_NOTE` (name is legacy — it also governs Skyrizi, what-to-expect, and symptom archival notes)

### Archiving

Uses `PATCH /v1/conversations/{conversationId}` with `archived: true`:
- SDK method: `spruceApi.updateConversation()`
- Gated by `SPRUCE_SMS_REMINDER_ARCHIVE` (same legacy-name caveat as above)
- The archive step **requires the note step to have succeeded first**; if the note post fails, the archive is skipped for that run.

### Feature flags controlling these actions

| Flag | Default | Purpose |
|------|---------|---------|
| `SPRUCE_SMS_REMINDER_INTERNAL_NOTE` | `false` | Post "Processed by LISA" note to outbound SMS conversations (all archivable sources, not just REMINDER) |
| `SPRUCE_SMS_REMINDER_ARCHIVE` | `false` | Archive outbound SMS conversations (all archivable sources) |
| `SPRUCE_ARCHIVE_SWEEPER` | `false` | Gate for the safety-net sweeper job itself (§5) |
| `SPRUCE_FAX_INTERNAL_NOTE` | `false` | Post note to fax conversations |
| `SPRUCE_FAX_ARCHIVE` | `false` | Archive fax conversations |

> **Note on dependencies:** The sweeper short-circuits to a no-op if either `SPRUCE_ARCHIVE_SWEEPER` or `SPRUCE_SMS_REMINDER_ARCHIVE` is off. Turning off `SPRUCE_SMS_REMINDER_ARCHIVE` while `SPRUCE_ARCHIVE_SWEEPER` is on would otherwise cause the sweeper to claim candidates, call `noteAndArchiveConversation` (which immediately returns `false`), and then release them on every tick — an effectively infinite loop until `sentDate` ages out of the lookback window.

### Key files

- `apps/web/services/integrations/spruce.ts` -> `postSpruceInternalNote()`, `archiveSpruceConversation()`
- `apps/web/services/webhooks/handlers/spruce_conversation.ts` -> `noteAndArchiveConversation()`, `ARCHIVABLE_SOURCE_LOG_PREFIXES`

---

## 5. Archive Sweeper (`spruceArchiveSweeper`)

**File:** `apps/web/services/jobs/definitions/spruceArchiveSweeper.ts`
**Introduced in:** MLID-2167

### Why it exists

The webhook handler archives Spruce conversations in milliseconds after an outbound SMS — it's the primary path. But the webhook can miss: the worker process can crash mid-handler, a transient Mongo write can fail, Spruce can fail to deliver the webhook, etc. Without a backstop, those messages would sit forever as unarchived conversations cluttering the Spruce inbox.

`spruceArchiveSweeper` is that backstop. It runs every 15 minutes and reconciles Mongo state against Spruce: for every outbound SMS that was sent but not yet stamped `spruce.archivedAt`, it runs the same note + archive flow the webhook would have run.

### Schedule & configuration

| Setting | Value |
|---------|-------|
| Cron | `*/15 * * * *` (every 15 minutes) |
| Timezone | `America/New_York` |
| Attempts | 1 (no internal retries — next run will pick it up) |
| Lock lifetime | 5 minutes |
| Concurrency | 1 (serialized) |
| Priority | low |
| `skipImmediate` | true (don't run immediately on scheduler startup) |

Registered in `apps/web/services/jobs/scheduler.ts`.

### Two-pass algorithm

The job runs two passes within each tick:

**Pass 1 — Primary archival (1-hour lookback)**

Find `Message` docs where:
- `source ∈ ARCHIVABLE_SOURCE_LOG_PREFIXES` (the 5 archivable sources from §4)
- `spruce.conversationId` exists (webhook linked it)
- `spruce.archivedAt` does **not** exist (not yet archived)
- `sentDate` is between `now - 60 min` and `now - 2 min` (the 2-minute skip lets the real-time webhook win)

Batch limit: 100 per run.

For each candidate:

1. **Claim atomically** via `findOneAndUpdate({ _id, archivedAt: { $exists: false } }, { $set: archivedAt: now })`. If the claim misses, the webhook beat us to it — skip.
2. Call `noteAndArchiveConversation()` — post the "Processed by LISA" note, wait ~2s, archive.
3. On success: leave `archivedAt` stamped, increment `archived`.
4. On failure: `$unset` the `archivedAt` stamp to release the claim so the next tick can retry, increment `released`.

**Pass 2 — Orphan recovery (24-hour lookback)**

Some messages get sent but never receive a `conversationId` because the corresponding webhook never landed. Pass 2 catches these:

Find `Message` docs where:
- `source ∈ ARCHIVABLE_SOURCE_LOG_PREFIXES`
- `spruce.requestId` exists (we know Spruce accepted the SMS)
- `spruce.conversationId` does **not** exist
- `spruce.archivedAt` does not exist
- `sentDate` is between `now - 24 h` and `now - 2 min`

For each orphan:

1. Look up a `ConversationItem` doc whose payload contains `data.data.object.requestId === requestId`. Webhook logging stores every conversation-item event, so the `conversationId` is usually recoverable from there.
2. If no match → log warning and count as `orphansMissing`.
3. If matched → backfill `spruce.conversationId` (and `spruce.appUrl` if present) onto the `Message`, then run the same atomic-claim + note + archive flow as Pass 1.

### Result object

The job returns a structured summary for logging and metrics:

```typescript
{
  scanned: number,           // Pass 1 candidates examined
  claimed: number,           // Pass 1 candidates successfully claimed
  archived: number,          // Pass 1 candidates successfully archived
  released: number,          // Pass 1 candidates where archival failed and was released
  orphansScanned: number,    // Pass 2 orphans examined
  orphansRecovered: number,  // Pass 2 orphans whose conversationId was recovered and archived
  orphansMissing: number     // Pass 2 orphans with no recoverable conversationId
}
```

### Key invariants

- **Idempotent**: `spruce.archivedAt` is the single source of truth. Webhook + sweeper can never both post the note for the same message; whichever writes `archivedAt` first wins.
- **Bounded scope**: lookback windows prevent the job from ever chewing through deep history.
- **Non-destructive on failure**: failed archival releases the claim rather than leaving `archivedAt` stamped on an un-archived conversation.
- **Feature-flag safe**: if either gating flag is disabled, the job returns all zeros immediately — no claim-then-release churn.

---

## 6. Admin Configuration

### UI

`apps/web/app/admin/integrations/spruce/page.tsx`

Features:
- Status display (enabled/disabled, last updated, webhook ID)
- Webhook endpoint URL and masked API key
- Test Connection button
- Enable/Disable toggle
- Subscribe/Unsubscribe webhook buttons

### API routes

| Route | Methods | Purpose |
|-------|---------|---------|
| `/api/admin/integrations/spruce` | GET, POST | Read config, test/enable/disable/update |
| `/api/admin/integrations/spruce/subscription` | POST, DELETE | Subscribe/unsubscribe webhooks |
| `/api/appointments/spruce` | GET | Returns spruceUrl and spruceOrganization config |

### Integration lifecycle

- `enableSpruceIntegration()`: Tests connection -> stores API key in Key Vault -> registers webhook -> creates MongoDB integration record -> stores webhook secret
- `disableSpruceIntegration()`: Fetches webhook -> deletes from Spruce -> marks as disabled in MongoDB

---

## 7. Spruce API Endpoints Used

### Outbound (App -> Spruce)

| Method | Path | Purpose | SDK Method |
|--------|------|---------|------------|
| POST | `/v1/internalendpoints/{id}/messages` | Send SMS | `postMessageFromEndpoint()` |
| POST | `/v1/conversations/{id}/messages` | Post internal note | `postConversationMessage()` |
| PATCH | `/v1/conversations/{id}` | Archive conversation | `updateConversation()` |
| POST | `/v1/webhooks/endpoints` | Register webhook | direct axios |
| DELETE | `/v1/webhooks/endpoints/{id}` | Unregister webhook | direct axios |
| GET | `/v1/organizations/self` | Test connection | direct axios |
| GET | `/v1/contacts` | List contacts | direct axios |
| GET | `/v1/contacts/{id}` | Get contact | direct axios |
| GET | `/v1/conversations` | List conversations | direct axios |
| GET | `/v1/conversations/{id}` | Get conversation | direct axios |
| GET | `/v1/conversations/{id}/items` | List conversation items | direct axios |
| GET | `/v1/internalendpoints` | List internal endpoints | `internalEndpoints()` |

### Inbound (Spruce -> App)

| Path | Events |
|------|--------|
| `/api/webhooks/spruce_conversations` | conversation.created/updated, conversationItem.created/updated |

---

## 8. Data Models

### Message model (`models/Message.ts`)

Spruce-specific fields:
```typescript
spruce?: {
  requestId: string;       // API response ID (from postMessageFromEndpoint)
  txt?: string;            // Message text
  phone?: string;          // Phone number (E.164)
  conversationId?: string; // Populated by webhook callback (or sweeper Pass 2)
  appUrl?: string;         // Conversation URL in Spruce app
  archivedAt?: Date;       // Set when the conversation has been archived in Spruce;
                           // shared idempotency key between the webhook fast path
                           // and the spruceArchiveSweeper safety net (see §4, §5)
}
```

Indexes: `spruce.requestId`, `spruce.appUrl`

### Conversation model (`models/Conversation.ts`)

```typescript
{
  conversation_id: string;    // Spruce conversation ID
  data: Schema.Types.Mixed;   // Full webhook payload
  createdAt: Date;
  updatedAt: Date;
}
```

Index: `{ conversation_id: 1 }`

### ConversationItem model (`models/ConversationItem.ts`)

```typescript
{
  conversation_id: string;      // Parent conversation
  conversation_item_id: string; // Spruce item ID (unique)
  data: Schema.Types.Mixed;     // Full webhook payload
  createdAt: Date;
  updatedAt: Date;
}
```

Indexes: `{ conversation_item_id: 1 }` (unique), `{ conversation_id: 1 }`, `{ conversation_id: 1, conversation_item_id: 1 }` (compound)

---

## 9. OpenAPI SDK

**Location:** `apps/web/apis/spruce/`

| File | Purpose |
|------|---------|
| `openapi.json` | OpenAPI 3.0 specification |
| `schemas.ts` | Generated TypeScript types |
| `index.ts` | Generated SDK class |

### SDK methods used in the codebase

| Method | Where used |
|--------|-----------|
| `auth(token)` | `spruce.ts` -> `configureSpruceApi()` |
| `internalEndpoints()` | `messageService.ts` -> `sendSpruceSms()` |
| `postMessageFromEndpoint(body, metadata)` | `messageService.ts` -> `sendSpruceSms()` |
| `postConversationMessage(body, metadata)` | `spruce.ts` -> `postSpruceInternalNote()` |
| `updateConversation(body, metadata)` | `spruce.ts` -> `archiveSpruceConversation()` |

---

## 10. Service Layer Functions

**File:** `apps/web/services/integrations/spruce.ts`

### Connection & Auth
- `testSpruceConnection(apiKey)` - Validates API key
- `configureSpruce(apiToken)` - Configures axios with Bearer token
- `configureSpruceApi()` - Lazy-loads OpenAPI SDK

### Integration Management
- `enableSpruceIntegration(endpointUrl, apiKey, userId)` - Full setup
- `disableSpruceIntegration(userId)` - Full teardown

### Webhook Operations
- `registerSpruceWebhook(endpointUrl)` - Returns webhookId + secret
- `deleteSpruceWebhook(webhookId)` - Removes webhook

### Data Retrieval
- `listSpruceContacts(paginationToken)` - Paginated contacts
- `getSpruceContact(contactId)` - Single contact
- `listSpruceConversations(options)` - Paginated conversations
- `getSpruceConversation(conversationId)` - Single conversation
- `listSpruceConversationItems(conversationId, options)` - Conversation messages
- `listSpruceInternalEndpoints()` - Organization endpoints

### Messaging
- `sendSpruceMessage(internalEndpointId, destination, text, options)` - Generic send
- `scheduleSpruceMessage(conversationId, text, scheduledAt, options)` - Scheduled send
- `postSpruceInternalNote(conversationId, noteText)` - Internal note
- `archiveSpruceConversation(conversationId)` - Archive

---

## 11. Feature Flags

Defined in `apps/web/utils/featureFlags/featureFlags.ts` (`FeatureFlag` enum + `DEFAULT_FEATURE_FLAGS`). Code-defined defaults are used when a flag is missing from MongoDB.

### Outbound SMS campaigns

| Flag | Default | Purpose |
|------|---------|---------|
| `APPOINTMENT_REMINDER` | `false` | Enables `REMINDER` SMS sending |
| `SEND_WHAT_TO_EXPECT` | `false` | Enables `WHAT_TO_EXPECT` SMS |
| `SEND_SYMPTOM` | `false` | Enables `SYMPTOM` (checkout) SMS |
| `SEND_SKYRIZI_OBI_TRAINING` | `false` | Enables `SKYRIZI_OBI_TRAINING` SMS campaign |
| `SEND_SKYRIZI_COMPLETE_PROMOTION` | `false` | Enables `SKYRIZI_COMPLETE_PROMOTION` SMS campaign |
| `REVERIFICATION_CAMPAIGN` | `false` | Enables `REVERIFICATION` SMS outreach |

### Post-send archival / cleanup

| Flag | Default | Purpose |
|------|---------|---------|
| `SPRUCE_SMS_REMINDER_INTERNAL_NOTE` | `false` | Post "Processed by LISA" note on archivable SMS conversations (all 5 archivable sources, not just REMINDER) |
| `SPRUCE_SMS_REMINDER_ARCHIVE` | `false` | Archive SMS conversations after send (all 5 archivable sources). **Required** for the sweeper to do any work. |
| `SPRUCE_ARCHIVE_SWEEPER` | `false` | Gate for the `spruceArchiveSweeper` cron job (§5) |
| `SPRUCE_FAX_INTERNAL_NOTE` | `false` | Post internal note on fax conversations |
| `SPRUCE_FAX_ARCHIVE` | `false` | Archive fax conversations after processing |

### Operational

| Flag | Default | Purpose |
|------|---------|---------|
| `LOGS_FOR_SPRUCE_CONVERSATION` | `true` | Azure Blob logging of raw webhook payloads (audit trail) |
| `INTAKE_PROCESSING` | `true` | Process intakes after fax document upload |

---

## 12. Configuration & Secrets

### Environment variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `SPRUCE_API_BASE_URL` | `https://api.sprucehealth.com` | API endpoint |
| `AZURE_STORAGE_CONNECTION_STRING` | - | Blob storage for faxes |
| `FAX_CONTAINER_NAME` | `incoming-faxes` | Azure container name |
| `APPOINTMENT_REMINDER_TARGET_CRITERIA` | - | Safety whitelist for outbound SMS campaigns (reminders, Skyrizi, reverification). `"*"` = allow all (production); empty/unset = block all (safe default for dev/staging); comma-separated phone numbers = testing whitelist. |

### Azure Key Vault secrets

| Secret name | Purpose |
|-------------|---------|
| `spruce-api-token` | Spruce API bearer token |
| `integrations-spruce-signing-key` | Webhook signature secret |

### MongoDB integration record

Collection: `integrations`, key: `'spruce'`

```typescript
{
  enabled: boolean;
  settings: {
    webhookId: string;
    endpointUrl: string;
    secretVaultRef: string;
    apiKey: string; // masked: '••••••••••••••••'
  }
}
```

---

## 13. Data Flow Diagram

```
                    OUTBOUND                              INBOUND
              (App -> Spruce)                        (Spruce -> App)

┌─────────────────────────────┐                   ┌──────────────────┐
│  Pulse jobs:                │                   │  Spruce Webhook  │
│  • appointmentReminder      │                   │  Events          │
│  • sendMessageProcessing    │                   │                  │
│  • skyriziObiTraining       │                   │                  │
│  • skyriziCompletePromotion │                   │                  │
│  • reverificationCampaign   │                   │                  │
└────────┬────────────────────┘                   └────────┬─────────┘
         │                                                 │
         v                                                 v
┌──────────────────┐    POST /v1/.../messages    ┌──────────────────┐
│  messageService  │ ──────────────────────────► │   Spruce Health  │
│  .sendSpruceSms()│                             │   API            │
└──────────────────┘                             └──────────────────┘
         │                                                 │
         v                                                 v
┌──────────────────┐                              ┌──────────────────┐
│  MongoDB         │                              │  Webhook Handler │
│  Message doc     │ ◄─── requestId linking ────  │  spruce_         │
│  (spruce.*)      │                              │  conversation.ts │
└────────┬─────────┘                              └────────┬─────────┘
         ^                                                 │
         │ stamp archivedAt                   ┌────────────┼─────────────┐
         │ (idempotency key)                  │            │             │
         │                                    v            v             v
         │                              ┌──────────┐ ┌─────────┐  ┌─────────────┐
         │                              │ Upsert   │ │ Link to │  │ Schedule    │
         │                              │ Convo/   │ │ Message │  │ Intake Job  │
         │                              │ Item doc │ │ doc     │  │ (fax only)  │
         │                              └──────────┘ └────┬────┘  └──────┬──────┘
         │                                                │              │
         │           (outbound SMS → fast path)           v              v
         │                                    ┌──────────────────┐  ┌─────────────┐
         │                                    │ noteAndArchive   │  │ Download PDF│
         │◄───────────────────────────────────│ Conversation()   │  │ Create      │
         │                                    │  • post note     │  │ Intake doc  │
         │                                    │  • archive       │  │ Upload to   │
         │                                    │  • stamp Message │  │ Azure Blob  │
         │                                    └──────────────────┘  └─────────────┘
         │
         │  SAFETY NET — runs every 15 min
         │  ┌────────────────────────────────────────┐
         │  │  spruceArchiveSweeper (Pulse cron)     │
         └──┤  • Pass 1: archive SENT messages       │
            │    missing spruce.archivedAt (1h back) │
            │  • Pass 2: recover orphans with        │
            │    no conversationId (24h back)        │
            │  Claims atomically on archivedAt,      │
            │  calls same noteAndArchiveConversation │
            └────────────────────────────────────────┘
```

---

## 14. Key Files Reference

| Category | File Path (relative to apps/web/) |
|----------|-----------------------------------|
| Service layer | `services/integrations/spruce.ts` |
| SMS sending (shared dispatcher) | `services/messages/messageService.ts` |
| Skyrizi campaign (shared base) | `services/messages/skyrizi/skyriziBaseService.ts` |
| Skyrizi OBI training | `services/messages/skyrizi/skyriziObiTrainingService.ts` |
| Skyrizi Complete promotion | `services/messages/skyrizi/skyriziCompletePromotionService.ts` |
| Reverification campaign | `services/messages/reverificationService.ts` |
| Webhook endpoint (active) | `pages/api/webhooks/spruce_conversations.ts` |
| Webhook endpoint (legacy stub) | `app/api/webhooks/spruce/route.ts` |
| Webhook handler | `services/webhooks/handlers/spruce_conversation.ts` |
| Fax intake job | `services/jobs/definitions/spruceIntake/spruceIntakeJob.ts` |
| Fax storage job | `services/jobs/definitions/spruceFax.ts` |
| Conversation tracking job | `services/jobs/definitions/spruceConversation.ts` |
| Archive sweeper job | `services/jobs/definitions/spruceArchiveSweeper.ts` |
| Skyrizi OBI training job | `services/jobs/definitions/skyriziObiTraining.ts` |
| Skyrizi Complete promotion job | `services/jobs/definitions/skyriziCompletePromotion.ts` |
| Appointment reminder job | `services/jobs/definitions/appointmentReminder.ts` |
| Send message job | `services/jobs/definitions/sendMessageJob/sendMessageJob.ts` |
| Job scheduler (cron registration) | `services/jobs/scheduler.ts` |
| Admin API | `app/api/admin/integrations/spruce/route.ts` |
| Subscription API | `app/api/admin/integrations/spruce/subscription/route.ts` |
| Appointments API | `app/api/appointments/spruce/route.ts` |
| Admin UI | `app/admin/integrations/spruce/page.tsx` |
| OpenAPI SDK | `apis/spruce/index.ts` |
| OpenAPI spec | `apis/spruce/openapi.json` |
| Message model | `models/Message.ts` |
| Conversation model | `models/Conversation.ts` |
| ConversationItem model | `models/ConversationItem.ts` |
| Message source enum | `types/messageDTO.ts` |
| Feature flags | `utils/featureFlags/featureFlags.ts` |
