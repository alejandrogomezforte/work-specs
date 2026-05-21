# MLID-2086 — Unlinked Order-Category Documents: Eligibility-Gated Propagation

## Task Reference

- **Jira**: [MLID-2086](https://localinfusion.atlassian.net/browse/MLID-2086)
- **Story Points**: 3
- **Branch**: `feature/MLID-2086-unlinked-order-document-filter` (off `develop`)
- **Base Branch**: `develop`
- **Status**: Planning (QA rejected previous attempt)

---

## Summary

When a document of `category === 'Order'` is uploaded, it starts unlinked (`linkedOrderIds: []`) and only becomes associated with a specific LISA order after the user links it via the patient documents UI (`/patient/[id]/document-records` → "Update Order Linking"). Per business rules:

> An uploaded document of category "Order" is only allowed to appear in an order's Documents tab, Order History log, and notification stream **if it is linked to that specific order**.

Today the Documents tab list already enforces this (`apps/web/app/api/orders/details/route.ts:435-451`), but the three sibling propagation surfaces — notification rows, SignalR broadcasts, and Order History audit entries — do not. They fan out to every order belonging to the patient regardless of category or linkage state.

This plan replaces the current "fan-out lives inside each side-effect function" design with an inverted-control orchestrator:

- A single function `propagateDocumentNotificationToOrder(order, document, ...)` owns the eligibility check and runs three independent steps in order: **notification row → audit log → SignalR broadcast**. Each step has its own try/catch and its own structured log so a failure in one does not abort the others.
- Call sites enumerate the orders themselves and hand each `(order, document)` pair to the orchestrator. The eligibility rule lives in exactly one place — the orchestrator — and matches the predicate already in use by the Documents tab filter.
- The orchestrator must be invoked from **all three triggers** that propagate documents to orders:
  1. **Document uploaded** via /intakes (manual `split-pdf` route and the `weinfuseUploader` job).
  2. **New order added** — backfill the patient's recent documents to the new order.
  3. **Document linked** to an order via /patient/document-records — fire once for each newly-added order id (delta against the previous `linkedOrderIds`), so re-saving the same form does not duplicate entries.

### The Eligibility Rule (single source of truth)

```ts
function isDocumentEligibleForOrder(document, order): boolean {
  if (document.category !== 'Order') return true;
  return (document.linkedOrderIds ?? []).includes(String(order._id));
}
```

This is the exact predicate already used at `apps/web/app/api/orders/details/route.ts:435-440`. Centralizing it in the orchestrator means the Documents tab list and the propagation pipeline share one rule.

---

## Codebase Analysis

### Document Model — `apps/web/models/Document.ts`

- The collection is `documents`.
- `category?: string` is free-form. The value `'Order'` (capital-O) is the discriminator at issue.
- `linkedOrderIds?: string[]` — declared at lines 36-37 (interface) and 117-120 (schema). Defaults to `[]` via `default: []`.
- Verified by grep: the only writer of `linkedOrderIds` in the codebase is `linkDocumentToOrders` (`apps/web/services/mongodb/document.ts:310-327`), which itself is only called from the link-orders route. The `split-pdf` upload route, the `weinfuseUploader` job, and the `migrate-intake-documents` script all rely on the schema default. **At upload time, every Order-category document has `linkedOrderIds: []`.**

### Today's Propagation Functions

**`notifyDocumentCreated`** (`apps/web/services/notifications/documentNotification.ts:163-208`)
- Internally calls `getOrdersByPatient(patientId, { expandRefs: false })` and loops over every order with an `assignedTo`.
- For each order calls `dispatchOrderDocumentNotification`, which calls `writeOrderDocumentNotification` (try/caught) then `broadcastDocumentAddedEvent` (try/caught).
- **No category or linkage check.** Order-category documents fan out to every patient order.

**`recordDocumentAddedToOrders`** (`apps/web/services/audit/documentAudit.ts:17-62`)
- Internally calls `getOrdersByPatient(patientId, { expandRefs: false })` and loops.
- For each order calls `writeAuditLog` with `field: 'documents'`, `to: { _id, category, source }`.
- The per-order `writeAuditLog` is **inside** the outer try/catch, so if one fails the rest of the loop is aborted.
- **No category or linkage check.**

**`notifyOrderOfRecentDocuments`** (`apps/web/services/notifications/documentNotification.ts:227-291`)
- Order-creation backfill: when a new order is created, surface recent (last 30 days) patient documents to it.
- Different shape: one order, many documents. Today writes one notification per doc and emits a single coalesced SignalR broadcast at the end (rationale in the docstring: "20 documents would trigger 20 round-trips per connected user").
- Does **not** write audit entries today.
- **No category or linkage check.** An unlinked Order doc surfaces as a notification on the new order today.
- After this refactor: becomes a thin caller-side fan-out that calls the orchestrator for each `(newOrder, existingDocument)` pair. The orchestrator's eligibility check filters unlinked Order docs. Notification + audit + SignalR all fire per eligible doc. The coalesced SignalR optimization is **dropped on purpose** — per team convention (`feedback_signalr_cheap_mongo_expensive`), SignalR broadcasts are cheap and we do not coalesce them; M Mongo writes are the real cost, and they stay at M either way.

### Today's Call Sites

| Caller | File | Lines | Notes |
|---|---|---|---|
| Manual upload (full intake) | `apps/web/app/api/documents/split-pdf/route.ts` | 350-365 | calls both `notifyDocumentCreated` + `recordDocumentAddedToOrders`, `source: 'upload'`, `auditSource: 'user'` |
| Manual upload (split intake) | `apps/web/app/api/documents/split-pdf/route.ts` | 602-617 | same shape |
| Webhook (JotForm → WeInfuse) | `apps/web/services/jobs/definitions/jotFormSubmission/weinfuseUploader.ts` | 344-359 | `source` is `'fax'` or `'e-order'` based on `intake.source`, `auditSource: 'webhook'` |
| Order creation backfill | `apps/web/services/notifications/documentNotification.ts:227` | — | called from order-creation flow (not changing the call site, only the function body) |
| **NEW** link-time trigger | `apps/web/app/api/documents/[documentId]/link-orders/route.ts` | 74 (after) | add per-delta-order orchestrator call after successful `linkDocumentToOrders` |

### Documents Tab Filter (precedent — unchanged)

`apps/web/app/api/orders/details/route.ts:435-451`:

```ts
documents: documents.filter(
  (doc) =>
    doc.category !== 'Order' ||
    (doc.linkedOrderIds ?? []).includes(String(trackerOrder._id))
)
```

This is the predicate we mirror inside the orchestrator. No change to the route.

### Link Endpoint Today

`apps/web/app/api/documents/[documentId]/link-orders/route.ts:12-91`:
- Auth + role check, JSON body parse, ObjectId validation
- Fetches `doc` via `getDocumentById(documentId)` on line 59
- Enforces `doc.category === 'Order'` on lines 67-72
- Calls `linkDocumentToOrders(documentId, sanitizedIds)` on line 74, which does `$set: { linkedOrderIds: orderIds }` — replaces the entire array, no delta tracking

The route has the previous `linkedOrderIds` available (on `doc`) before the `$set` runs, so the delta can be computed in the route handler without changing the service signature.

---

## Architecture

### File Layout

The orchestrator lives in `documentNotification.ts` alongside its per-order primitives. The audit step is exported from `documentAudit.ts` as a single-order helper and imported into the orchestrator. This keeps the leaf modules focused (notifications service vs audit service) while letting the orchestrator coordinate.

**`apps/web/services/notifications/documentNotification.ts`** — refactored:
- Keep: `writeOrderDocumentNotification(params)` (already has own try/catch + logger; rename to clarify it's the notification step)
- Keep: `broadcastDocumentAddedEvent(signalr, payload)` (already has own try/catch + logger)
- Add: `propagateDocumentNotificationToOrder(order, document, source, actor)` — the orchestrator
- Refactor: `notifyDocumentCreated(params)` — becomes a thin caller-side fan-out helper that enumerates `getOrdersByPatient` and calls the orchestrator for each `(order, document)` pair. Eligibility logic is removed from this function — it lives in the orchestrator.
- Refactor: `notifyOrderOfRecentDocuments(params)` — becomes a thin caller-side fan-out. Iterates the patient's recent documents and calls `propagateDocumentNotificationToOrder({ order, document, ... })` for each. The orchestrator handles eligibility, notification, audit, and SignalR per item. The previous coalesced-SignalR optimization is dropped (M broadcasts is acceptable; M DB writes are the real cost and they stay at M).

**`apps/web/services/audit/documentAudit.ts`** — refactored:
- Add: `writeOrderDocumentAuditEntry(params)` — per-order audit step with own try/catch + structured log. Mirrors the shape of `writeOrderDocumentNotification`.
- Refactor: `recordDocumentAddedToOrders(params)` — fan-out across patient orders becomes the caller's responsibility (via the orchestrator). The exported per-order step is what the orchestrator calls.

**`apps/web/app/api/documents/[documentId]/link-orders/route.ts`** — new trigger:
- Before `$set`: capture `previousLinkedOrderIds = doc.linkedOrderIds ?? []`
- After successful `linkDocumentToOrders`: compute `newlyAddedOrderIds = sanitizedIds.filter(id => !previousLinkedOrderIds.includes(id))`
- For each `orderId` in `newlyAddedOrderIds`, fetch the order via `getOrderById(orderId, 'new')` (fall back to `'maintenance'` if not found, mirroring the MLID-2085 fallback) and call `propagateDocumentNotificationToOrder`. Fire-and-forget with `void` (matches the upload-path pattern).

### Orchestrator Shape

```ts
// apps/web/services/notifications/documentNotification.ts

import { writeOrderDocumentAuditEntry } from '@/services/audit/documentAudit';
import { isDocumentEligibleForOrder } from '@/services/notifications/documentEligibility';

export interface PropagateDocumentNotificationToOrderParams {
  order: { _id: unknown; assignedTo?: string | null };
  document: { _id: unknown; category?: string; linkedOrderIds?: string[] };
  source: DocumentNotificationSource;
  actorId: string;
  auditSource: AuditSource;
  getSignalR: SignalRResolver;
}

export async function propagateDocumentNotificationToOrder(
  params: PropagateDocumentNotificationToOrderParams
): Promise<void> {
  const { order, document, source, actorId, auditSource, getSignalR } = params;
  const orderId = String(order._id);
  const documentId = String(document._id);

  if (!isDocumentEligibleForOrder(document, order)) {
    logger.info(
      'Document propagation skipped — ineligible (unlinked Order-category document)',
      { orderId, documentId, category: document.category }
    );
    return;
  }

  // Step 1: Notification row (own try/catch inside)
  if (order.assignedTo) {
    await writeOrderDocumentNotification({
      orderId,
      assignedTo: order.assignedTo,
      documentId,
      category: document.category ?? 'Document',
      source,
    });
  }

  // Step 2: Audit log entry (own try/catch inside)
  await writeOrderDocumentAuditEntry({
    orderId,
    documentId,
    category: document.category ?? 'Document',
    documentSource: source,
    actorId,
    auditSource,
  });

  // Step 3: SignalR broadcast (own try/catch inside)
  const signalr = await getSignalR();
  if (signalr) {
    await broadcastDocumentAddedEvent(signalr, {
      orderId,
      documentId,
      category: document.category ?? 'Document',
      source,
    });
  }
}
```

The eligibility predicate is extracted to its own file (`apps/web/services/notifications/documentEligibility.ts`) so it can be imported by tests and by the Documents tab filter later without circular deps.

### Step Independence

Each step is wrapped in its own try/catch inside its own function. The orchestrator awaits them sequentially but each step's failure path is internal — none of them throw. So:

- If `writeOrderDocumentNotification` fails → logged with `orderId`, `documentId`, `error`. Audit and SignalR still run.
- If `writeOrderDocumentAuditEntry` fails → logged. SignalR still runs.
- If `broadcastDocumentAddedEvent` fails → logged. Already last; no downstream effect.

Each step's success path also logs an `info` line with `orderId` and `documentId` so we can correlate in App Insights.

---

## Implementation Steps

### Step 1 — Extract eligibility predicate

- **Files**:
  - `apps/web/services/notifications/documentEligibility.ts` (new)
  - `apps/web/services/notifications/documentEligibility.test.ts` (new)

- **What**:

  RED — Write the test file with these cases:

  ```
  describe('isDocumentEligibleForOrder', () => {
    it('returns true for non-Order documents regardless of linkage')
    it('returns true for Order-category documents whose linkedOrderIds includes the order id')
    it('returns false for Order-category documents whose linkedOrderIds is empty')
    it('returns false for Order-category documents whose linkedOrderIds does not include the order id')
    it('returns false for Order-category documents with undefined linkedOrderIds')
    it('compares order._id via String() to handle ObjectId vs string inputs')
  })
  ```

  GREEN — Create the predicate:

  ```ts
  // apps/web/services/notifications/documentEligibility.ts
  export interface EligibilityDocument {
    category?: string;
    linkedOrderIds?: string[];
  }
  export interface EligibilityOrder {
    _id: unknown;
  }

  export function isDocumentEligibleForOrder(
    document: EligibilityDocument,
    order: EligibilityOrder
  ): boolean {
    if (document.category !== 'Order') return true;
    return (document.linkedOrderIds ?? []).includes(String(order._id));
  }
  ```

  REFACTOR — Confirm typing reuses existing `IDocument` and `OrderRef` shapes where appropriate (or keeps a narrow interface so non-Mongoose callers can use it too).

### Step 2 — Add per-order audit step `writeOrderDocumentAuditEntry`

- **Files**:
  - `apps/web/services/audit/documentAudit.ts`
  - `apps/web/services/audit/documentAudit.test.ts`

- **What**:

  RED — Add tests for a new `writeOrderDocumentAuditEntry` helper:

  ```
  describe('writeOrderDocumentAuditEntry', () => {
    it('writes one audit log entry for the given order with category, source, documentId')
    it('logs and swallows errors when writeAuditLog throws')
    it('logs an info line on success including orderId and documentId')
  })
  ```

  GREEN — Implement:

  ```ts
  export interface WriteOrderDocumentAuditEntryParams {
    orderId: string;
    documentId: string;
    category: string;
    documentSource: 'fax' | 'e-order' | 'upload';
    actorId: string;
    auditSource: AuditSource;
  }

  export async function writeOrderDocumentAuditEntry(
    params: WriteOrderDocumentAuditEntryParams
  ): Promise<void> {
    const { orderId, documentId, category, documentSource, actorId, auditSource } = params;
    try {
      await writeAuditLog({
        entityType: 'order',
        entityId: orderId,
        action: 'update',
        actorId,
        source: auditSource,
        changes: [
          { field: 'documents', from: null, to: { _id: documentId, category, source: documentSource } },
        ],
      });
      logger.info('Document audit entry written', { orderId, documentId, category });
    } catch (error) {
      logger.error('Failed to write document audit entry', { orderId, documentId, error });
    }
  }
  ```

### Step 3 — Add orchestrator `propagateDocumentNotificationToOrder`

- **Files**:
  - `apps/web/services/notifications/documentNotification.ts`
  - `apps/web/services/notifications/documentNotification.test.ts`

- **What**:

  RED — Add tests:

  ```
  describe('propagateDocumentNotificationToOrder', () => {
    it('returns early and logs info when document is ineligible (Order, unlinked)')
    it('writes notification, audit entry, and SignalR for an eligible (order, document) pair, in that order')
    it('does not write a notification row when order.assignedTo is missing')
    it('continues to audit and SignalR steps if notification step throws (handled internally)')
    it('continues to SignalR step if audit step throws')
    it('skips SignalR broadcast when getSignalR resolver returns null')
    it('uses category="Document" fallback when document.category is undefined')
  })
  ```

  GREEN — Implement the orchestrator (see "Orchestrator Shape" above). Reuse `writeOrderDocumentNotification`, `broadcastDocumentAddedEvent`, and the new `writeOrderDocumentAuditEntry`. Each step is invoked with `await` but never throws, so step independence is enforced at the step implementation level — the orchestrator itself does not need try/catches around the awaits.

  REFACTOR — Ensure the step functions each log `info` on success and `error` on failure with `orderId`, `documentId`, and the relevant error.

### Step 4 — Convert `notifyDocumentCreated` to caller-side fan-out

- **Files**:
  - `apps/web/services/notifications/documentNotification.ts`
  - `apps/web/services/notifications/documentNotification.test.ts`

- **What**:

  RED — Add a test that verifies `notifyDocumentCreated`:
  - Calls `propagateDocumentNotificationToOrder` once per patient order
  - Skips orders with no `assignedTo`
  - Does NOT itself check the eligibility predicate (delegated to orchestrator)
  - Returns early when `ORDER_DOCUMENT_NOTIFICATIONS` flag is off
  - Returns early when `patientId` is missing
  - Returns early when patient has no orders

  Also add behavioral tests at this layer for the eligibility outcome (Order/unlinked produces 0 calls, Order/linked produces 1 call, non-Order produces N calls).

  GREEN — Rewrite `notifyDocumentCreated`:

  ```ts
  export async function notifyDocumentCreated(
    params: NotifyDocumentCreatedParams
  ): Promise<void> {
    try {
      const { documentId, patientId, category, source, createdBy } = params;
      if (!(await FeatureFlagService.getFeatureFlag(FeatureFlag.ORDER_DOCUMENT_NOTIFICATIONS))) return;
      if (!patientId) return;

      const orders = await getOrdersByPatient(patientId, { expandRefs: false });
      if (orders.length === 0) return;

      // The orchestrator needs the document's linkedOrderIds to evaluate eligibility.
      const document = await getDocumentById(documentId);
      if (!document) return;

      const getSignalR = createSignalRResolver(documentId);

      for (const order of orders) {
        await propagateDocumentNotificationToOrder({
          order,
          document,
          source,
          actorId: createdBy,
          auditSource: 'user',  // see Step 5 — auditSource becomes a param
          getSignalR,
        });
      }
    } catch (error) {
      logger.error('Document notification: Unexpected error in notifyDocumentCreated', { documentId: params.documentId, error });
    }
  }
  ```

  Note: `notifyDocumentCreated` historically did not write audit entries — `recordDocumentAddedToOrders` was a separate function. After this refactor, the orchestrator writes both. **Callers that previously called `notifyDocumentCreated` AND `recordDocumentAddedToOrders` must drop the second call** — see Step 6.

### Step 5 — Update `NotifyDocumentCreatedParams` to carry audit metadata

- **Files**:
  - `apps/web/services/notifications/documentNotification.ts`

- **What**: Add `actorId` and `auditSource` fields to `NotifyDocumentCreatedParams`. Plumb them through to the orchestrator. The audit-source value (`'user' | 'webhook'`) was previously a parameter to `recordDocumentAddedToOrders`; it migrates to `notifyDocumentCreated` since that is the surviving entry point.

  No new tests at this layer — Step 4 already covers the wiring.

### Step 6 — Drop `recordDocumentAddedToOrders` calls from upload sites

- **Files**:
  - `apps/web/app/api/documents/split-pdf/route.ts`
  - `apps/web/app/api/documents/split-pdf/route.test.ts` (if it asserts on `recordDocumentAddedToOrders`)
  - `apps/web/services/jobs/definitions/jotFormSubmission/weinfuseUploader.ts`
  - `apps/web/services/jobs/definitions/jotFormSubmission/__tests__/weinfuseUploader.test.ts` (likewise)

- **What**:

  RED — Update the existing route test to assert that `recordDocumentAddedToOrders` is **not** called, and that `notifyDocumentCreated` is called with the new `actorId` and `auditSource` fields.

  GREEN — In both call sites, remove the `void recordDocumentAddedToOrders({...})` block and merge its `actorId` / `auditSource` values into the `notifyDocumentCreated` call:

  ```ts
  void notifyDocumentCreated({
    documentId: String(createdDoc._id),
    patientId: patient?._id?.toString(),
    category: uploadData.category,
    source: 'upload',
    createdBy: session.user?.id || session.user?.email || 'system',
    actorId: session.user?.id || '',
    auditSource: 'user',
  });
  ```

  At this point `recordDocumentAddedToOrders` has no callers and is deleted from `documentAudit.ts` (per `feedback_delete_unused_code`). `writeOrderDocumentAuditEntry` remains as the per-order primitive the orchestrator uses.

### Step 7 — Convert `notifyOrderOfRecentDocuments` to orchestrator fan-out

- **Files**:
  - `apps/web/services/notifications/documentNotification.ts`
  - `apps/web/services/notifications/documentNotification.test.ts`

- **What**:

  RED — Update the test suite for `notifyOrderOfRecentDocuments`:

  ```
  it('calls propagateDocumentNotificationToOrder once per recent patient document')
  it('skips Order-category documents whose linkedOrderIds does not include the new order id (eligibility delegated to orchestrator)')
  it('includes Order-category documents whose linkedOrderIds includes the new order id')
  it('writes notification + audit + SignalR for each eligible document (via orchestrator)')
  it('returns early when ORDER_DOCUMENT_NOTIFICATIONS flag is off')
  it('returns early when assignedTo or patientId is missing')
  it('returns early when the patient has no recent documents')
  ```

  GREEN — Rewrite the function body so the inner loop hands off to the orchestrator:

  ```ts
  export async function notifyOrderOfRecentDocuments(
    params: NotifyOrderOfRecentDocumentsParams
  ): Promise<void> {
    const { orderId, assignedTo, patientId } = params;
    try {
      if (!(await FeatureFlagService.getFeatureFlag(FeatureFlag.ORDER_DOCUMENT_NOTIFICATIONS))) return;
      if (!assignedTo || !patientId) return;

      const since = new Date(Date.now() - BACKFILL_LOOKBACK_DAYS * MILLIS_PER_DAY);
      const documents = await getDocumentsByPatientId(patientId, { since });
      if (documents.length === 0) return;

      const order = { _id: orderId, assignedTo };
      const getSignalR = createSignalRResolver(orderId);

      for (const document of documents) {
        await propagateDocumentNotificationToOrder({
          order,
          document,
          source: 'upload',
          actorId: 'system',
          auditSource: 'system',  // see notes below
          getSignalR,
        });
      }
    } catch (error) {
      logger.error(
        'Document notification: Unexpected error in notifyOrderOfRecentDocuments',
        { orderId, error }
      );
    }
  }
  ```

  Notes:
  - The `auditSource` for this trigger is `'system'` because the order-creation flow is server-initiated, not a user upload action. If the existing `AuditSource` union does not include `'system'`, extend it or pick the closest existing value (`'webhook'` if order creation is webhook-driven; check the order-creation entry point).
  - The previous coalesced single-SignalR-broadcast logic is removed entirely. Each orchestrator call's SignalR step emits one broadcast. Per `feedback_signalr_cheap_mongo_expensive` this is the desired tradeoff.
  - The "representative document" hack used at line 277 is no longer needed and is deleted.

### Step 8 — Wire the link-time trigger

- **Files**:
  - `apps/web/app/api/documents/[documentId]/link-orders/route.ts`
  - `apps/web/app/api/documents/[documentId]/link-orders/__tests__/route.test.ts`

- **What**:

  RED — Add tests:

  ```
  describe('PUT /api/documents/[documentId]/link-orders — propagation trigger', () => {
    it('fires propagateDocumentNotificationToOrder for each newly-added orderId')
    it('does not fire for orderIds that were already in linkedOrderIds')
    it('does not fire when the request is a re-save with no new orderIds')
    it('does not fire for orderIds that were removed')
    it('uses the document AFTER the linkedOrderIds update so eligibility resolves true')
    it('continues silently when an order lookup returns null')
    it('does not block the 200 response on propagation failures (fire-and-forget)')
  })
  ```

  GREEN — Modify the handler around line 74:

  ```ts
  const previousLinkedOrderIds = doc.linkedOrderIds ?? [];
  const updated = await linkDocumentToOrders(documentId, sanitizedIds);
  if (!updated) {
    return NextResponse.json({ error: 'Document not found' }, { status: 404 });
  }

  const newlyAddedOrderIds = sanitizedIds.filter(
    (id) => !previousLinkedOrderIds.includes(id)
  );

  if (newlyAddedOrderIds.length > 0) {
    // Fire-and-forget — does not block the response
    void firePropagationForNewlyLinkedOrders({
      document: updated,
      orderIds: newlyAddedOrderIds,
      actorId: session.user?.id || '',
      auditSource: 'user',
    });
  }
  ```

  The `firePropagationForNewlyLinkedOrders` helper lives in `documentNotification.ts` next to the orchestrator. It:
  1. Builds a `SignalRResolver` once for the batch.
  2. For each `orderId`, calls `getOrderById(orderId, 'new')`, falling back to `'maintenance'` (mirroring MLID-2085's lookup pattern). If both return null, log a warning and continue.
  3. Calls `propagateDocumentNotificationToOrder({ order, document: updated, source: 'upload', actorId, auditSource, getSignalR })`.

  Source attribution: the link-time trigger uses `'upload'` because the original upload pathway used that source. The audit entry's `source` field will read `'upload'` consistently.

  REFACTOR — Confirm logging at the helper boundary so operations can trace which orders received the trigger after a link operation.

### Step 9 — Type-check, lint, run focused tests

- **Commands** (from `apps/web/`):
  ```
  npm run types:check
  npm run lint:fix
  npm run test -- documentNotification documentAudit link-orders documentEligibility
  ```
- Coverage check on modified files (80% threshold).

### Step 10 — Manual verification on local dev

1. Start `npm run dev`.
2. Upload a document of category "Order" via Intake Analyser for a patient who has at least one active order.
3. Open the order's Documents tab: confirm the document does NOT appear, the counter does NOT increase.
4. Open the order's Order History tab: confirm no new entry was logged.
5. Navigate to `/patient/[id]/document-records`, find the Order document, click "Update Order Linking", select that order, save.
6. Confirm the order's Documents tab now shows the document, the counter shows the increment, and the Order History tab has a new "New Document received (Document type = Order)" entry.
7. Re-save the same link selection: confirm no duplicate entries.
8. Add a second order to the link: confirm only the new order gets a new entry, the first order does not get a duplicate.
9. Upload a non-Order document (e.g. Clinical): confirm it propagates immediately to all of the patient's orders (regression check).

---

## Test Cases (consolidated)

### Eligibility predicate
- non-Order → eligible regardless of linkage
- Order + linkedOrderIds includes target → eligible
- Order + linkedOrderIds empty → ineligible
- Order + linkedOrderIds present but excludes target → ineligible
- Order + linkedOrderIds undefined → ineligible
- ObjectId vs string comparison handled via `String()`

### Per-order steps (each in isolation)
- `writeOrderDocumentNotification`: writes one notification, logs on success, swallows error on failure
- `writeOrderDocumentAuditEntry`: writes one audit row, logs on success, swallows error on failure
- `broadcastDocumentAddedEvent`: emits one SignalR event, logs on success, swallows error on failure

### Orchestrator
- Ineligible → no steps run, info log emitted
- Eligible → all three steps run in order (notification, audit, SignalR)
- assignedTo missing → notification skipped, audit + SignalR still run
- Each step independently survives the others' failures
- SignalR resolver returns null → broadcast skipped, other steps complete

### `notifyDocumentCreated` (upload entry point)
- Feature flag off → no fan-out
- patientId missing → no fan-out
- 0 patient orders → no fan-out
- N orders → orchestrator called N times
- Eligibility branches respected at the orchestrator layer (Order/unlinked → 0 effective; Order/linked-to-one → 1 effective; non-Order → N effective)

### `notifyOrderOfRecentDocuments` (order-creation backfill — trigger 2)
- Non-Order documents → orchestrator runs all three steps per doc (M broadcasts, M notifications, M audit entries)
- Order-category linked-to-this-order → orchestrator runs all three steps
- Order-category not linked → orchestrator returns early (no notification, no audit, no broadcast)
- Empty patient-document set → early return, no orchestrator call
- Feature flag off → early return
- Missing `assignedTo` or `patientId` → early return

### Link endpoint
- Re-save same array → 0 newly-added → no propagation
- Removal only → 0 newly-added → no propagation
- Add A to empty → 1 newly-added → 1 orchestrator call
- Add B to [A] → 1 newly-added → 1 orchestrator call for B only
- Add [A, B] to empty → 2 newly-added → 2 orchestrator calls
- Order lookup returns null for one orderId → that one is logged and skipped, the others proceed
- Propagation throws → response is still 200 (fire-and-forget)
- After link, document's `linkedOrderIds` includes the new orderIds → orchestrator's eligibility check passes (consistency)

---

## Out of Scope

- **"Document unlinked" audit entries.** Removing an order from `linkedOrderIds` does not produce any new history record. The original spec does not request this.
- **Backfilling production data.** Existing stray notification rows or audit entries written by the pre-fix code remain in the database. No migration. If operations needs cleanup, that is a separate ticket.
- **Order reassignment trigger.** Mentioned in earlier discussion as a hypothetical. MLID-2085 already handles the reassignment problem at the read layer; re-firing notifications on reassignment is not added here.

---

## Risks

- **Behavior change on the upload path.** Today an upload of a non-Order document writes notifications, SignalR, and audit entries to every patient order. After this refactor that is still true (the orchestrator runs all three steps for eligible docs). Eligible == non-Order, or Order with the target order in `linkedOrderIds`. There should be no regression for non-Order uploads.
- **Audit field shape change.** `recordDocumentAddedToOrders` is deleted and replaced by `writeOrderDocumentAuditEntry` invoked from the orchestrator. The actual audit document shape (`field: 'documents'`, `to: { _id, category, source }`) is preserved exactly so the existing `auditLog.config.ts` renderer at lines 34-41 keeps working without change.
- **Race between `linkDocumentToOrders` and the orchestrator's eligibility check.** The orchestrator is called with the `updated` document returned by `linkDocumentToOrders` (which uses `{ new: true }`), so the `linkedOrderIds` it sees already includes the just-linked orders. No race.
- **SignalR availability under failure.** If SignalR is misconfigured at the time of an upload or link, notifications and audit entries still land; the SignalR broadcast is logged and skipped. This matches today's behavior in `documentNotification.ts:140-149`.

---

## Files Touched (summary)

| File | Change |
|---|---|
| `apps/web/services/notifications/documentEligibility.ts` | NEW — eligibility predicate |
| `apps/web/services/notifications/documentEligibility.test.ts` | NEW — predicate tests |
| `apps/web/services/notifications/documentNotification.ts` | Refactor — add orchestrator; convert `notifyDocumentCreated` and `notifyOrderOfRecentDocuments` to orchestrator-based fan-outs |
| `apps/web/services/notifications/documentNotification.test.ts` | Update tests |
| `apps/web/services/audit/documentAudit.ts` | Add `writeOrderDocumentAuditEntry`, remove `recordDocumentAddedToOrders` |
| `apps/web/services/audit/documentAudit.test.ts` | Update tests |
| `apps/web/app/api/documents/split-pdf/route.ts` | Drop `recordDocumentAddedToOrders` call, plumb `actorId` + `auditSource` |
| `apps/web/app/api/documents/split-pdf/route.test.ts` | Update assertions |
| `apps/web/services/jobs/definitions/jotFormSubmission/weinfuseUploader.ts` | Same: drop audit call, plumb fields |
| `apps/web/services/jobs/definitions/jotFormSubmission/__tests__/weinfuseUploader.test.ts` | Update assertions |
| `apps/web/app/api/documents/[documentId]/link-orders/route.ts` | Add link-time trigger (delta + propagation helper call) |
| `apps/web/app/api/documents/[documentId]/link-orders/__tests__/route.test.ts` | Add propagation tests |
