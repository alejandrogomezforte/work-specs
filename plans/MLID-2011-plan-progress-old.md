# MLID-2011 — Orders: Document Notifications

## Epic Workflow

This template follows the [SW Workflow](../dx/sw-workflow.md). For each sub-task, run the workflow cycle once and update this document to track progress. Read this document first to understand overall context before starting each cycle.

For the git branch strategy, see [epic-git-branch-strategy.md](../dx/epic-git-branch-strategy.md).

---

## Purpose

Provide real-time visibility to Infusion Guides when new documents are added to active orders. Currently document arrival is communicated via Spruce (the patient messaging app), creating noise that competes with patient messages (which have a ~2hr SLA). This epic replaces that with in-app notifications powered by WebSockets, so Infusion Guides can immediately acknowledge new documents and progress orders toward the ~3-day "Ready to Schedule" target.

---

## Architecture Decision

**WebSockets (Socket.IO)** — Decided with Dvir Rassovsky. The app already uses Socket.IO 4.7.4 for real-time features. This epic builds a **reusable notifications framework** on top of the existing Socket.IO infrastructure, then uses it to deliver document notifications.

MVP/demo repo: `github.com/alejandrogomezforte/websockets-demo`

---

## Current Socket.IO State (apps/web)

| Component | File | What it does |
|-----------|------|--------------|
| Server singleton | `services/io/socketService.ts` | `SocketService` class, `new Server(server, { path: '/api/socket_io' })` |
| Client context | `utils/context/socket-context.tsx` | `SocketProvider` + `useSocket()` hook, reconnection with 5 retries |
| Init API route | `pages/api/socket.ts` | NextAuth session check → initializes Socket.IO on HTTP server |
| Call status emit | `services/io/changeCallStatus.ts` | `io.emit('callStatusChanged', ...)` |
| Global call emit | `services/io/globalCallStatusHandler.ts` | `io.emit('globalCallStatusChanged', ...)` |
| Types | `types/io.ts` | `IO`, `RequiredIO`, `ExtendedSocket`, `CustomNextApiResponse` |

**Current limitations:**
- No rooms — all events are global `io.emit()` (every client receives everything)
- No namespaces — everything on the default namespace
- No auth on the socket connection itself (only on the `/api/socket` init route)
- Only 2 event types: `callStatusChanged` and `globalCallStatusChanged`
- No delta/sync pattern — just fire-and-forget events

---

## Proposed Architecture: Layered Notification System

Inspired by the demo project's patterns (REST for writes, WS for reads, delta broadcasts, rooms, sequence numbers), adapted to Socket.IO and the existing codebase.

### Core Principles

1. **REST for writes, Socket.IO for reads** — All mutations (acknowledge, bulk-acknowledge) go through REST API routes. Socket.IO only pushes data server→client.
2. **Room-scoped delivery** — Socket.IO has built-in rooms (no manual Map needed). Each user joins `user:{userId}` on connect. Notifications are emitted to specific user rooms, not broadcast globally.
3. **Delta broadcasts** — Push individual notification events (`notification:created`, `notification:updated`), not the entire notification list.
4. **Full sync on connect** — When a client connects (or reconnects), fetch current notification state via REST API, then listen for deltas via Socket.IO.
5. **Sequence numbers for gap detection** — Each broadcast carries a monotonic `seq`. Client detects gaps and triggers a full re-sync via REST.
6. **Feature-flag gated** — `DOCUMENT_NOTIFICATIONS` flag gates everything at API + UI level.

### Layer Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        FRONTEND LAYERS                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  UI Components (D3, D4, D5)                             │    │
│  │  Order Tracker badges, Documents tab, History tab       │    │
│  └──────────────────────┬──────────────────────────────────┘    │
│                         │ uses                                  │
│  ┌──────────────────────▼──────────────────────────────────┐    │
│  │  useNotifications Hook                                  │    │
│  │  - Subscribe to notification state                      │    │
│  │  - Unread counts (total, per-order)                     │    │
│  │  - Acknowledge actions (calls REST API)                 │    │
│  └──────────────────────┬──────────────────────────────────┘    │
│                         │ uses                                  │
│  ┌──────────────────────▼──────────────────────────────────┐    │
│  │  NotificationContext + Provider                         │    │
│  │  - Holds notification state                             │    │
│  │  - Listens to Socket.IO events (notification:created,   │    │
│  │    notification:updated)                                │    │
│  │  - Applies deltas to local state                        │    │
│  │  - Sequence number tracking + gap detection             │    │
│  │  - Full sync via REST on connect/reconnect/gap          │    │
│  └──────────────────────┬──────────────────────────────────┘    │
│                         │ uses                                  │
│  ┌──────────────────────▼──────────────────────────────────┐    │
│  │  SocketContext (existing, enhanced)                     │    │
│  │  - Socket.IO connection management                      │    │
│  │  - Room auto-join on connect (user:{userId})            │    │
│  │  - Connection status (connected/disconnected/reconnect) │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                    REST (mutations)  +  Socket.IO (real-time reads)
                              │
