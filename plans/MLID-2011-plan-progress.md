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

**SignalR is a 3rd-party service the webapp integrates with** — like Spruce or WeInfuse. Connection string stored in Azure Key Vault. Per-user JWT tokens issued via `POST /api/signalr/negotiate`. The server broadcasts events to all connected browsers; each browser decides if the event is relevant to its current view.

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
- **Notifications are persisted** — The `notifications` collection stores notification state. SignalR broadcasts are transient (fire and forget) — they just trigger clients to re-fetch from MongoDB.
- **Client SDK** — `@microsoft/signalr` npm package. Handles transport negotiation (WebSocket → SSE → Long Polling), auto-reconnection, and connection lifecycle.
- **Hub** — Single hub named `li-hub`. Generic — reusable for any future real-time broadcast needs.
- **Connection string** — Stored in Azure Key Vault (secret name: `signalr-primary-connection-string`, env override: `SIGNALR_KV_SECRET_NAME`).
- **Auth model** — Per-user JWT tokens. Each browser client calls `POST /api/signalr/negotiate` to get a `{ url, accessToken }` pair, then connects to SignalR with that token.
- **Existing Socket.IO** — Coexists. Call-status events (`callStatusChanged`, `globalCallStatusChanged`) remain on Socket.IO. No migration in this epic.

---

## Notification Data Model (PE Implementation)

The Principal Engineer implemented the notification data model in branch `feature/MLID-2011-notifications-data-model`. This is the **canonical model** — all epic work builds on it.

### Two Collections

#### `Notification` model (`models/Notification.ts`)

One document per event. Shared across users — not duplicated per recipient.

```typescript
type NotificationType = 'task' | 'user';
type NotificationPriority = 'low' | 'normal' | 'high';

interface INotificationEntity {
  entityType: string; // domain name: 'order', 'document', 'patient', 'appointment', 'clinical-review', etc.
  entityId: string; // document _id reference
}

interface INotification extends Document {
  // _id inherited from Document as ObjectId (not overridden)
  title: string; // short display title
  message: string; // notification body
  type: NotificationType; // determines read tracking behavior (see below)
  priority: NotificationPriority; // 'low' | 'normal' | 'high', default: 'normal'
  targetUserId: string | null; // target a specific user (at least one target required)
  targetLocationId: string | null; // target all users assigned to this location
  targetRole: string | null; // target all users with this role
  entities: INotificationEntity[]; // linked entities — polymorphic, multiple allowed
  isRead: boolean; // default: false — only meaningful for 'task' type
  readBy: string | null; // userId who acknowledged it — only meaningful for 'task' type
  readAt: Date | null; // when acknowledged — only meaningful for 'task' type
  expiresAt: Date | null; // optional expiration (sparse index, NOT TTL — no auto-deletion)
  createdAt: Date; // auto (timestamps: true)
  updatedAt: Date; // auto (timestamps: true)
  createdBy: string; // userId or 'system'
}
```

**Validation:** A `pre('validate')` hook enforces that at least one of `targetUserId`, `targetLocationId`, or `targetRole` must be set. Creating a notification with all three as `null` will throw a validation error.

**Targeting** uses OR logic: a notification matches a user if **at least one** of the targeting criteria is satisfied:

- `targetUserId === user.id` OR
- `targetLocationId IN user.locations` (user can belong to multiple locations, matched via `$in`) OR
- `targetRole === user.role`

**Read tracking** differs by type:

- `type: 'task'` — **Actionable items that any team member can acknowledge.** Read state tracked **inline** on the notification itself (`isRead`, `readBy`, `readAt`). Once **anyone** marks it as read, it's read for **everyone** in the target group. Example: a new fax arrives for an order → notification targets all users at the clinic location → the first user to review it clicks "Acknowledge" → notification clears for the entire team.

- `type: 'user'` — **Informational notifications where each user independently tracks their own read state.** Read state tracked in a **separate `NotificationRead` collection**. One user marking it read does **not** affect others. Example: "You have been assigned to order #456" → each user has their own read receipt.

#### `NotificationRead` model (`models/NotificationRead.ts`)

Per-user read tracking for `user`-type notifications. One document per user per notification. Each record means "this user has read this notification."

```typescript
interface INotificationRead extends Document {
  // _id inherited from Document as ObjectId (not overridden)
  notificationId: ObjectId; // ref -> notifications._id
  userId: string; // who read it
  readAt: Date; // when they read it (default: Date.now)
}
```

Unique index on `(notificationId, userId)` — one read receipt per user per notification.

### Indexes

**`notifications` collection:**

| Index                                                  | Purpose                                                                             |
| ------------------------------------------------------ | ----------------------------------------------------------------------------------- |
| `{ targetUserId: 1, createdAt: -1 }`                   | User's notifications, newest first                                                  |
| `{ targetLocationId: 1, createdAt: -1 }`               | Location-scoped notifications                                                       |
| `{ targetRole: 1, createdAt: -1 }`                     | Role-scoped notifications                                                           |
| `{ "entities.entityType": 1, "entities.entityId": 1 }` | All notifications for a specific entity                                             |
| `{ expiresAt: 1 }` (sparse)                            | Query index for expiration filtering (NOT a TTL index — no auto-deletion by design) |

**`notificationreads` collection:**

| Index                                       | Purpose                                    |
| ------------------------------------------- | ------------------------------------------ |
| `{ notificationId: 1, userId: 1 }` (unique) | One read receipt per user per notification |
| `{ userId: 1 }`                             | "What has this user read?"                 |

### Service Layer (`services/mongodb/notification.ts`)

