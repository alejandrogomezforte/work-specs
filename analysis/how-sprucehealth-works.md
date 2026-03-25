# How Spruce Health Works in li-call-processor

## Overview

Spruce Health is integrated for three purposes: **SMS messaging** to patients, **fax document intake**, and **conversation tracking**. The integration uses an OpenAPI-generated SDK client and manages credentials through Azure Key Vault.

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

| Source | Feature Flag | Description |
|--------|-------------|-------------|
| `REMINDER` | `APPOINTMENT_REMINDER` | Daily SMS appointment reminders (9 AM EST) |
| `WHAT_TO_EXPECT` | `SEND_WHAT_TO_EXPECT` | Pre-appointment info messages |
| `SYMPTOM` | `SEND_SYMPTOM` | Symptom check-in messages |

### Key file

`apps/web/services/messages/messageService.ts` -> `sendSpruceSms()`

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

After an SMS or fax is processed (confirmed by webhook), the app can perform cleanup:

### Internal notes

Uses `POST /v1/conversations/{conversationId}/messages` with `internal: true`:
- SDK method: `spruceApi.postConversationMessage()`
- Posts "Processed by LISA" note to the conversation

### Archiving

Uses `PATCH /v1/conversations/{conversationId}` with `archived: true`:
- SDK method: `spruceApi.updateConversation()`

### Feature flags controlling these actions

| Flag | Default | Purpose |
|------|---------|---------|
| `SPRUCE_SMS_REMINDER_INTERNAL_NOTE` | `false` | Post note to SMS reminder conversations |
| `SPRUCE_SMS_REMINDER_ARCHIVE` | `false` | Archive SMS reminder conversations |
| `SPRUCE_FAX_INTERNAL_NOTE` | `false` | Post note to fax conversations |
| `SPRUCE_FAX_ARCHIVE` | `false` | Archive fax conversations |

### Key file

`apps/web/services/integrations/spruce.ts` -> `postSpruceInternalNote()`, `archiveSpruceConversation()`

---

## 5. Admin Configuration

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

## 6. Spruce API Endpoints Used

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

## 7. Data Models

### Message model (`models/Message.ts`)

Spruce-specific fields:
```typescript
spruce?: {
  requestId: string;      // API response ID (from postMessageFromEndpoint)
  txt?: string;           // Message text
  phone?: string;         // Phone number
  conversationId?: string; // Populated by webhook callback
  appUrl?: string;        // Conversation URL in Spruce app
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

## 8. OpenAPI SDK

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

## 9. Service Layer Functions

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

## 10. Feature Flags

| Flag | Default | Purpose |
|------|---------|---------|
| `LOGS_FOR_SPRUCE_CONVERSATION` | `true` | Azure Blob logging of webhook payloads |
| `SPRUCE_FAX_ARCHIVE` | `false` | Archive fax conversations after processing |
| `SPRUCE_FAX_INTERNAL_NOTE` | `false` | Post internal note to fax conversations |
| `SPRUCE_SMS_REMINDER_INTERNAL_NOTE` | `false` | Post "Processed by LISA" note to SMS conversations |
| `SPRUCE_SMS_REMINDER_ARCHIVE` | `false` | Archive SMS reminder conversations |
| `APPOINTMENT_REMINDER` | `false` | Enable appointment reminder SMS sending |
| `SEND_WHAT_TO_EXPECT` | `false` | Enable "what to expect" SMS messages |
| `SEND_SYMPTOM` | `false` | Enable symptom check-in SMS messages |
| `INTAKE_PROCESSING` | `true` | Process intakes after fax document upload |

---

## 11. Configuration & Secrets

### Environment variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `SPRUCE_API_BASE_URL` | `https://api.sprucehealth.com` | API endpoint |
| `AZURE_STORAGE_CONNECTION_STRING` | - | Blob storage for faxes |
| `FAX_CONTAINER_NAME` | `incoming-faxes` | Azure container name |

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

## 12. Data Flow Diagram

```
                    OUTBOUND                              INBOUND
              (App -> Spruce)                        (Spruce -> App)

┌──────────────────┐                              ┌──────────────────┐
│  sendMessageJob  │                              │  Spruce Webhook  │
│  (Pulse)         │                              │  Events          │
└────────┬─────────┘                              └────────┬─────────┘
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
└──────────────────┘                              └────────┬─────────┘
                                                           │
                                              ┌────────────┼────────────┐
                                              │            │            │
                                              v            v            v
                                        ┌──────────┐ ┌─────────┐ ┌─────────┐
                                        │ Upsert   │ │ Link to │ │ Schedule│
                                        │ Convo/   │ │ Message │ │ Intake  │
                                        │ Item doc │ │ doc     │ │ Job     │
                                        └──────────┘ └─────────┘ └────┬────┘
                                                                      │
                                                                      v
                                                                ┌─────────────┐
                                                                │ Download PDF│
                                                                │ Create      │
                                                                │ Intake doc  │
                                                                │ Upload to   │
                                                                │ Azure Blob  │
                                                                └─────────────┘
```

---

## 13. Key Files Reference

| Category | File Path (relative to apps/web/) |
|----------|-----------------------------------|
| Service layer | `services/integrations/spruce.ts` |
| SMS sending | `services/messages/messageService.ts` |
| Webhook endpoint (active) | `pages/api/webhooks/spruce_conversations.ts` |
| Webhook endpoint (legacy stub) | `app/api/webhooks/spruce/route.ts` |
| Webhook handler | `services/webhooks/handlers/spruce_conversation.ts` |
| Fax intake job | `services/jobs/definitions/spruceIntake/spruceIntakeJob.ts` |
| Fax storage job | `services/jobs/definitions/spruceFax.ts` |
| Conversation tracking job | `services/jobs/definitions/spruceConversation.ts` |
| Send message job | `services/jobs/definitions/sendMessageJob/sendMessageJob.ts` |
| Admin API | `app/api/admin/integrations/spruce/route.ts` |
| Subscription API | `app/api/admin/integrations/spruce/subscription/route.ts` |
| Appointments API | `app/api/appointments/spruce/route.ts` |
| Admin UI | `app/admin/integrations/spruce/page.tsx` |
| OpenAPI SDK | `apis/spruce/index.ts` |
| OpenAPI spec | `apis/spruce/openapi.json` |
| Message model | `models/Message.ts` |
| Conversation model | `models/Conversation.ts` |
| ConversationItem model | `models/ConversationItem.ts` |
| Feature flags | `utils/featureFlags/featureFlags.ts` |