┌─────────────────────────────────────────────────────────────────┐
│                        BACKEND LAYERS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  API Routes (REST)                                      │    │
│  │  GET  /api/notifications       — fetch user's notifs    │    │
│  │  PATCH /api/notifications/:id  — acknowledge one        │    │
│  │  PATCH /api/notifications/bulk — acknowledge many       │    │
│  └──────────────────────┬──────────────────────────────────┘    │
│                         │ calls                                 │
│  ┌──────────────────────▼──────────────────────────────────┐    │
│  │  Notification Service (services/notifications/)         │    │
│  │  - create(notification) → saves + emits via broadcast   │    │
│  │  - getByUser(userId, filters)                           │    │
│  │  - acknowledge(notificationId, userId)                  │    │
│  │  - bulkAcknowledge(notificationIds, userId)             │    │
│  │  - getUnreadCounts(userId) → { total, byOrder }         │    │
│  └──────────┬───────────────────────┬──────────────────────┘    │
│             │ uses                  │ uses                      │
│  ┌──────────▼──────────┐ ┌─────────▼──────────────────────┐    │
│  │  Notification Model │ │  Notification Broadcast Service │    │
│  │  (Mongoose)         │ │  (services/notifications/       │    │
│  │  - MongoDB CRUD     │ │   broadcast.ts)                 │    │
│  │  - Indexes on       │ │  - emitToUser(userId, event,    │    │
│  │    userId, orderId, │ │    payload)                     │    │
│  │    acknowledged      │ │  - Sequence number management   │    │
│  └─────────────────────┘ │  - Uses Socket.IO rooms         │    │
│                          └─────────────────────────────────┘    │
│                                     │ uses                      │
│  ┌──────────────────────────────────▼──────────────────────┐    │
│  │  SocketService (existing, enhanced)                     │    │
│  │  - Room management: join user:{userId} on connect       │    │
│  │  - io.to(room).emit(event, payload)                     │    │
│  │  - Auth: validate session on connection (middleware)     │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Notification Triggers (entry points)                   │    │
│  │  - Document webhook handlers                            │    │
│  │  - Document upload routes                               │    │
│  │  - Calls NotificationService.create() with payload      │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Socket.IO Events

**Server → Client (via rooms, not global broadcast):**

| Event | Payload | When |
|-------|---------|------|
| `notification:created` | `{ notification, seq }` | New notification created |
| `notification:updated` | `{ notification, seq }` | Notification acknowledged |

**Client → Server: NONE** — All mutations go through REST.

**On connect/reconnect:** Client calls `GET /api/notifications` for full state sync, then listens for deltas.

### Room Strategy

Socket.IO built-in rooms (no custom room manager needed):

```
user:{userId}  — each user's personal room, joined on connect
```

On connection, the server:
1. Validates the session (NextAuth)
2. Joins the socket to `user:{userId}`
3. Existing call-status events continue as global `io.emit()` (no breaking changes)

### Notification Data Model (MongoDB)

```typescript
interface INotification {
  _id: ObjectId;
  userId: ObjectId;           // Who receives this notification
  type: 'document_added';     // Extensible for future types
  orderId: ObjectId;          // The order this relates to
  referenceId: string;        // e.g., document ID
  referenceType: string;      // e.g., 'document'
  title: string;              // e.g., "Clinical Notes - Inflectra"
  message: string;            // e.g., "New document added to order #319308"
  metadata: {                 // Type-specific data
    documentType?: string;    // e.g., "Clinical", "Insurance Card", "Order"
    source?: string;          // "fax" | "e-order" | "upload"
  };
  acknowledged: boolean;      // Has the user marked it as read?
  acknowledgedAt?: Date;
  createdAt: Date;
  updatedAt: Date;
}
```

**Indexes:**
- `{ userId: 1, acknowledged: 1, createdAt: -1 }` — fetch unread notifications per user
- `{ userId: 1, orderId: 1, acknowledged: 1 }` — unread count per order for a user
- `{ createdAt: 1 }` — TTL index for cleanup (if we decide on expiry)

### New File Structure

```
apps/web/
├── services/
│   └── notifications/
│       ├── notificationService.ts    — CRUD + business logic
│       ├── notificationBroadcast.ts  — Socket.IO room-scoped emit + seq
│       └── notificationModel.ts      — Mongoose schema (or in models/)
├── models/
│   └── Notification.ts               — Mongoose model
├── types/
│   └── notification.ts               — TypeScript interfaces
├── app/api/
│   └── notifications/
│       ├── route.ts                  — GET (list), PATCH (bulk acknowledge)
│       └── [id]/
│           └── route.ts              — PATCH (acknowledge one)
└── utils/context/
    └── notification-context.tsx      — React context + provider + hook
```