| Function                                                        | Purpose                                                                                                                                                                              |
| --------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `createNotification(data)`                                      | Create a notification (validated: at least one target field required)                                                                                                                |
| `getNotificationsForUser({ userId, locationIds, role })`        | Get notifications matching user OR location OR role, paginated (default: limit 50, offset 0), sorted newest first. Excludes expired notifications.                                   |
| `getUnreadCountForUser({ userId, locationIds, role })`          | Count unread across both types via `$facet` aggregation (task: `isRead=false`, user: `$lookup` against `notificationreads` with `$size: 0`). Excludes expired notifications.         |
| `getCountForEntity({ entityType, entityId, ..., unreadOnly? })` | Count notifications for a specific entity (e.g., all for orderId X). When `unreadOnly: true`, uses same `$facet` pattern as `getUnreadCountForUser`. Excludes expired notifications. |
| `markTaskAsRead(notificationId, userId)`                        | Flip `isRead=true`, set `readBy` and `readAt` via `findOneAndUpdate` with `type: 'task'` guard — returns `null` if called on a `user`-type notification (prevents corruption)        |
| `markUserNotificationAsRead(notificationId, userId)`            | Insert a `NotificationRead` record (idempotent — silently ignores duplicate key error 11000)                                                                                         |
| `bulkMarkUserNotificationsAsRead(notificationIds, userId)`      | Batch insert `NotificationRead` records via `insertMany({ ordered: false })` (handles both `MongoBulkWriteError` and error code 11000 for duplicates)                                |
| `isReadByUser(notificationId, userId)`                          | Check if a `NotificationRead` record exists for this user+notification                                                                                                               |

### Entity Linking

Notifications can link to one or more entities. This enables queries like "show me all notifications for this order" or "how many unread notifications does this patient have?"

```typescript
entities: [
  { entityType: 'order', entityId: '6612a1b2c3d4e5f6a7b8c9d0' },
  { entityType: 'document', entityId: '6612a1b2c3d4e5f6a7b8c9d1' },
];
```

**Supported entity types** (open-ended, no enum constraint): `order`, `document`, `patient`, `appointment`, `clinical-review`, and any future entity type — no schema changes needed.

### SignalR Layer (PE Implementation — separate PR, not in data model PR)

| Component           | File                                 | Purpose                                                                                                                                |
| ------------------- | ------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------- |
| Server-side service | `services/signalr/index.ts`          | Parse connection string from Key Vault, generate JWT tokens, `negotiate()` for clients, `generateServerToken()` for server→SignalR API |
| Negotiate API route | `app/api/signalr/negotiate/route.ts` | POST, authenticated. Returns `{ url, accessToken }` for client SDK                                                                     |
| React context       | `utils/context/signalr-context.tsx`  | `SignalRProvider` + `useSignalR()` hook. Auto-reconnect. Exposes `connection` and `isConnected`                                        |

**Key details:**

- Hub name: `li-hub`
- Auth model: **per-user JWT tokens** via negotiate endpoint
- Key Vault secret: `signalr-primary-connection-string` (env override: `SIGNALR_KV_SECRET_NAME`)

### Notification Type Semantics (Resolved)

The PE's data model defines two notification types with distinct read-tracking behaviors:

| Type   | Use Case                                                                        | Read Tracking                                                                                                                            | Mutation                                                                                               | Example                                                                                                                  |
| ------ | ------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------ |
| `task` | Actionable items that **any** team member can acknowledge                       | **Inline** on the notification: `isRead`, `readBy`, `readAt`. Once anyone marks it read, it's read for **everyone**.                     | `markTaskAsRead(notificationId, userId)`                                                               | New fax arrives for order → targets clinic location → first user to review clicks "Acknowledge" → clears for entire team |
| `user` | Informational notifications where **each user** independently tracks read state | **Separate collection** `notificationreads`. Each user has their own read receipt — one user marking it read does **not** affect others. | `markUserNotificationAsRead(notificationId, userId)` or `bulkMarkUserNotificationsAsRead(ids, userId)` | "You have been assigned to order #456" → each user sees/dismisses independently                                          |

### Document Notification Type Decision (Updated 2026-04-07)

**Requirement:** Only the user **assigned to** the order can view and acknowledge documents — both at document level and at "all documents" level.

**Decision:** Document notifications use **`user` type** with `targetUserId` set to the order's assigned user. This replaces the previous `task` type / `targetLocationId` approach.

- **No `acknowledged` flag on documents** — badge counts driven entirely from `Notification` + `NotificationRead` queries.
- **No document model changes needed.**

### Update 2026-04-14 — Decouple Badge Visibility from Notification Ownership

**Business requirement (from Alejandro, confirmed during D3-T1 manual QA):**

> "It doesn't matter if order ABC is assigned to user John and has 4 unread notifications associated to it. When I, Alejandro, visit the New Orders page, I should still see that order and the badge that says 4 unread. Business priority over PE architecture."

**What changes:**

| Concern                                           | Before (2026-04-07)                                                        | After (2026-04-14)                                                           |
| ------------------------------------------------- | -------------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| **Notification ownership** (who can Mark as Read) | `targetUserId = order.assignedTo`                                          | **Unchanged** — still only the targeted user can acknowledge                 |
| **Per-entity badge count visibility**             | Filtered by `buildUserTargetFilter(viewer)` — only the assignee saw badges | **Global** — every authenticated user sees the same count on every order     |
| **"Unread" semantics for entity badge**           | `notificationreads` matched against **viewer's** userId                    | `notificationreads` matched against **`targetUserId`** (the intended reader) |

**Why:** A document notification represents a real-world "pending item on an order" that matters to the whole team, even though only the assignee can clear it. Hiding the count from non-assignees makes the Orders tab useless as a team dashboard — supervisors and collaborators can't see which orders have outstanding work.

**Scope of service-layer change:**

- `getCountForEntity()` in `services/mongodb/notification.ts` — **drop** `buildUserTargetFilter()` from the `$match`. For `type: 'user'`, change the `notificationreads` `$lookup` to match `userId === notification.targetUserId` (the target), not the viewer.
- `getNotificationsForUser()` / `getUnreadCountForUser()` (header bell) — **unchanged.** The inbox dropdown is still strictly per-viewer.
- `markUserNotificationAsRead()` / `bulkMarkUserNotificationsAsRead()` — **unchanged.** Still writes a read receipt keyed to `userId` and still rejects non-targets at the route level (the "Mark as Read" button is hidden for non-assignees).
- Pre-validate hook requiring at least one target field — **unchanged.** Every notification still has an owner.

**Impact on PE (Anatoliy) architecture:**

