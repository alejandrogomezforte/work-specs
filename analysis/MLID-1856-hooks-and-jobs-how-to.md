# MLID-1856 --- Webhooks & Jobs: How to Test Locally

## Overview

This document explains how the appointment webhook and job trigger endpoints work in `li-call-processor`, and how to test the checkout SMS flow locally.

---

## 1. `POST /api/webhooks/looker_appointments`

**Stage URL**: `https://app-stage.mylocalinfusion.ai/api/webhooks/looker_appointments`
**Local URL**: `http://localhost:8080/api/webhooks/looker_appointments`

**Route file**: `apps/web/pages/api/webhooks/looker_appointments.ts` (Pages Router)

**Authentication**: Requires header `x-looker-webhook-token` validated against the `LOOKER_WEBHOOK_TOKEN` env var (set in `.env.local`). Returns 403 if missing/invalid.

- Validator located at: `apps/web/services/webhooks/middleware/lookerAuth.ts`

**What it does**:

1. Validates token + POST method
2. Optionally stores payload to Azure Blob Storage (if feature flag `LOGS_FOR_LOOKER_APPOINTMENTS` is enabled)
3. Calls `webhookUpsertAppointments(payload)` which processes the appointment data
4. If an appointment is **new with status "complete"** or **changes to "complete"** (from "active" or "rescheduled"):
   - Calls `setSymptomMessage(appointmentId, undefined, patientId)`
   - Creates a `Message` doc in MongoDB with:
     - `source: 'symptom'`
     - `status: READY`
     - `dateToSend`: next business day at 1 PM UTC (9 AM ET)

**Key files in the flow**:

| File | Purpose |
|------|---------|
| `apps/web/pages/api/webhooks/looker_appointments.ts` | Route entry point |
| `apps/web/services/webhooks/handlers/looker_appointments.ts` | Handler (calls upsert) |
| `apps/web/services/webhooks/handlers/util/webhookUpsertAppointments.ts` | Appointment upsert logic + setSymptomMessage caller |
| `apps/web/services/messages/messageService.ts` | `setSymptomMessage()` creates the Message doc |

---

## 2. `POST /api/jobs/trigger`

**Stage URL**: `https://app-stage.mylocalinfusion.ai/api/jobs/trigger`
**Local URL**: `http://localhost:8080/api/jobs/trigger`

**Route file**: Exists in both:
- `apps/web/app/api/jobs/trigger.ts` (App Router)
- `apps/web/pages/api/jobs/trigger.ts` (Pages Router)

**Authentication**: **None** --- no auth required.

**Body**:

```json
{
  "name": "appointmentReminder",
  "data": { "force": true }
}
```

- `name` (required): The job name to trigger
- `data` (optional): Data to pass to the job handler (`force: true` bypasses the feature flag check)
- `options` (optional): `{ delay, priority }` for scheduling later

**What it does**:

1. If `options.delay` is specified, schedules the job for later via `scheduleJobForLater()`
2. Otherwise, calls `scheduleImmediateJob()` for immediate execution
3. Returns `{ success, message, jobId }`

**What the `appointmentReminder` job does** (when triggered):

1. Checks if `APPOINTMENT_REMINDER` feature flag is enabled (bypassed with `force: true`)
2. Queries appointments scheduled 2-3 days in the future via `pendingAppointments()` cursor
3. For each pending appointment, calls `setAppointmentReminderMessage(appointmentId)`
4. Creates `Message` docs with `source: 'reminder'`, `status: READY`, `dateToSend: today 1 PM UTC`

**Key files**:

| File | Purpose |
|------|---------|
| `apps/web/app/api/jobs/trigger.ts` | Route entry point |
| `apps/web/services/jobs/definitions/appointmentReminder.ts` | Job definition |
| `apps/web/services/jobs/scheduler.ts` | Scheduling utilities + recurring job setup |

**Note**: The `appointmentReminder` job also runs automatically on a daily cron at 9 AM EST (`0 9 * * *`), defined in `scheduler.ts`.

---

## 3. The Actual SMS Sender: `sendMessageJob`

Neither endpoint above sends the SMS directly. They only create `Message` docs in MongoDB. The **`sendMessageJob`** handles delivery:

- **Schedule**: Runs every 2 minutes (`*/2 * * * *`)
- **Location**: `apps/web/services/jobs/definitions/sendMessageJob/sendMessageJob.ts`

