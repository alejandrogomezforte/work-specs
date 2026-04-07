# MLID-2011 — Orders: Document Notifications

## Epic Workflow

This template follows the [SW Workflow](../dx/sw-workflow.md). For each sub-task, run the workflow cycle once and update this document to track progress. Read this document first to understand overall context before starting each cycle.

For the git branch strategy, see [epic-git-branch-strategy.md](../dx/epic-git-branch-strategy.md).

---

## Purpose

Provide real-time visibility to Infusion Guides when new documents are added to active orders. Currently document arrival is communicated via Spruce (the patient messaging app), creating noise that competes with patient messages (which have a ~2hr SLA). This epic replaces that with in-app notifications powered by a real-time messaging layer, so Infusion Guides can immediately acknowledge new documents and progress orders toward the ~3-day "Ready to Schedule" target.

---

## Architecture Decision

**Azure SignalR Service** — The messaging layer will be built using Azure SignalR Service. The previous plan to use raw Socket.IO was rejected due to the complexity of orchestrating WebSockets in Azure infrastructure. Azure SignalR provides a managed real-time broadcast service that handles scaling, connection management, and Azure-native integration out of the box.

**SignalR is a 3rd-party service the webapp integrates with** — like Spruce or WeInfuse. The webapp is the SignalR client, not individual users. Connection string / API token stored in Azure Key Vault. No per-user authentication to SignalR. The server broadcasts events to all connected browsers; each browser decides if the event is relevant to its current view.

This epic builds a **reusable SignalR integration** for real-time broadcast, then uses it to deliver document notifications. The SignalR layer is generic — future features (intakes, location changes, etc.) can reuse it to broadcast events to all connected clients.

---

## Messaging Layer: Azure SignalR

### How It Works

```
┌──────────────┐                                        ┌──────────────────┐
│   Browser A  │ ◄──── 3. Receive broadcast ───────┐    │  Next.js API     │
│   (Bob)      │                                   │    │  (document       │
└──────────────┘                                   │    │   triggers)      │
                                              ┌────┴──┐ │                  │
┌──────────────┐                              │ Azure │ │ 1. Document      │
│   Browser B  │ ◄──── 3. Receive broadcast ──┤SignalR│◄┤    created/      │
│   (Lisa)     │                              │Service│ │    acknowledged  │
└──────────────┘                              └────┬──┘ │                  │
                                                   │    │ 2. REST API:     │
┌──────────────┐                                   │    │    POST broadcast│
│   Browser C  │ ◄──── 3. Receive broadcast ───────┘    └──────────────────┘
│   (anyone)   │
└──────────────┘
```

1. A document is created or acknowledged → the server-side trigger fires
2. Server calls SignalR REST API to **broadcast** to all connected clients (no user targeting, no rooms)
3. All connected browsers receive the event. Each client checks if the event is relevant to its current view and re-fetches data if needed.

### Key Design Points

- **App-level integration** — SignalR is a 3rd-party service. The webapp authenticates to SignalR using a connection string stored in Azure Key Vault. Individual users do NOT authenticate to SignalR.
- **Rooms as topic channels** — Rooms are used to categorize broadcasts by feature area (like logging categories). This epic uses a room called `orders`. Clients viewing order-related pages join the `orders` room and only receive order-related events. Future features (intakes, locations, etc.) get their own rooms.
- **Notifications are transient** — Fire and forget. Nothing persisted. Source of truth is always MongoDB (the `documents` collection).
- **Client SDK** — `@microsoft/signalr` npm package. Handles transport negotiation (WebSocket → SSE → Long Polling), auto-reconnection, and connection lifecycle.
- **Hub** — Single hub named `notifications`. Generic — reusable for any future real-time broadcast needs.
- **Connection string** — Stored in Azure Key Vault (secret name TBD by Anatoliy), accessed via existing Key Vault integration. Local dev uses `DEV_SECRET_` fallback.
- **Existing Socket.IO** — Coexists. Call-status events (`callStatusChanged`, `globalCallStatusChanged`) remain on Socket.IO. No migration in this epic.

---

## Document Acknowledgement: `acknowledged` flag on Document model

**No new collections.** The acknowledgement state lives directly on the existing `documents` collection as a simple boolean flag. When any user clicks "Mark as Read", the document is marked as acknowledged globally.

### Schema change on `documents` collection