### Enhancements to Existing Socket.IO

Changes to existing files (non-breaking):

1. **`services/io/socketService.ts`** — Add room join on connection:
   - On `connection` event, read user session → `socket.join('user:' + userId)`
   - Add `emitToUser(userId, event, payload)` method using `io.to('user:' + userId).emit()`

2. **`utils/context/socket-context.tsx`** — No changes needed (connection management stays the same)

3. **`pages/api/socket.ts`** — May need to pass user identity to socket via handshake auth

### Data Flow Example

```
Document uploaded (webhook/upload)
  │
  ▼
Notification Trigger (in webhook handler / upload route)
  │ calls NotificationService.create({
  │   userId: assignedGuideId,
  │   type: 'document_added',
  │   orderId, referenceId: documentId,
  │   title: docName, metadata: { source: 'fax' }
  │ })
  │
  ▼
NotificationService.create()
  │ 1. Saves to MongoDB
  │ 2. Calls NotificationBroadcast.emitToUser(userId, 'notification:created', { notification, seq })
  │
  ▼
NotificationBroadcast.emitToUser()
  │ io.to('user:' + userId).emit('notification:created', { notification, seq: ++seq })
  │
  ▼
Client (NotificationContext)
  │ Receives 'notification:created'
  │ Checks seq continuity (gap? → full re-sync via REST)
  │ Applies delta: adds notification to local state
  │ Updates unread counts
  │
  ▼
UI Components
  │ useNotifications() hook re-renders
  │ - Order Tracker badge updates
  │ - Documents tab badge updates
  │ - "New" tag appears on document row
```

---

## Phases

### Phase A: Notifications Framework (foundation)
Design and implement a generic, reusable real-time notifications system using WebSockets. This is the infrastructure layer that future notification types (not just documents) can plug into.

### Phase B: Document Notifications (feature)
Build the specific document notification experience on top of the framework — badges in the Order Tracker, acknowledgement flow in Order Details, and audit trail entries.

---

## Deliverable 1: Notifications Framework (Phase A)

**Goal:** End-to-end notification infrastructure — backend (model, service, Socket.IO, API) + frontend (context, hooks). Testable as a unit: create a notification via API → it arrives in real-time on the client, can be read, acknowledged, and bulk-acknowledged.

| Task ID | Summary | SP | Status | Branch | Merged to Epic | Notes |
|---------|---------|:--:|:------:|--------|:--------------:|-------|
| D1-T1 | Feature flag `DOCUMENT_NOTIFICATIONS` + notification TypeScript types | - | To Do | - | - | |
| D1-T2 | Notification Mongoose model + MongoDB schema | - | To Do | - | - | |
| D1-T3 | Notification service layer (create, get, acknowledge, bulk-acknowledge) | - | To Do | - | - | |
| D1-T4 | Socket.IO notification events (emit on create, room-per-user) | - | To Do | - | - | |
| D1-T5 | API routes: GET notifications, PATCH acknowledge, PATCH bulk-acknowledge | - | To Do | - | - | |
| D1-T6 | Notification React context + provider (connects to Socket.IO) | - | To Do | - | - | |
| D1-T7 | `useNotifications` hook (subscribe, unread count, acknowledge actions) | - | To Do | - | - | |

**D1 PR to develop:** Not yet

---

## Deliverable 2: Document Notification Triggers (Phase B)

**Goal:** Wire document upload/creation events to produce notifications. When a VA or the system adds a document to an order, a notification is created and pushed via WebSocket. Testable: upload a document to an order → notification appears in real-time for the assigned user.

| Task ID | Summary | SP | Status | Branch | Merged to Epic | Notes |
|---------|---------|:--:|:------:|--------|:--------------:|-------|
| D2-T1 | Investigate all document creation entry points (webhooks, uploads, fax, e-order) | - | To Do | - | - | Investigation/spike |
| D2-T2 | Emit document notification on each entry point | - | To Do | - | - | |
| D2-T3 | Include source metadata in notification payload (Fax vs E-Order) | - | To Do | - | - | |

**D2 PR to develop:** Not yet

---

## Deliverable 3: Order Tracker — Notification Badge & Filter (Phase B)

**Goal:** Show notification indicators on the Orders list view and allow filtering by "Action Required". Testable: orders with unread documents show a badge; clicking "Action Required" filters the list; badge updates in real-time when a new document arrives.

| Task ID | Summary | SP | Status | Branch | Merged to Epic | Notes |
|---------|---------|:--:|:------:|--------|:--------------:|-------|
| D3-T1 | Notification badge/icon on Order ID column (unread doc count) | - | To Do | - | - | |
| D3-T2 | "Action Required" quick filter for orders with unread notifications | - | To Do | - | - | |
| D3-T3 | Real-time badge update via Socket.IO (no page refresh needed) | - | To Do | - | - | |

