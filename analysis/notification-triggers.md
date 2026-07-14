# Notification triggers — how order-scoped notifications are created

This document describes every event that **creates** an order-scoped notification (a row in the `notifications` collection), the functions involved, where in the app each trigger lives, and the call chain that ends in a durable notification write.

Scope: "order-scoped notification" means a `notifications` document whose `entities` array contains both an `order` tuple and a `document` tuple, targeted at the order's assignee. These are the notifications the orders-tracker table badge, the Documents-tab badge, and the "New" chip all count. Feature flag `ORDER_DOCUMENT_NOTIFICATIONS` gates every trigger below — when it is off, nothing is created.

## The shared orchestrator

All triggers funnel through one function:

- **`propagateDocumentNotificationToOrder(params)`** in `apps/web/services/notifications/documentNotification.ts`.

For a single `(order, document)` pair it runs three isolated steps, each in its own `try/catch` so one failure does not abort the others:

1. **Notification write** — `writeOrderDocumentNotification` → `createNotification` (`apps/web/services/mongodb/notification.ts`), which inserts the `notifications` row (`type: 'user'`, `targetUserId: order.assignedTo`, entities = order + document).
2. **Audit write** — `writeOrderDocumentAuditEntry` (`apps/web/services/audit/documentAudit.ts`), which adds the order-history "document added" entry.
3. **SignalR broadcast** — `broadcastDocumentAddedEvent` → `signalr.broadcastToAll('documentAdded', …)` so open clients update their badges live without a reload.

Before those steps, the orchestrator applies two gates and returns early if either fails:

- **Eligibility gate** — `isDocumentEligibleForOrder(document, order)` (`apps/web/services/notifications/documentEligibility.ts`). Filters out an Order-category document that is not linked to *this* specific order. Mirrors the Documents-tab filter so the tab list and every propagation surface share one rule.
- **Assignee gate** — if `order.assignedTo` is falsy, all three side-effects are deferred (no orphan audit rows, no notification with no recipient).
- **Self-action suppression** — if the actor equals `order.assignedTo`, only the in-app notification is skipped (audit + SignalR still run).

The triggers differ only in their **fan-out shape** — which orders and which documents they hand to the orchestrator.

---

## Section 1 — Triggers list (short description)

1. **Document uploaded through intakes review** — a document saved from the split-PDF intake flow fans out to all of the patient's new and maintenance orders.
2. **Document uploaded through e-order / fax** — a document saved by the background upload job fans out to all of the patient's new and maintenance orders.
3. **New order created** — backfills the patient's existing documents onto the newly-created new order.
4. **Maintenance order created** — backfills the patient's existing documents onto the newly-created maintenance order.
5. **Document linked to one or more orders** — notifies each order in the newly-linked delta (the path that fires for an Order-category document).
6. **Order gets its first assignee (initial assignment)** — backfills the patient's existing documents onto an order that was created without an assignee, now that there is someone to notify.

Related events that do **not** create notifications:

- **Order reassigned (populated → different user)** — *retargets* existing notifications (flips `targetUserId`); creates nothing new.
- **Document unlinked from an order** — *deletes* the matching notification rows.

---

## Section 2 — Detailed explanation per trigger

### 1. Document uploaded through intakes review

- **Where in the app:** the intake review / split-PDF upload flow. Route `apps/web/app/api/documents/split-pdf/route.ts` (two upload paths, both call the same entry point).
- **Entry point:** `notifyDocumentCreated({ documentId, patientId, category, source: 'upload', actorId, auditSource })` in `documentNotification.ts`.
- **Fan-out shape:** one document → many orders. Looks up every order for the patient via `getOrdersByPatient(patientId, { expandRefs: false, collections: ['new', 'maintenance'] })` and calls the orchestrator once per order.
- **Call chain:**
  `POST /api/documents/split-pdf` → `createDocumentRecord` → `notifyDocumentCreated` → `getOrdersByPatient` → for each order: `propagateDocumentNotificationToOrder` → `writeOrderDocumentNotification` → `createNotification`.
- **Note:** at upload time a document is born unlinked (`linkedOrderIds: []`), so an Order-category document uploaded here is filtered by the eligibility gate until it is later linked (see trigger 5).

### 2. Document uploaded through e-order / fax

- **Where in the app:** the background upload job that ingests e-order and fax documents. `apps/web/services/jobs/definitions/jotFormSubmission/weinfuseUploader.ts`.
- **Entry point:** `notifyDocumentCreated({ …, source: 'e-order' | 'fax' })` (the source label is derived from the intake source; `spruce` maps to `fax`, otherwise `e-order`).
- **Fan-out shape:** identical to trigger 1 — one document → all of the patient's new and maintenance orders.
- **Call chain:**
  upload job → `createDocumentRecord` → `notifyDocumentCreated` → `getOrdersByPatient` → per order: `propagateDocumentNotificationToOrder` → `createNotification`.
- **Difference from trigger 1:** only the `source` label on the notification message ("via E-Order" / "via Fax" vs "via Upload"); the fan-out and orchestration are the same.

### 3. New order created