```typescript
// New fields added to IDocument interface and DocumentSchema
{
  acknowledged: boolean;      // default: false — has anyone marked this as read?
  acknowledgedAt?: Date;      // when it was acknowledged
  acknowledgedBy?: string;    // userId who acknowledged it (audit trail)
}
```

### How it drives the UI

| UI Element | Query |
|---|---|
| "New" tag on document row | `document.acknowledged === false` |
| "Mark as Read" button shown | `document.acknowledged === false` |
| Documents tab badge (unread count) | `count(documents where patientId=X AND acknowledged=false)` |
| Order Tracker badge | Same count, aggregated per order via patient link |
| "Mark all as Read" | `updateMany({ patientId, acknowledged: false }, { acknowledged: true, acknowledgedAt, acknowledgedBy })` |

### Why this works

- **Simple** — no new collection, no per-user state, no notification persistence
- **Generic** — if future features need acknowledgement (intakes, etc.), same pattern: add a flag to the domain entity
- **Source of truth** — the document IS the state. UI reads from existing collections. SignalR just says "hey, go re-check."

---

## Proposed Architecture

### Core Principles

1. **SignalR = transient push alerts** — Fire and forget. No persistence. Just tells connected clients "something changed, re-fetch if relevant."
2. **Existing collections = source of truth** — `documents.acknowledged` drives the UI state. No notifications collection.
3. **REST for mutations** — "Mark as Read" calls existing document API routes to flip `acknowledged: true`. SignalR broadcasts the change.
4. **Feature-flag gated** — `DOCUMENT_NOTIFICATIONS` flag gates everything at API + UI level.

### Layer Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        FRONTEND LAYERS                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  UI Components (D3)                                     │    │
│  │  Order Tracker badges, Documents tab, History tab       │    │
│  └──────────────────────┬──────────────────────────────────┘    │
│                         │ uses                                  │
│  ┌──────────────────────▼──────────────────────────────────┐    │
│  │  SignalRContext (new, generic)                           │    │
│  │  - @microsoft/signalr HubConnection management          │    │
│  │  - Connects to SignalR using app-level connection info   │    │
│  │  - Auto-reconnect with withAutomaticReconnect()         │    │
│  │  - Connection status (connected/disconnected/reconnect) │    │
│  │  - Exposes .on(event, handler) for any consumer         │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  On SignalR event received → component re-fetches data from     │
│  existing REST APIs (order details, documents list, etc.)       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                              │
             REST (queries + mutations)  +  SignalR (broadcast)
                              │
┌─────────────────────────────────────────────────────────────────┐
│                        BACKEND LAYERS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  API Routes (REST) — existing + new                     │    │
│  │  PATCH /api/documents/:id/acknowledge  — mark one read  │    │
│  │  PATCH /api/documents/acknowledge-bulk — mark all read  │    │
│  │  GET  /api/orders/details              — already exists  │    │
│  └──────────────────────┬──────────────────────────────────┘    │
│                         │ calls                                 │
│  ┌──────────────────────▼──────────────────────────────────┐    │
│  │  SignalR Broadcast Service (services/signalr/)          │    │
│  │  - Generic: broadcastToRoom(room, event, payload)       │    │
│  │  - Authenticates using connection string from Key Vault │    │
│  │  - Calls SignalR REST API to broadcast to a room        │    │
│  │  - Rooms = topic channels (e.g., 'orders', 'intakes')  │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Document Triggers (entry points)                       │    │
│  │  - Document webhook handlers                            │    │
│  │  - Document upload routes                               │    │
│  │  - On document creation: broadcast 'documentAdded'      │    │
│  │  - On acknowledge: broadcast 'documentAcknowledged'     │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Document Model (existing, enhanced)                    │    │
│  │  - Added: acknowledged, acknowledgedAt, acknowledgedBy  │    │
│  │  - New docs default to acknowledged: false              │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### SignalR Events (server → client, transient)

| Event (target) | Payload | When |
|-----------------|---------|------|
| `documentAdded` | `{ documentId, patientId, orderId, category, source }` | New document created/uploaded |
| `documentAcknowledged` | `{ documentId, patientId }` | Document marked as read |

**These are fire-and-forget.** Client receives the event and re-fetches relevant data from existing REST APIs. If the client misses an event (offline), the next page load or API call will have the correct state from MongoDB.

### Data Flow: New Document Added