**D3 PR to develop:** Not yet

---

## Deliverable 4: Order Details — Documents Tab Acknowledgement (Phase B)

**Goal:** In the order detail view, show unread count on Documents tab, highlight new documents, and provide acknowledge (single + bulk) functionality. Testable: navigate to order detail → Documents tab shows unread badge → new docs highlighted with "New" tag → "Mark as Read" clears per-doc → "Mark all as Read" clears all → badge updates in real-time.

| Task ID | Summary | SP | Status | Branch | Merged to Epic | Notes |
|---------|---------|:--:|:------:|--------|:--------------:|-------|
| D4-T1 | Documents tab header badge showing unread count | - | To Do | - | - | |
| D4-T2 | "New" tag/highlight on unread documents in the table | - | To Do | - | - | |
| D4-T3 | "Mark as Read" button per document row | - | To Do | - | - | |
| D4-T4 | "Mark all as Read" bulk acknowledge button | - | To Do | - | - | |
| D4-T5 | On acknowledge, clear notification badge in real-time | - | To Do | - | - | |

**D4 PR to develop:** Not yet

---

## Deliverable 5: Order History — Document Audit Trail (Phase B)

**Goal:** Log document additions in the Order History tab with category, description, timestamp, and source differentiation. Testable: add a document → Order History shows entry with "Document" category, document title/type, timestamp, and source (Fax/E-Order).

| Task ID | Summary | SP | Status | Branch | Merged to Epic | Notes |
|---------|---------|:--:|:------:|--------|:--------------:|-------|
| D5-T1 | Audit log entry on document addition (Category: Document, description with title + type) | - | To Do | - | - | |
| D5-T2 | Source differentiation in audit entries (Fax vs E-Order) | - | To Do | - | - | |

**D5 PR to develop:** Not yet

---

## Task List

### D1: Notifications Framework (? SP)
- D1-T1 — Feature flag + notification types
- D1-T2 — Notification Mongoose model
- D1-T3 — Notification service layer
- D1-T4 — Socket.IO notification events
- D1-T5 — API routes (GET, PATCH acknowledge, PATCH bulk)
- D1-T6 — Notification React context + provider
- D1-T7 — `useNotifications` hook

### D2: Document Notification Triggers (? SP)
- D2-T1 — Investigate document creation entry points
- D2-T2 — Emit notifications on document creation
- D2-T3 — Source metadata (Fax vs E-Order)

### D3: Order Tracker UI (? SP)
- D3-T1 — Notification badge on Order ID
- D3-T2 — "Action Required" quick filter
- D3-T3 — Real-time badge update

### D4: Documents Tab Acknowledgement (? SP)
- D4-T1 — Documents tab unread badge
- D4-T2 — "New" tag on unread documents
- D4-T3 — "Mark as Read" per document
- D4-T4 — "Mark all as Read" bulk
- D4-T5 — Clear badge on acknowledge

### D5: Order History Audit (? SP)
- D5-T1 — Audit log entry on document addition
- D5-T2 — Source differentiation (Fax vs E-Order)

---

## Dependency Graph

```
D1 (Notifications Framework)
 ├── D2 (Document Triggers) ── depends on D1
 │    ├── D3 (Order Tracker UI) ── depends on D2
 │    └── D4 (Documents Tab) ── depends on D2
 └── D5 (Order History Audit) ── independent, can parallel with D2+
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
- **Total Story Points**: TBD (to be estimated during grooming)
- **Epic Branch**: `epic/MLID-2011-document-notifications`
- **Deliverables**: 5
- **Total Tasks**: 20

---

## Non-Code Tasks

| Task ID | Summary | Status |
|---------|---------|:------:|
| - | Review designer specs at li-playground and prototype HTML | To Do |
| - | Codebase investigation: current Socket.IO setup, document upload flows, order tracker pipeline | To Do |
| - | Estimate story points per task after investigation | To Do |
| - | Create Jira sub-tasks once plan is approved | To Do |

---

## Open Questions

1. **Notification persistence** — How long do notifications live? Should there be a TTL or cleanup job?
2. **Multi-user acknowledgement** — If User A acknowledges a doc, does it clear for all users or just User A? (Likely per-user based on the "Infusion Guide managing the order" framing)
3. **Notification scope** — Only documents on orders assigned to the user, or all open orders?
4. **Offline/reconnect** — When a user reconnects after being offline, how do they catch up? (API fetch on connect + Socket.IO for live updates)
5. **Existing document upload paths** — Need to catalog all entry points where documents get attached to orders (webhooks, manual upload, fax integration, e-order)
6. **Order History tab** — Does it already exist? What's the current audit trail mechanism? Need to investigate.