The `Notification` model's target fields retain their original meaning (OWNERSHIP / read-permission). What changes is the _entity badge count_ query — it now treats notifications as a team-visible backlog keyed by entity, while preserving single-user read semantics. This is an additive reinterpretation, not a breaking change to the data model.

**Files touched by this update:**

- `apps/web/services/mongodb/notification.ts` — `getCountForEntity` pipeline
- `apps/web/services/mongodb/notification.test.ts` — new cases for cross-user visibility and target-based unread
- `apps/web/app/api/notifications/counts/route.ts` — still session-auth'd, but no viewer scoping on the count
- `apps/web/app/api/notifications/counts/route.test.ts` — update expectations

**Downstream tasks affected:** D3-T1 (MLID-2084) — see "Update 2026-04-14" section of its plan.

### Scenario 1: Document Added to Order

1. Document saved to MongoDB
2. Create `user` notification — targeted to `targetUserId` (order's assigned user):
   ```typescript
   createNotification({
     type: 'user',
     targetUserId: order.assignedTo,
     entities: [
       { entityType: 'order', entityId: orderId },
       { entityType: 'document', entityId: documentId },
     ],
     createdBy: 'system',
   });
   ```
3. SignalR broadcast to all connected clients → assigned user's client re-fetches badge counts

### Scenario 2: Document Marked as Read (single)

1. Call `markUserNotificationAsRead(notificationId, userId)` — inserts a `NotificationRead` record (idempotent).
2. SignalR broadcast to all connected clients → badge counts update

### Scenario 3: All Documents Marked as Read (bulk)

1. Query unread notification IDs for the order, then call `bulkMarkUserNotificationsAsRead(notificationIds, userId)`.
2. SignalR broadcast → badge counts update

### What's Already Built

**PR #1049 — Data Model** (`feature/MLID-2011-notifications-data-model`):

| Component                                                                                                                            | Status |
| ------------------------------------------------------------------------------------------------------------------------------------ | ------ |
| `Notification` Mongoose model + schema + indexes + target validation                                                                 | Done   |
| `NotificationRead` Mongoose model + schema + indexes                                                                                 | Done   |
| Notification service (CRUD, counts, mark read) + tests (33 service + 4 model = 37 tests)                                             | Done   |
| PR review fixes: type guard on `markTaskAsRead`, `MongoBulkWriteError` handling, expiry filtering, `_id` type fix, target validation | Done   |

**Separate PE work — SignalR layer** (not in PR #1049, expected in a future PR):

| Component                                                 | Status                        |
| --------------------------------------------------------- | ----------------------------- |
| SignalR server-side service (negotiate, token generation) | Done (PE, separate branch/PR) |
| SignalR negotiate API route                               | Done (PE, separate branch/PR) |
| SignalR React context + `useSignalR()` hook               | Done (PE, separate branch/PR) |
| `@microsoft/signalr` client SDK dependency                | Done (PE, separate branch/PR) |

### What's Still Needed

| Component                                                                       | Status                                                                                         |
| ------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| ~~`acknowledged` flag on Document model~~                                       | **Not needed** — badge counts driven entirely from `Notification` + `NotificationRead` queries |
| Notification API routes (`GET /notifications`, `PATCH /notifications/:id/read`) | Still needed (D2-T1) — uses `markUserNotificationAsRead()`                                     |
| SignalR server-side **broadcast** function (send event to all/group)            | Still needed (D1-T1) — PE built negotiate/token but not `broadcastToRoom()`                    |
| Document creation triggers (webhook/upload → create notification + broadcast)   | Still needed (D2-T2)                                                                           |
| Feature flag `ORDER_DOCUMENT_NOTIFICATIONS`                                     | Still needed (D1-T1)                                                                           |
| UI components (badges, "New" tags, "Mark as Read")                              | Still needed (D3)                                                                              |

---

## Proposed Architecture

### Core Principles

1. **SignalR = transient push alerts** — Fire and forget. No persistence. Tells all connected clients "something changed, re-fetch."
2. **`notifications` + `notificationreads` collections = source of truth** — Notification state lives in `Notification` model, per-user read state in `NotificationRead` collection. No flags on the `documents` collection.
3. **REST for mutations** — "Mark as Read" calls API routes that insert `NotificationRead` records via `markUserNotificationAsRead()`. SignalR broadcasts the change to all clients.
4. **Feature-flag gated** — `ORDER_DOCUMENT_NOTIFICATIONS` flag gates everything at API + UI level.

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
│  │  POST  /api/signalr/negotiate          — PE built       │    │
│  │  GET   /api/notifications              — user's notifs  │    │
│  │  PATCH /api/notifications/:id/read     — mark read      │    │
│  │  GET   /api/orders/details             — already exists  │    │
│  └──────────────────────┬──────────────────────────────────┘    │
│                         │ calls                                 │
│  ┌──────────────────────▼──────────────────────────────────┐    │
│  │  SignalR Service (services/signalr/) — PE built         │    │
│  │  - negotiate(userId) → { url, accessToken }             │    │
│  │  - generateServerToken() for server→SignalR API         │    │
│  │  - Connection string from Key Vault                     │    │
│  │  - broadcastToRoom() / sendToAll() — not yet built (D1-T1) │  │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Document Triggers (entry points)                       │    │
│  │  - Document webhook handlers                            │    │
│  │  - Document upload routes                               │    │
│  │  - On creation: create user notification + broadcast    │    │
│  │  - On mark read: insert NotificationRead + broadcast    │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Notification Model (PE built — Mongoose)               │    │
│  │  - notifications + notificationreads collections        │    │
│  │  - Document notifications: type 'user', per-user read   │    │
│  │    receipts via NotificationRead collection              │    │
│  │  - Multi-entity linking via entities[] ($elemMatch)     │    │
│  │  - Targeting: targetUserId (order's assigned user)      │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### SignalR Events (server → client, transient)

| Event (target)  | Payload                                                | When                                 |
| --------------- | ------------------------------------------------------ | ------------------------------------ |
| `documentAdded` | `{ documentId, patientId, orderId, category, source }` | New document created/uploaded        |
| `documentRead`  | `{ documentId, orderId, userId }`                      | Document notification marked as read |

**These are fire-and-forget.** Client receives the event and re-fetches relevant data from existing REST APIs. If the client misses an event (offline), the next page load or API call will have the correct state from MongoDB.

### Data Flow: New Document Added

```
Document uploaded (webhook/upload)
  │
  ▼
Document Trigger (in webhook handler / upload route)
  │ 1. Saves document to MongoDB
  │ 2. Creates notification via createNotification():
  │      { title: "New Document Added",
  │        message: "A new fax has been added to Order #ORD-1234",
  │        type: 'user',
  │        targetUserId: order.assignedTo,
  │        entities: [{ entityType: 'order', entityId },
  │                    { entityType: 'document', entityId }],
  │        createdBy: 'system' }
  │ 3. Broadcasts SignalR event to all connected clients
  │
  ▼
Azure SignalR Service → ALL clients
  │
  ▼
Assigned user's client (listening via SignalRContext)
  │ connection.on('documentAdded', handler)
  │ Re-fetches unread counts from Notification + NotificationRead queries
  │ UI updates: badge count increments, "New" tag shows on document row
```

### Data Flow: Mark as Read (single)

```
Assigned user clicks "Mark as Read" on a document
  │
  ▼
API Route (PATCH /api/notifications/:id/read)
  │ 1. markUserNotificationAsRead(notificationId, userId)
  │    — inserts NotificationRead record (idempotent)
  │ 2. Broadcasts SignalR event to all connected clients
  │
  ▼
Azure SignalR Service → ALL clients
  │
  ▼
Assigned user's client re-fetches unread counts
  │ Badge count decrements, "New" tag disappears on that document row
```

### Data Flow: Mark All as Read (bulk)

```
Assigned user clicks "Mark all as Read" on an order
  │
  ▼
API Route (PATCH /api/notifications/bulk-read)
  │ 1. Query unread notification IDs for the order
  │ 2. bulkMarkUserNotificationsAsRead(notificationIds, userId)
  │    — batch inserts NotificationRead records (skips duplicates)
  │ 3. Broadcasts SignalR event to all connected clients
  │
  ▼
Azure SignalR Service → ALL clients
  │
  ▼
Assigned user's client re-fetches unread counts
  │ All badge counts clear, all "New" tags disappear
```

### File Structure

**Already built by PE — PR #1049** (data model, `feature/MLID-2011-notifications-data-model`):

```
apps/web/
├── models/
│   ├── Notification.ts               — ✅ Mongoose model + target validation hook
│   ├── Notification.test.ts          — ✅ Model validation tests (4 tests)
│   └── NotificationRead.ts           — ✅ Mongoose model (notificationreads collection)
└── services/
    └── mongodb/
        ├── notification.ts           — ✅ Notification service (CRUD, counts, mark read)
        └── notification.test.ts      — ✅ Service tests (33 tests)
```

**Already built by PE — separate branch/PR** (SignalR layer):

```
apps/web/
├── services/
│   └── signalr/
│       └── index.ts                  — ✅ Server-side service (negotiate, token generation)
├── app/api/
│   └── signalr/
│       └── negotiate/
│           └── route.ts              — ✅ POST: SignalR negotiate endpoint
└── utils/context/
    └── signalr-context.tsx           — ✅ SignalR React context + useSignalR() hook
```

**Still needed** (to be built in D1/D2/D3):

```
apps/web/
├── services/
│   └── signalr/
│       └── broadcast.ts              — Server-side broadcast function (D1-T1)
├── app/api/
│   ├── notifications/
│   │   └── route.ts                  — GET: list user's notifications (D2-T1)
│   └── notifications/[id]/read/
│       └── route.ts                  — PATCH: mark read via markUserNotificationAsRead (D2-T1)
```

---

## Phases

### Phase 0: Azure SignalR Infrastructure (prerequisite)

Provision and configure the Azure SignalR Service resource, secrets, and environment configuration. Owned by Anatoliy (Principal Engineer). This is a blocker for all other phases.

### Phase A: SignalR Integration (foundation)

Application-level SignalR integration. The PE has already built the negotiate service, client context, and notification data model. Remaining work: feature flag + server-side broadcast function (to push events to connected clients).

### Phase B: Document Notifications (feature)

Build the document notification experience: trigger SignalR events on document creation, create `user`-type notifications targeting the order's assigned user, badges in Order Tracker, "Mark as Read" flow in Documents tab, audit trail in Order History. No document model changes — all state lives in `notifications` + `notificationreads` collections.

---

## Deliverable 0: Azure SignalR Infrastructure (Phase 0) — Assigned to Anatoliy

**Goal:** Provision the Azure SignalR Service resource and make the connection string available to the application in all environments (dev, staging, prod). This is a prerequisite for all application-level work. No application code changes — infrastructure and configuration only.

**Owner:** Anatoliy (Principal Engineer)

| Task ID | Summary                                                                             | SP  | Status | Branch | Merged to Epic | Notes                                                             |
| ------- | ----------------------------------------------------------------------------------- | :-: | :----: | ------ | :------------: | ----------------------------------------------------------------- |
| D0-T1   | Provision Azure SignalR Service + Key Vault configuration + connectivity validation |  3  | To Do  | -      |       -        | [MLID-2079](https://localinfusion.atlassian.net/browse/MLID-2079) |

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

**Open questions for Anatoliy (most resolved by PE implementation):**

1. ~~**Key Vault secret name**~~ — **Resolved:** `signalr-primary-connection-string` (env override: `SIGNALR_KV_SECRET_NAME`)
2. ~~**DEV_SECRET fallback variable name**~~ — **Resolved:** Uses `SIGNALR_KV_SECRET_NAME` env var override for local dev
3. **Pricing tier for staging** — Free tier has limits (20 concurrent connections, 20K messages/day). Is Standard needed for staging, or is Free sufficient for testing?
4. **Region** — Which Azure region are the existing resources in? The SignalR resource should be co-located.
5. **Service mode** — We plan to use the service as a broadcast channel (server sends to all clients). What's the recommended mode (Default vs Serverless) for this pattern?
6. ~~**Existing Socket.IO coexistence**~~ — **Resolved:** Coexists. No migration in this epic.
7. ~~**Hub naming**~~ — **Resolved:** `li-hub`
8. **Connection limits** — Standard tier supports 1K concurrent connections per unit. How many concurrent users do we expect in prod? Do we need multiple units?
9. ~~**Client connection setup**~~ — **Resolved:** Per-user JWT tokens via `POST /api/signalr/negotiate`. Client uses `@microsoft/signalr` SDK with `withAutomaticReconnect()`.

**D0 PR to develop:** Not needed (infrastructure only, no app code)

---

## Deliverable 1: SignalR Integration (Phase A)

**Goal:** Complete the SignalR integration layer. The PE has already built: negotiate service, negotiate API route, React context + `useSignalR()` hook, `Notification` + `NotificationRead` models, and the full notification service. Remaining: feature flag + server-side broadcast function to push events to connected clients.

**Depends on:** D0 (SignalR resource must be provisioned and connection string available)

| Task ID | Summary                                                                                         | SP  | Status | Branch                                                                     | Merged to Epic | Notes                                                                                                                                                                                                                                                                |
| ------- | ----------------------------------------------------------------------------------------------- | :-: | :----: | -------------------------------------------------------------------------- | :------------: | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| D1-T1   | Feature flag `ORDER_DOCUMENT_NOTIFICATIONS` + SignalR server-side broadcast function            |  3  |  Done  | `feature/MLID-2080-signalr-broadcast-feature-flag`                         |      Yes       | [MLID-2080](https://localinfusion.atlassian.net/browse/MLID-2080)                                                                                                                                                                                                    |
| D1-T2   | Notification data model (Notification + NotificationRead Mongoose models, notification service) |  3  |  Done  | `feature/MLID-2011-notifications-data-model` (merged via PR #1049 to epic) |      Yes       | [MLID-2081](https://localinfusion.atlassian.net/browse/MLID-2081) — Delivered by PE under PR #1049 (referenced as `MLID-2011`, not the sub-task key). SignalR React context + service foundation was separately merged directly to develop via PR #1046 (MLID-2079). |

**D1 PR to develop:** Deferred — D1 alone has no user-facing behavior worth QA validation. Will merge to develop as part of D2 PR once notification API routes and document triggers are wired up, giving QA something testable end-to-end.

---

## Deliverable 2: Document Notification Wiring (Phase B)

**Goal:** Wire document creation entry points to create notifications (using PE's model) and broadcast SignalR events. Add API routes for reading and acknowledging notifications. Notifications are `user` type targeting the order's assigned user — no document model changes needed.

**Depends on:** D1

| Task ID | Summary                                                                                 | SP  | Status | Branch                                             | Merged to Epic | Notes                                                             |
| ------- | --------------------------------------------------------------------------------------- | :-: | :----: | -------------------------------------------------- | :------------: | ----------------------------------------------------------------- |
| D2-T1   | Notification API routes (list, mark read via `markUserNotificationAsRead`)              |  3  |  Done  | `feature/MLID-2082-notification-api-routes`        |      Yes       | [MLID-2082](https://localinfusion.atlassian.net/browse/MLID-2082) |
| D2-T2   | Investigate document creation entry points + create notifications + emit SignalR events |  5  |  Done  | `feature/MLID-2083-document-notification-triggers` |      Yes       | [MLID-2083](https://localinfusion.atlassian.net/browse/MLID-2083) |

**D2 PR to develop:** Deferred — will ship as a single PR with D3 once the full notification UI is built, giving QA the complete end-to-end feature to test.

---

## Deliverable 3: Notification UI — Order Tracker, Documents Tab, Order History (Phase B)

**Goal:** Full notification UI experience across all order views. (1) Order Tracker — badges and "Action Required" filter based on unacknowledged documents, (2) Documents tab — "New" tags, "Mark as Read" (single + bulk), unread badge on tab header, (3) Order History — audit trail for document additions. All update in real-time via SignalR events.

**Depends on:** D2

| Task ID | Summary                                                                        | SP  | Status | Branch | Merged to Epic | Notes                                                                                                                                                                                                                               |
| ------- | ------------------------------------------------------------------------------ | :-: | :----: | ------ | :------------: | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| D3-T1   | Order Tracker: notification badge on Order ID + "Action Required" quick filter |  5  |  Done  | `feature/MLID-2084-order-tracker-badges` |      Yes       | [MLID-2084](https://localinfusion.atlassian.net/browse/MLID-2084) — Badge visibility globalized per 2026-04-14 decision. Manual QA verified on 05HP with seeded test data. `NewOrders.tsx` component test blocked by pre-existing BSON ESM issue in `__mocks__/bson.js` (not from this ticket) — backend files at 100% coverage. |
| D3-T2   | Documents Tab: unread badge, "New" tags, "Mark as Read" (single + bulk)        |  5  |  Done  | `feature/MLID-2085-documents-tab-mark-as-read` |      Yes       | [MLID-2085](https://localinfusion.atlassian.net/browse/MLID-2085) — Manual QA verified by Alejandro 2026-04-20. SignalR real-time event verification deferred to end-of-epic E2E pass. Merged to epic via `a5c26763`. |
| D3-T3   | Order History: document audit trail entries with source differentiation        |  3  |  Done  | `feature/MLID-2086-order-history-document-audit` |      Yes       | [MLID-2086](https://localinfusion.atlassian.net/browse/MLID-2086) — Implemented 2026-04-20/21. `renderedChange` hook on `AuditFieldConfig` pre-computes display text server-side. `recordDocumentAddedToOrders()` service writes one audit entry per linked order. Wired at upload (`split-pdf/route.ts`) and e-order/fax (`weinfuseUploader.ts` with `intake.source === 'spruce'` detection). Manual QA verified on order 05Gy with seeded audit data. Merged to epic via `b6e64638`. |

**D3 PR to develop:** All three D3 tasks merged to epic — D3-T1 (`20ca9ec4`), D3-T2 (`a5c26763`), D3-T3 (`b6e64638`). Ready to ship combined D2+D3 PR to develop.

---

## Task List

### D0: Azure SignalR Infrastructure (3 SP) — Anatoliy

- D0-T1 — MLID-2079 — Provision SignalR resource + Key Vault + CORS + smoke test (3 SP)

### D1: SignalR Integration (3 SP remaining)

- D1-T1 — MLID-2080 — Feature flag + SignalR server-side broadcast function (3 SP)
- D1-T2 — MLID-2081 — Frontend: SignalR React context + provider (3 SP) — In Review (PE PR pending)

### D2: Document Notification Wiring (8 SP)

- D2-T1 — MLID-2082 — Notification API routes (list, mark read via `markUserNotificationAsRead`) (3 SP)
- D2-T2 — MLID-2083 — Investigate entry points + create notifications + emit SignalR events (5 SP)

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
- **Epic Branch**: `epic/MLID-2011-order-document-notifications`
- **Deliverables**: 4 (D0–D3)
- **Total Tasks**: 8

---

## Non-Code Tasks

| Task ID | Summary                                                               | Status |
| ------- | --------------------------------------------------------------------- | :----: |
| -       | Review designer specs at li-playground and prototype HTML             | To Do  |
| -       | Codebase investigation: document upload flows, order tracker pipeline | To Do  |
| -       | Estimate story points per task after investigation                    | To Do  |
| -       | Create Jira sub-tasks once plan is approved                           | To Do  |

---

## Resolved Questions

- **SignalR integration model** — Per-user JWT tokens via negotiate endpoint. Each browser calls `POST /api/signalr/negotiate` to get connection info. Hub name: `li-hub`. Connection string in Key Vault (`signalr-primary-connection-string`).
- **Notification data model** — PE built two Mongoose models (PR #1049): `Notification` (shared, one doc per event) + `NotificationRead` (per-user read receipts for `user`-type notifications). Two types: `task` (actionable, any team member can acknowledge, inline global read via `isRead`/`readBy`/`readAt`) and `user` (informational, per-user independent read state via `notificationreads` collection). Targeting via `targetUserId` / `targetLocationId` / `targetRole` with OR logic. Entity linking is polymorphic with open-ended `entityType` strings (`order`, `document`, `patient`, `appointment`, `clinical-review`, etc.).
- **Notification service** — Full CRUD service built by PE (PR #1049, 37 tests after review fixes): `createNotification()` (with target validation), `getNotificationsForUser()` (paginated, default limit 50, excludes expired), `getUnreadCountForUser()` (`$facet` aggregation, excludes expired), `getCountForEntity()` (with optional `unreadOnly`, excludes expired), `markTaskAsRead()` (type-guarded — only updates `task`-type, returns `null` for `user`-type), `markUserNotificationAsRead()` (idempotent, ignores duplicate key), `bulkMarkUserNotificationsAsRead()` (`insertMany({ ordered: false })`, handles `MongoBulkWriteError`), `isReadByUser()`.
- **Notification type for document events** — ~~`task` type~~ **Updated 2026-04-07: `user` type.** Only the user assigned to the order can view and acknowledge documents. Notification targets `targetUserId` (order's assigned user). Read tracking via `NotificationRead` collection using `markUserNotificationAsRead()` and `bulkMarkUserNotificationsAsRead()`.
- **Recipient targeting for document notifications** — ~~`targetLocationId` (all users at the clinic location)~~ **Updated 2026-04-07: `targetUserId`** (the order's assigned user). Single user sees and acknowledges.
- **SignalR client context** — PE built `SignalRProvider` + `useSignalR()` hook with auto-reconnect (`[0, 2000, 5000, 10000, 30000]` delays).
- **Notification retention** — PE added `expiresAt` field with sparse index (intentionally NOT a TTL index — no auto-deletion). Notifications with `expiresAt` in the past are excluded from queries via `buildUserTargetFilter()` but remain in the collection. Notifications with `expiresAt: null` (the default) live forever.
- **Offline/reconnect** — `@microsoft/signalr` SDK has `withAutomaticReconnect()`. On reconnect, client re-fetches data from existing REST APIs. No gap detection needed — MongoDB is the source of truth.
- **Existing Socket.IO** — Coexists. No migration in this epic. Call-status events stay on Socket.IO.
- **Azure SignalR infrastructure** — Owned by Anatoliy (D0). Free tier for dev, Standard for staging/prod.

## D3-T3 Refinement (added 2026-04-20)

### Corrected understanding

**This task is NOT about surfacing notifications in Order History. It's about writing an entry to the existing `auditLogs` collection every time a document is added to an order, so the existing Order History tab displays it.**

The previous plan conflated notifications with history. They're independent: notifications (already built in D2) drive the real-time badges/chips; audit logs (D3-T3) drive the permanent history tab entries.

### Existing infrastructure (confirmed via codebase investigation)

**Collection:** `auditLogs` — accessed via `COLLECTION.AuditLogs`.

**Service layer:** `apps/web/services/mongodb/auditLog.ts`
- `writeAuditLog({ entityType, entityId, action, actorId, changes, source })` — inserts a row. Fire-and-forget; swallows and logs errors.
- `getAuditLog(params)` — paginated read with `$lookup` against `users` for actor name.
- `getAuditLogFilterOptions(entityType, entityId)` — distinct fields/categories/actors for the filter UI.

**Schema** (`apps/web/types/auditLog.ts`):
```typescript
{
  entityType: 'order',            // currently hardcoded — no change needed
  entityId: string,               // order _id
  action: 'create' | 'update' | 'delete',
  actorId: string,                // userId, or '' for system (renders as "System")
  changes?: [{ field, from, to }],
  source: 'user' | 'system' | 'webhook' | 'job',
  timestamp: Date
}
```

**Field→label/category config:** `apps/web/services/mongodb/auditLog.config.ts` — `ORDER_FIELD_CONFIG` map. Unknown fields fall through `humanizeFieldName` + category "Other".

**Renderer:** `AuditLogTable.tsx` → `flattenEntries`:
- `action: 'create'` → single row "Order Created", no change text
- `action: 'update'` with `changes[]` → one row per change, formatted as `Original value: X\nNew value: Y`

**Order History tab:** Already exists at `apps/web/app/orders-tracker/[category]/[orderId]/order-history/` with page, hooks, filters, and table. **No UI work needed** — the row will render automatically once the audit entries are written.

### Document → order linkage (already built in D2-T2)

`apps/web/services/notifications/documentNotification.ts` — `notifyDocumentCreated()` already performs the full document → patient → active `neworders` lookup:

1. Looks up patient by `patientId` → reads `we_infuse_id_IV`.
2. Finds all active `neworders` where `patient.we_infuse_id_IV` matches and `deleted !== true`.
3. Iterates each order; per order, creates notification + broadcasts SignalR event.

**This is the right place to piggyback the audit log write** — same loop, same per-order scope, same entry points. One audit log row per active order per document.

### Fitting document additions into the existing `auditLogs` schema

The `auditLogs` schema is entity-agnostic (`entityType` is a discriminator — designed for `'order'` today, more entities later) and `changes[].from` / `changes[].to` are typed `unknown` — they already accept scalar references to other collections (`assignedTo: '<userId>'`, `provider: '<providerNpi>'`) as well as primitive values (`dueDate`, `notes`). Adding a document to an order is just another relation change, following the same pattern, with no schema extensions required.

**Data shape stored in the collection** — `changes[0]`:

```typescript
{
  field: 'documents',       // name of the relation on the order
  from: null,               // document did not exist before this event
  to: {                     // snapshot of what was added — preserved forever
    _id: '<documentId>',    // document id (always string for consistency)
    category: 'Lab Results',// human-readable document category
    source: 'fax',          // 'fax' | 'e-order' | 'upload'
  },
}
```

Why this fits cleanly:

- **No new fields on `AuditLogChange`.** `to: unknown` already accepts object values.
- **No overloading of the entry-level `source` enum.** Document origin lives on the `to` snapshot as a property; the entry-level `source` still means "who wrote the audit row" (`user` / `system` / `webhook` / `job`).
- **`from: null, to: <object>` = "added" semantic**, mirroring the way other relation fields (`assignedTo`, `provider`) read before/after. A future `documentRemoved` event would use the same field with `from: <snapshot>, to: null`.
- **No new action type.** Stays `action: 'update'` — consistent with the "something about this order changed" meaning of an update action. Other options rejected:
  - `action: 'create'` — already means "order was created" in the renderer (`AuditLogTable.tsx:31-40`).
  - A new `action: 'relation-added'` — ripples through types, API filters, the filter UI, and the `AuditAction` union. Not worth it for a visual-only concern.

### Designer reference (Screenshot 2026-04-20 100214.png)

Row layout matches the existing table:

| Date & Time | Field            | Category      | Change                                                    | User   |
| ----------- | ---------------- | ------------- | --------------------------------------------------------- | ------ |
| 10/8/2025   | **New Document** | **Documents** | **New Document received (Document type = Lab Results)**   | System |

**The "Change" cell text is built in the renderer, not read from the collection.** The existing `AuditLogTable.flattenEntries` at `AuditLogTable.tsx:48` constructs the two-line `Original value: ${change.from}\nNew value: ${change.to}` string from raw `from`/`to` values. We follow the same pattern — the renderer reads the structured `to` object and formats it.

**Text format (literal):**
```
New Document received (Document type = <to.category>)
```

Source (`to.source`) is stored but not displayed in this iteration — the designer reference shows no source in the Change text. Source stays on the snapshot so a future UI change (e.g., a source column or suffix) can surface it without a data migration.

### Gap Resolutions — D3-T3

#### Gap 1 — Renderer has no single-description cell format

**Decision:** Extend `AuditFieldConfig` with an optional `renderChange?: (change: AuditLogChange) => string` hook. When set, `flattenEntries` delegates to the hook instead of the default `Original value: X\nNew value: Y` formatter. This keeps the renderer generic (no hardcoded field checks) and makes the mapping discoverable via the config.

For the `documents` field, the hook reads the structured `to` snapshot:

```typescript
documents: {
  label: 'New Document',
  category: 'Documents',
  renderChange: (change) => {
    const to = change.to as { category?: string } | null;
    return `New Document received (Document type = ${to?.category ?? 'Unknown'})`;
  },
}
```

The hook is free to read any field on the snapshot object — keeps display decisions out of the data layer and co-located with the field config.

#### Gap 2 — Storing "source" (Fax / E-Order / Upload) without new schema fields

**Decision:** Put `source` on the `to` snapshot object, not on a new top-level field. The `AuditLogChange` schema already allows `to: unknown`, so a nested `source` property is free — it's just a property on the snapshot.

```typescript
to: { _id, category, source }   // source lives here
```

Rejected alternatives:

- **New `meta?: Record<string, unknown>` field on `AuditLogChange`** — unnecessary given `to` already accepts objects. Adding a field would be an outward schema change for a concern that's already expressible.
- **Overload the entry-level `source` enum** — that field currently means "who wrote the audit row" (user vs webhook vs job). Overloading it with document origin would break the meaning for every consumer.
- **A second change entry `{ field: 'documentSource', to: 'Fax' }`** — would render as a second row in the history table with no designer guidance on how it should look.

**Display status:** not rendered in this iteration (per the designer reference text). Stored so a future UI change can surface it without a data migration.

**Source → label mapping** (mirrors `documentNotification.ts` `SOURCE_LABELS`):
- `upload` → `Upload`
- `e-order` → `E-Order`
- (future) `fax` → `Fax`

#### Gap 3 — `actorId` and audit-log `source` for document events

**Decision:**
- **E-Order / webhook ingestion:** `actorId: ''` (renders as "System"), audit-log `source: 'webhook'`.
- **Manual upload (logged-in user):** `actorId: session.user.id`, audit-log `source: 'user'`.
- **System-generated (jobs, migrations):** `actorId: ''`, `source: 'system'`.

The entry point decides; `notifyDocumentCreated`'s caller passes `actorId` and `source` forward to the audit write.

#### Gap 4 — Order History tab shown above the Documents tab — consistency

**Decision:** No changes to the Order History tab UI. Entries appear automatically via the existing page pipeline because `entityType: 'order'` + `entityId: orderId` already scope the query. The new field "New Document" will appear in the filter dropdowns automatically via `getAuditLogFilterOptions` (distinct field values).

### Implementation sketch (for the plan-task phase)

1. Extend `AuditFieldConfig` type with an optional `renderChange?: (change: AuditLogChange) => string` hook.
2. Add `documents` to `ORDER_FIELD_CONFIG`:
   ```typescript
   documents: {
     label: 'New Document',
     category: 'Documents',
     renderChange: (change) => {
       const to = change.to as { category?: string } | null;
       return `New Document received (Document type = ${to?.category ?? 'Unknown'})`;
     },
   }
   ```
3. Update `AuditLogTable.flattenEntries` to check `getFieldConfig(change.field).renderChange` — if defined, use it; otherwise keep the existing two-line format. Serialization note: existing rendering calls `displayValue(change.from)` / `displayValue(change.to)` which coerces objects via `String()` — that produces `[object Object]`. The `renderChange` branch runs before `displayValue`, so object-valued changes only need to reach the default path for fields that don't define a hook (and today, no existing field stores an object — so there's no regression risk).
4. Create `services/audit/documentAudit.ts` with `recordDocumentAddedToOrders({ documentId, patientId, category, source, actorId })`. Mirrors `notifyDocumentCreated`'s patient → `neworders` lookup (same `we_infuse_id_IV` → `patient.we_infuse_id_IV` + `deleted !== true` filter) and writes one `writeAuditLog` row per order:
   ```typescript
   writeAuditLog({
     entityType: 'order',
     entityId: orderId,
     action: 'update',
     actorId,                // userId for uploads, '' for webhook/system
     source: actorId ? 'user' : 'webhook',
     changes: [{
       field: 'documents',
       from: null,
       to: { _id: documentId, category, source },
     }],
   })
   ```
5. Wire it at the same entry points that call `notifyDocumentCreated` (both calls sit next to each other — same data needed, same fire-and-forget semantics). Confirm the call signature exposes `actorId` at each entry point.
6. Unit tests for:
   - The renderer: hook resolution precedence (hook wins over default format); default format preserved for fields without a hook.
   - `getFieldConfig('documents')` returns the config with the hook; unknown fields still fall through to `humanizeFieldName`.
   - `recordDocumentAddedToOrders`: patient-not-found / no-we-infuse-id / no-orders / assigned-and-unassigned orders / partial failure (one of N orders throws — the others still get a row); feature-flag gating.
   - Integration: at each document entry point, audit write fires with the correct payload.

### Alternative considered (rejected)

**Bundle the audit write inside `notifyDocumentCreated`.** Rejected — mixes two concerns (transient push + permanent history) into one function. They share lookup logic but have independent failure modes, feature-flag gating semantics, and testing surfaces. Better to keep them parallel and call both from the same entry points.

### Feature-flag gating

**Decision:** **Gated behind `ORDER_DOCUMENT_NOTIFICATIONS`** alongside notifications. If the epic is toggled off, no audit entries are written either — consistent with the existing flag's "this epic is live" semantics. Revisit if the flag survives post-epic (it probably should not).

### Remaining open questions

1. **Document categories used for the "Document type" text** — confirm the category string we receive from each entry point is the human-readable label (e.g., "Lab Results") or an enum key we need to format. `documentNotification.ts` passes `category` through raw — need to verify against actual document schema.
2. **Fax ingestion entry point** — D2-T2 enumerated only `upload` and `e-order`. The epic mentions "Fax" explicitly. Does fax go through a separate entry point, or is it folded into `upload` via the fax → upload gateway? This affects whether we need a third `source` value.

---

## Open Questions

1. **Existing document upload paths** — Need to catalog all entry points where documents get attached to orders (webhooks, manual upload, fax integration, e-order). Investigation task D2-T2.
2. ~~**Order History tab**~~ — **Resolved 2026-04-20:** tab already exists at `/orders-tracker/[category]/[orderId]/order-history`, backed by the existing `auditLogs` collection. D3-T3 writes to this collection — no UI work on the tab itself.
3. ~~**`task` vs `user` type for document events**~~ — **Resolved (updated 2026-04-07):** `user` type. Only the user assigned to the order can view and acknowledge documents. Per-user read tracking via `NotificationRead` collection.
4. ~~**`acknowledged` flag on documents**~~ — **Resolved:** Not needed. Badge counts are driven entirely from `Notification` + `NotificationRead` queries. No document model changes required.
5. ~~**Recipient targeting for document notifications**~~ — **Resolved (updated 2026-04-07, amended 2026-04-14):** `targetUserId` still set to the order's assigned user for Mark-as-Read ownership, but entity-scoped badge counts are now **globally visible** to all authenticated users (see "Update 2026-04-14 — Decouple Badge Visibility from Notification Ownership" section).
6. **Server-side broadcast** — PE built negotiate + token generation but not the broadcast function. Need to build `broadcastToRoom()` or `sendToAll()` to push events to connected clients.
7. **SignalR env vars need to be set up in staging/prod infra** — As of D1-T1, three SignalR constants in `apps/web/services/signalr/index.ts` now read from env vars with fallback defaults:

   - `SIGNALR_HUB_NAME` (default: `li-hub`)
   - `SIGNALR_TOKEN_EXPIRY` (default: `1h`)
   - `SIGNALR_REST_API_VERSION` (default: `2022-06-01`)

   **Local dev is set up** — all three are defined in `apps/web/.env.local`.

   **Infra-side gap (action needed)** — I don't know how Anatoliy pushes env vars into our Azure infra (container env, Key Vault references, terraform variables, App Service config, etc.). Need guidance from him on how to add these three env vars to **staging** and **prod** environments so they're available when the app runs in the cloud. Fallback defaults in code mean nothing breaks if he doesn't add them — but ideally they should be explicit per environment so per-env overrides are possible.

   **Action:** Ask Anatoliy how env vars are wired up in staging/prod and coordinate adding the three `SIGNALR_*` vars. Not blocking any current work — fallback defaults are correct for now.