```
Document uploaded (webhook/upload)
  │
  ▼
Document Trigger (in webhook handler / upload route)
  │ 1. Saves document to MongoDB (acknowledged: false)
  │ 2. Calls signalrClient.broadcastToRoom('orders', 'documentAdded', {
  │      documentId, patientId, orderId, category, source
  │    })
  │
  ▼
signalrClient.broadcastToRoom()
  │ POST https://<instance>.service.signalr.net/api/v1/hubs/notifications/groups/orders
  │ Body: { "target": "documentAdded", "arguments": [payload] }
  │
  ▼
Azure SignalR Service → all clients in the 'orders' room
  │
  ▼
Client (listening via SignalRContext)
  │ connection.on('documentAdded', handler)
  │ If relevant to current view → re-fetch order details / documents list
  │ UI updates: badge appears, "New" tag shows on document row
```

### Data Flow: Mark as Read

```
User clicks "Mark as Read" on a document
  │
  ▼
PATCH /api/documents/:id/acknowledge
  │ 1. Updates document: { acknowledged: true, acknowledgedAt, acknowledgedBy }
  │ 2. Calls signalrClient.broadcastToRoom('orders', 'documentAcknowledged', { documentId, patientId })
  │
  ▼
All clients in 'orders' room receive event → re-fetch if on that order's view
  │ "New" tag disappears, badge count decrements
```

### New File Structure

```
apps/web/
├── services/
│   └── signalr/
│       └── signalrClient.ts          — Server-side broadcast client (generic, reusable)
│                                        Uses connection string from Key Vault
├── types/
│   └── signalr.ts                    — SignalR event types (generic)
├── app/api/
│   └── documents/
│       ├── [id]/
│       │   └── acknowledge/
│       │       └── route.ts          — PATCH: mark one document as read
│       └── acknowledge-bulk/
│           └── route.ts              — PATCH: mark all patient docs as read
└── utils/context/
    └── signalr-context.tsx           — SignalR HubConnection provider + hook (generic)
```

---

## Phases

### Phase 0: Azure SignalR Infrastructure (prerequisite)
Provision and configure the Azure SignalR Service resource, secrets, and environment configuration. Owned by Anatoliy (Principal Engineer). This is a blocker for all other phases.

### Phase A: SignalR Integration (foundation)
Application-level SignalR integration: server-side broadcast client (uses connection string from Key Vault) + client-side SDK context. Generic and reusable — not tied to documents or notifications specifically. No per-user auth to SignalR.

### Phase B: Document Notifications (feature)
Build the document notification experience: `acknowledged` flag on documents, trigger SignalR events on document creation, badges in Order Tracker, acknowledgement flow in Documents tab, audit trail in Order History.

---

## Deliverable 0: Azure SignalR Infrastructure (Phase 0) — Assigned to Anatoliy

**Goal:** Provision the Azure SignalR Service resource and make the connection string available to the application in all environments (dev, staging, prod). This is a prerequisite for all application-level work. No application code changes — infrastructure and configuration only.

**Owner:** Anatoliy (Principal Engineer)