**What it does**:

1. Finds all `Message` docs with `status: READY` and `dateToSend` within today
2. Only sends during work hours (9 AM - 3 PM ET) unless `immediately: true`
3. Skips holidays for SYMPTOM and WHAT_TO_EXPECT messages (not REMINDER)
4. For each message, calls the appropriate template:
   - **SYMPTOM** messages -> `symptomTextMessageFromTemplate()`
   - **REMINDER** messages -> `messageFromTemplate()`
   - **WHAT_TO_EXPECT** messages -> `whatToExpectMessageFromTemplate()`
5. Calls `sendSpruceSms()` -> hits the Spruce API to deliver the SMS
6. Updates Message status to SENT (or DECLINED/ERROR on failure)
7. Retries up to 5 times on failure

**Key files**:

| File | Purpose |
|------|---------|
| `apps/web/services/jobs/definitions/sendMessageJob/sendMessageJob.ts` | Job that processes and sends messages |
| `apps/web/services/jobs/definitions/appointmentReminder/messageTemplate.ts` | Message text templates |
| `apps/web/services/messages/messageService.ts` | `sendSpruceSms()` Spruce API call |

---

## 4. Spruce Archival (Post-Send)

After an SMS is sent via Spruce and the recipient responds, Spruce sends a webhook callback:

**Endpoint**: `POST /api/webhooks/spruce_conversation`

**What happens**:

1. `handleSpruceConversation()` is called
2. `archiveSmsConversation()` checks if the conversation is linked to a REMINDER or SYMPTOM message
3. If found and feature flags `SPRUCE_SMS_REMINDER_INTERNAL_NOTE` + `SPRUCE_SMS_REMINDER_ARCHIVE` are enabled:
   - Posts a "Processed by LISA" internal note
   - Archives the Spruce conversation (2-second delay between note and archive)

**Key files**:

| File | Purpose |
|------|---------|
| `apps/web/services/webhooks/handlers/spruce_conversation.ts` | Archival logic |

---

## Complete Flow Diagram

```
[Looker sends appointment data]
        |
        v
POST /api/webhooks/looker_appointments (requires x-looker-webhook-token)
        |
        v
webhookUpsertAppointments()
        |
        v
[If appointment status = "complete"]
        |
        v
setSymptomMessage() -> creates Message doc (source=symptom, status=READY, dateToSend=next biz day 1PM UTC)
        |
        v
sendMessageJob (runs every 2 min) picks it up when dateToSend arrives
        |
        v
symptomTextMessageFromTemplate() -> sendSpruceSms() -> SMS delivered via Spruce API
        |
        v
[Patient/Spruce responds]
        |
        v
POST /api/webhooks/spruce_conversation -> archiveSmsConversation() -> archives conversation
```

---

## Local Testing Guide

### Prerequisites

1. Start the web server: `npm run dev:web` (port 8080, also starts the job worker)
2. Ensure `.env.local` has:
   - `MONGODB_URI` pointing to a test database
   - `LOOKER_WEBHOOK_TOKEN` for webhook auth
   - `SPRUCE_API_KEY` (only if you want SMS to actually send)

### Testing Each Piece

| What to test | How | Notes |
|---|---|---|
| **Message template** | Already covered by unit tests (`messageTemplate.spec.ts`) | `npm run test -- messageTemplate` |
| **Guard removal** | POST the webhook with a patient that already has completed appointments | Verify a new Message doc is created in MongoDB (no longer blocked) |
| **Job trigger** | `POST http://localhost:8080/api/jobs/trigger` with `{"name": "appointmentReminder", "data": {"force": true}}` | No auth needed |
| **Archival** | Harder locally --- needs a real Spruce conversation callback | Best tested in staging |
| **UI labels** | Start dev server, navigate to Communications page | Verify "checkout" shows instead of "symptom" |

### Example cURL Commands

**Trigger the appointment reminder job**:

```bash
curl -X POST http://localhost:8080/api/jobs/trigger \
  -H "Content-Type: application/json" \
  -d '{"name": "appointmentReminder", "data": {"force": true}}'
```

**Send an appointment webhook** (replace token with your local value):

```bash
curl -X POST http://localhost:8080/api/webhooks/looker_appointments \
  -H "Content-Type: application/json" \
  -H "x-looker-webhook-token: YOUR_TOKEN_HERE" \
  -d '[{ ... appointment payload ... }]'
```