- **Where in the app:** the new-orders creation route. `apps/web/app/api/orders/new/route.ts` (POST handler, after `autoAssignOrders`).
- **Entry point:** `notifyOrderOfRecentDocuments({ orderId, assignedTo, patientId })` in `documentNotification.ts`.
- **Fan-out shape:** one new order → many documents. Loads the patient's documents via `getDocumentsByPatientId(patientId, { since })` and calls the orchestrator once per document. The `since` window is controlled by `BACKFILL_LOOKBACK_DAYS` in `documentNotification.ts` (currently `-1` = disabled = all documents regardless of age).
- **Call chain:**
  `POST /api/orders/new` → `createOrder(..., 'new')` → `autoAssignOrders` → for each created order: `notifyOrderOfRecentDocuments` → `getDocumentsByPatientId` → per document: `propagateDocumentNotificationToOrder` → `createNotification`.
- **Purpose:** an order created *after* its documents already exist would otherwise miss them (the document-side triggers only fire at document-creation time). This backfill closes that gap for new orders.

### 4. Maintenance order created

- **Where in the app:** the maintenance-orders creation route. `apps/web/app/api/orders/maintenance/route.ts` (POST handler, after `autoAssignOrders`).
- **Entry point:** `notifyOrderOfRecentDocuments({ orderId, assignedTo, patientId })` — the same function as trigger 3.
- **Fan-out shape:** one new maintenance order → the patient's existing documents (same backfill semantics as trigger 3).
- **Call chain:**
  `POST /api/orders/maintenance` → `createOrder(..., 'maintenance')` → `autoAssignOrders` → for each created order: `notifyOrderOfRecentDocuments` → `getDocumentsByPatientId` → per document: `propagateDocumentNotificationToOrder` → `createNotification`.
- **History:** this trigger was missing from the maintenance route — the new-orders route (trigger 3) had it, but the maintenance route did not, so intake- and manually-created maintenance orders never backfilled notifications for pre-existing documents. Added by hotfix branch `hotfix/MLID-2447-maintenance-order-notification-backfill` (see `docs/agomez/postmortem/MLID-2447.md`). Because the intake flow POSTs maintenance orders to this same route, both intake and manual creation are covered by the single fix.

### 5. Document linked to one or more orders

- **Where in the app:** the link-orders action. `apps/web/app/api/documents/[documentId]/link-orders/route.ts` (PUT handler).
- **Entry point:** `handleOrderLink({ documentId, document, newOrders, actorId, auditSource })` in `documentNotification.ts`.
- **Fan-out shape:** one document → the *delta* set of newly-linked orders only (the caller computes which orderIds were newly added; re-saves and removals do not fire this). Each newly-linked order carries its collection (`'new' | 'maintenance'`), so the order resolves in a single `getOrderById(orderId, category)` call.
- **Call chain:**
  `PUT /api/documents/[documentId]/link-orders` → update the document's `linkedOrderIds` → `handleOrderLink` → per newly-linked order: `getOrderById` → `propagateDocumentNotificationToOrder` → `createNotification`.
- **Why it matters:** this is the path that lets an **Order-category** document produce a notification. Order-category documents are filtered by the eligibility gate everywhere else (they only notify the order they are actually linked to), so the link action is where their single notification is created.

### 6. Order gets its first assignee (initial assignment)

- **Where in the app:** the order *update* routes, when `assignedTo` goes from empty to populated. Both `apps/web/app/api/orders/new/route.ts` and `apps/web/app/api/orders/maintenance/route.ts` (PUT handlers) call the same helper.
- **Entry point:** `handleOrderReassignment(order, previousAssignedTo, newAssignedTo)` in `apps/web/services/notifications/handleOrderReassignment.ts`. It branches:
  - **Initial assignment** (`previousAssignedTo` empty, `newAssignedTo` populated) → calls `notifyOrderOfRecentDocuments` (same backfill as triggers 3/4) — **this creates notifications**.
  - **Standard reassignment** (both populated, different) → calls `reassignUserNotificationsForOrder` — this **retargets** existing notifications, creating nothing new.
- **Call chain (initial assignment):**
  `PUT /api/orders/{new|maintenance}` → `updateOrder` → per updated order: `handleOrderReassignment` → `notifyOrderOfRecentDocuments` → `getDocumentsByPatientId` → per document: `propagateDocumentNotificationToOrder` → `createNotification`.
- **Why it exists:** the orchestrator's assignee gate suppresses notification writes when an order has no assignee. An order created at an IG-less location therefore skips its documents at creation time; this trigger re-runs the backfill once an assignee finally exists, so the new assignee sees the documents that arrived while the order was unassigned.

---

## Non-creating events (for completeness)

- **Order reassigned (populated → different user)** — `handleOrderReassignment` takes its retarget branch: `reassignUserNotificationsForOrder(orderId, previousAssignedTo, newAssignedTo)` flips `targetUserId` on the existing rows. Prior `notificationreads` rows stop matching the unread join, so the new assignee sees everything as unread. No new notification rows are written.
- **Document unlinked from an order** — the link-orders route computes the removed delta and calls `handleOrderUnlink`, which deletes the matching `notifications` rows and their `notificationreads`, writes a remove-side audit entry, and broadcasts `documentUnlinked`. This removes notifications rather than creating them.