| Task ID | Summary | SP | Status | Branch | Merged to Epic | Notes |
|---------|---------|:--:|:------:|--------|:--------------:|-------|
| D0-T1 | Provision Azure SignalR Service + Key Vault configuration + connectivity validation | 3 | To Do | - | - | [MLID-2079](https://localinfusion.atlassian.net/browse/MLID-2079) |

**D0-T1 Requirements:**
- Provision Azure SignalR Service resource in **Serverless mode** in dev, staging, and prod environments
  - Free tier for dev, Standard for staging/prod
  - Region should match existing Azure resources
- Store the SignalR connection string in **Azure Key Vault** in all environments
  - Add `DEV_SECRET_` fallback env var for local development
- Configure **CORS** on the SignalR resource to allow app origins
  - Dev: `localhost:8080`
  - Staging/Prod: the app domain(s)
- Validate end-to-end connectivity with a quick **smoke test** from Node.js (negotiate → connect → receive message)

**D0-T1 Acceptance criteria:**
- SignalR resource is provisioned in Serverless mode in all environments
- Connection string is accessible via Azure Key Vault (and local `DEV_SECRET_` fallback)
- CORS allows the app origin in each environment
- A simple smoke test confirms negotiate → connect → receive message works from Node.js

**Open questions for Anatoliy:**
1. **Key Vault secret name** — What naming convention should we follow? Suggestion: `azure-signalr-connection-string`. The app code will reference this name to fetch the connection string.
2. **DEV_SECRET fallback variable name** — Suggestion: `DEV_SECRET_azure_signalr_connection_string` (matching the existing `DEV_SECRET_*` pattern). Confirm or adjust.
3. **Pricing tier for staging** — Free tier has limits (20 concurrent connections, 20K messages/day). Is Standard needed for staging, or is Free sufficient for testing?
4. **Region** — Which Azure region are the existing resources in? The SignalR resource should be co-located.
5. **Service mode** — We plan to use the service as a broadcast channel (server sends to all clients). What's the recommended mode (Default vs Serverless) for this pattern?
6. **Existing Socket.IO coexistence** — The app currently uses Socket.IO for call-status events (`callStatusChanged`, `globalCallStatusChanged`). Plan is to keep both running in parallel. Any concerns from an infra perspective?
7. **Hub naming** — We plan to use a single hub called `notifications`. Is there a naming convention or restriction on the Azure side?
8. **Connection limits** — Standard tier supports 1K concurrent connections per unit. How many concurrent users do we expect in prod? Do we need multiple units?
9. **Client connection setup** — How should browser clients connect to the SignalR service? The app (not individual users) authenticates to SignalR. Need guidance on the client SDK connection pattern that works with this model.

**D0 PR to develop:** Not needed (infrastructure only, no app code)

---

## Deliverable 1: SignalR Integration (Phase A)

**Goal:** Generic, reusable SignalR integration — server-side broadcast client + client-side React context. Testable: call broadcast from server → all connected clients receive the event. Not tied to documents or any specific feature.

**Depends on:** D0 (SignalR resource must be provisioned and connection string available)

| Task ID | Summary | SP | Status | Branch | Merged to Epic | Notes |
|---------|---------|:--:|:------:|--------|:--------------:|-------|
| D1-T1 | Feature flag `DOCUMENT_NOTIFICATIONS` + SignalR server-side broadcast client | 3 | To Do | - | - | [MLID-2080](https://localinfusion.atlassian.net/browse/MLID-2080) |
| D1-T2 | Frontend: SignalR React context + provider (`SignalRContext`) | 3 | To Do | - | - | [MLID-2081](https://localinfusion.atlassian.net/browse/MLID-2081) |

**D1 PR to develop:** Not yet

---

## Deliverable 2: Document Notifications + Acknowledgement (Phase B)

**Goal:** Add `acknowledged` flag to the Document model. Wire document creation entry points to broadcast SignalR events. Add acknowledge API routes. Testable: upload a document → it's saved with `acknowledged: false` → SignalR event fires → calling acknowledge route flips the flag → SignalR event fires again.

**Depends on:** D1

| Task ID | Summary | SP | Status | Branch | Merged to Epic | Notes |
|---------|---------|:--:|:------:|--------|:--------------:|-------|
| D2-T1 | `acknowledged` flag on Document model + acknowledge API routes | 3 | To Do | - | - | [MLID-2082](https://localinfusion.atlassian.net/browse/MLID-2082) |
| D2-T2 | Investigate document creation entry points + emit `documentAdded` events | 5 | To Do | - | - | [MLID-2083](https://localinfusion.atlassian.net/browse/MLID-2083) |

**D2 PR to develop:** Not yet

---

## Deliverable 3: Notification UI — Order Tracker, Documents Tab, Order History (Phase B)

**Goal:** Full notification UI experience across all order views. (1) Order Tracker — badges and "Action Required" filter based on unacknowledged documents, (2) Documents tab — "New" tags, "Mark as Read" (single + bulk), unread badge on tab header, (3) Order History — audit trail for document additions. All update in real-time via SignalR events.

**Depends on:** D2

| Task ID | Summary | SP | Status | Branch | Merged to Epic | Notes |
|---------|---------|:--:|:------:|--------|:--------------:|-------|
| D3-T1 | Order Tracker: notification badge on Order ID + "Action Required" quick filter | 5 | To Do | - | - | [MLID-2084](https://localinfusion.atlassian.net/browse/MLID-2084) |
| D3-T2 | Documents Tab: unread badge, "New" tags, "Mark as Read" (single + bulk) | 5 | To Do | - | - | [MLID-2085](https://localinfusion.atlassian.net/browse/MLID-2085) |
| D3-T3 | Order History: document audit trail entries with source differentiation | 3 | To Do | - | - | [MLID-2086](https://localinfusion.atlassian.net/browse/MLID-2086) |

**D3 PR to develop:** Not yet

---

## Task List

### D0: Azure SignalR Infrastructure (3 SP) — Anatoliy
- D0-T1 — MLID-2079 — Provision SignalR resource + Key Vault + CORS + smoke test (3 SP)

### D1: SignalR Integration (6 SP)
- D1-T1 — MLID-2080 — Feature flag + SignalR server-side broadcast client (3 SP)
- D1-T2 — MLID-2081 — Frontend: SignalR React context + provider (3 SP)

### D2: Document Notifications + Acknowledgement (8 SP)
- D2-T1 — MLID-2082 — `acknowledged` flag on Document model + acknowledge API routes (3 SP)
- D2-T2 — MLID-2083 — Investigate entry points + emit `documentAdded` events (5 SP)

### D3: Notification UI (13 SP)
- D3-T1 — MLID-2084 — Order Tracker: badges + "Action Required" filter (5 SP)
- D3-T2 — MLID-2085 — Documents Tab: "New" tags + "Mark as Read" (single + bulk) (5 SP)
- D3-T3 — MLID-2086 — Order History: audit trail entries (3 SP)

---

## Dependency Graph

```
D0 (Azure SignalR Infrastructure — Anatoliy)
 └── D1 (SignalR Integration) ── depends on D0
      └── D2 (Document Notifications + Acknowledgement) ── depends on D1
           └── D3 (Notification UI) ── depends on D2
```

---

## Design Reference

- **Orders list view:** Notification badge/icon next to Order ID for orders with unread documents
- **Order detail — Documents tab:**
  - Badge on "Documents" tab header showing unread count (e.g., circled "1")
  - "New" chip/tag next to received date on unread documents
  - "Mark as Read" button per document row (only on unread)
  - "Mark all as Read" button at top-right of documents section
- **Order detail — Order History tab:** Audit entries for document additions with source info
- **Designer specs:** `li-playground.vercel.app/intakes`
- **Prototype HTML:** Attached to MLID-2011 in Jira

---

## Epic Reference

- **Jira**: [MLID-2011](https://localinfusion.atlassian.net/browse/MLID-2011)
- **Total Story Points**: 30
- **Epic Branch**: `epic/MLID-2011-document-notifications`
- **Deliverables**: 4 (D0–D3)
- **Total Tasks**: 8

---

## Non-Code Tasks

| Task ID | Summary | Status |
|---------|---------|:------:|
| - | Review designer specs at li-playground and prototype HTML | To Do |
| - | Codebase investigation: document upload flows, order tracker pipeline | To Do |
| - | Estimate story points per task after investigation | To Do |
| - | Create Jira sub-tasks once plan is approved | To Do |

---

## Open Questions

1. **Existing document upload paths** — Need to catalog all entry points where documents get attached to orders (webhooks, manual upload, fax integration, e-order). Investigation task D2-T2.
2. **Order History tab** — Does it already exist? What's the current audit trail mechanism? Need to investigate.
3. **Key Vault secret name** — Anatoliy to confirm naming convention for the SignalR connection string secret.

## Resolved Questions

- **SignalR integration model** — The webapp is the SignalR client, not individual users. Connection string in Key Vault. Server broadcasts to rooms (topic channels). No per-user auth, no negotiate endpoint.
- **Rooms as topic channels** — Rooms categorize broadcasts by feature area (like logging). `orders` for this epic. Future: `intakes`, `locations`, etc. Clients join the room relevant to their current view.
- **Notification persistence** — No. Notifications are transient SignalR broadcasts (fire and forget). Source of truth is the existing MongoDB collections.
- **Acknowledgement model** — Global per-document, not per-user. `acknowledged` boolean flag on the `documents` collection. When anyone clicks "Mark as Read", the document is marked as read for everyone. All other connected clients see the change via SignalR broadcast.
- **No new collections** — No `notifications` collection. The `documents` collection drives UI state. SignalR just pushes transient alerts.
- **Offline/reconnect** — `@microsoft/signalr` SDK has `withAutomaticReconnect()`. On reconnect, client re-fetches data from existing REST APIs. No gap detection needed — MongoDB is the source of truth.
- **Existing Socket.IO** — Coexists. No migration in this epic. Call-status events stay on Socket.IO.
- **Azure SignalR infrastructure** — Owned by Anatoliy (D0). Free tier for dev, Standard for staging/prod.