### Important Notes

- **SMS won't actually send** unless your local env has a valid Spruce API token in Azure Key Vault (secret name: `spruce-api-token`). There is no `SPRUCE_API_KEY` env var --- the token is fetched at runtime from the vault configured in `AZURE_KEY_VAULT_URL`.
- **sendMessageJob only sends during 9 AM - 3 PM ET** --- outside those hours, messages stay in READY status
- **dateToSend matters** --- a symptom message created today has `dateToSend` set to the next business day, so `sendMessageJob` won't pick it up until then (unless you manually update the MongoDB doc)

---

## 5. How `sendMessageProcessing` Works

When you `POST /api/jobs/trigger` with `{ "name": "sendMessageProcessing" }`, here is the full chain of events:

### Step 1: Route handler creates a Pulse job doc

- **Route file**: `apps/web/pages/api/jobs/trigger.ts`
- The handler calls `scheduleImmediateJob("sendMessageProcessing", {})`
- This creates a document in the Pulse MongoDB collection (configured by `PULSE_COLLECTION` env var, e.g. `pulseJobsAGomez`) with `name: "sendMessageProcessing"` and `nextRunAt: now`
- **Important**: The route does NOT run the job itself --- it only enqueues it

### Step 2: Pulse worker picks up the job

- The Pulse worker runs in the background as part of `npm run dev:web`
- It polls the Pulse collection and picks up jobs where `nextRunAt` is in the past
- It invokes the registered handler: `sendMessageJob.ts`

### Step 3: `sendMessageJob` processes ready messages

**File**: `apps/web/services/jobs/definitions/sendMessageJob/sendMessageJob.ts`

For each message source (REMINDER, SYMPTOM, WHAT_TO_EXPECT, etc.), the job:

1. **Queries messages** --- finds all `Message` docs with `status: "ready"` and `dateToSend` within today's date range
2. **Checks feature flags** --- e.g. `SEND_SYMPTOM` must be enabled for symptom messages (bypassed if `force: true` is in job data)
3. **Filters by work hours** --- `isWorkHours()` only allows sending between 9 AM--3 PM ET, unless `message.immediately` is set
4. **Skips holidays** --- SYMPTOM and WHAT_TO_EXPECT messages are skipped on holidays (not REMINDER)
5. **Checks whitelist** --- `isAllowedToSend()` matches patient's `mobile_phone` against `APPOINTMENT_REMINDER_TARGET_CRITERIA` env var. If set to `*`, all numbers are allowed. Otherwise, only comma-separated numbers in the list are allowed.
6. **Generates message text** --- calls the appropriate template function:
   - SYMPTOM → `symptomTextMessageFromTemplate()`
   - REMINDER → `messageFromTemplate()`
   - WHAT_TO_EXPECT → `whatToExpectMessageFromTemplate()`
7. **Sends via Spruce** --- `sendSpruceSms()` authenticates with the Spruce API using a token from Azure Key Vault (secret: `spruce-api-token`), then delivers the SMS
8. **Updates message status** --- changes from `ready` to `sent` on success, or `error` on failure (retries up to 5 times)

### Key files

| File | Purpose |
|------|---------|
| `apps/web/pages/api/jobs/trigger.ts` | Route --- enqueues the job in Pulse |
| `apps/web/services/jobs/scheduler.ts` | `scheduleImmediateJob()` creates the Pulse job doc |
| `apps/web/services/jobs/definitions/sendMessageJob/sendMessageJob.ts` | Job handler --- queries messages, filters, sends SMS |
| `apps/web/services/jobs/definitions/appointmentReminder/isAllowedToSend.ts` | Whitelist check against `APPOINTMENT_REMINDER_TARGET_CRITERIA` |
| `apps/web/services/jobs/definitions/appointmentReminder/messageTemplate.ts` | SMS text templates |
| `apps/web/services/messages/messageService.ts` | `sendSpruceSms()` --- Spruce API call |
| `apps/web/services/integrations/spruce.ts` | Spruce API client --- auth via Key Vault |

### Testing locally

```bash
curl -X POST http://localhost:8080/api/jobs/trigger \
  -H "Content-Type: application/json" \
  -d '{"name": "sendMessageProcessing"}'
```

No `force: true` needed --- the job doesn't use a `force` flag. Work hours and feature flags still apply.
