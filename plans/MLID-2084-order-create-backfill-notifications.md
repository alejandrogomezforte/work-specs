# MLID-2084 — Order Create: Backfill Document Notifications

## Task Reference

- **Jira**: [MLID-2084](https://localinfusion.atlassian.net/browse/MLID-2084)
- **Story Points**: 5
- **Branch**: `feature/MLID-2084-order-create-backfill-notifications`
- **Base Branch**: `develop` (epic MLID-2011 is already merged to develop — do NOT base off any old epic branch)
- **Epic Plan**: `docs/agomez/plans/MLID-2011-plan-progress.md`
- **Status**: Done

---

## Summary

QA found (2026-05-11) that when a user uploads documents for a patient and then creates a new order, the order badge shows no unread count even though the Documents tab already has content. The existing `notifyDocumentCreated` helper fires at upload time only, so documents uploaded before the order exists have no corresponding notifications.

This task adds a complementary order-creation-time trigger, `notifyOrderOfRecentDocuments`, that backfills notifications for all documents the patient had in the last 30 days. The new helper mirrors the existing notification shape and SignalR event exactly — so the badge query, badge renderer, and real-time listener require no UI changes. The upload-time trigger is left intact and unchanged in behavior.

The implementation also extracts a private primitive, `dispatchOrderDocumentNotification`, from the existing loop in `notifyDocumentCreated`, so both triggers share the same per-(order, document) body rather than duplicating the notification shape and error handling independently.

---

## Codebase Analysis

### Order creation entry point

`apps/web/app/api/orders/new/route.ts` POST handler (lines 90–157). After `createOrder(...)` returns (line 130), `autoAssignOrders` is called (line 140) and its result overwrites `newOrders` if non-null. The final `newOrders` variable, after the auto-assign block, carries the correct `assignedTo` for each order. The backfill trigger must be called after this block.

`createOrder` may return two orders when an `orderProtocol` is used — iterating `newOrders` handles this naturally.

### FullOrder patient field shape

`FullOrder<'new'>` has `patient: PatientDetails` when hydrated. In the route, `createOrder` resolves the patient reference via the aggregation pipeline, so `order.patient` should be a `PatientDetails` object with `_id: string`. However, if no patient was matched (e.g. the order body contained no `weInfusePatientId`), `order.patient` may be an unresolved string `_id` or absent. The implementation must guard against this before extracting `patient._id`.

Guard pattern (inside the per-order loop):
```typescript
if (
  !order.patient ||
  typeof order.patient !== 'object' ||
  !('_id' in order.patient)
) {
  continue;
}
const patientId = String(order.patient._id);
```

### Existing `notifyDocumentCreated`

`apps/web/services/notifications/documentNotification.ts` — the per-order body is currently inlined inside a `for` loop. The source label, notification fields, SignalR payload, and error handling logic will be extracted into a private primitive function in the same file, so both `notifyDocumentCreated` (existing) and `notifyOrderOfRecentDocuments` (new) call the primitive.

The existing `DocumentNotificationSource` type already includes `'upload' | 'e-order' | 'fax'`. No enum change is needed. The new backfill helper always passes `'upload'` as the source — no source field is stored on `Document`, so `'upload'` is the fallback label used for all documents created outside of a known e-order or fax flow.

### getDocumentsByPatientId in document service

`apps/web/services/mongodb/document.ts` line 110 — current signature:
```typescript
export async function getDocumentsByPatientId(
  patientId: string,
  options?: { limit?: number }
): Promise<IDocument[]>
```
Sort is `createdAt: -1`. The query does not filter by date. This function needs a `since?: Date` option so the backfill helper can restrict to the 30-day window without pulling the full document history for long-standing patients.

### getOrdersByPatient in ordersTracker

`apps/web/services/mongodb/ordersTracker.ts` — `getOrdersByPatient(patientId, { expandRefs: false })` returns `OrderRef[]` where `OrderRef = { _id: string; assignedTo?: string }`. This overload already exists and is used by `notifyDocumentCreated`. The new helper calls the same function with the same overload, which is consistent.

### Feature flag

`FeatureFlag.ORDER_DOCUMENT_NOTIFICATIONS` — already defined and gating the existing upload-time trigger. The new backfill helper checks the same flag. No new flag.

### SignalR broadcast payload

The existing `notifyDocumentCreated` broadcasts `{ documentId, orderId, source, category }` — no `patientId` (PHI minimization). The new primitive must match this shape exactly so the badge listener picks up the events.

### Test files to extend

- `apps/web/services/notifications/documentNotification.test.ts` — add describe blocks for the private primitive extraction (regression guard on existing behavior) and for `notifyOrderOfRecentDocuments`
- `apps/web/services/mongodb/document.test.ts` — add cases for `getDocumentsByPatientId` with `since` option
- `apps/web/app/api/orders/new/route.test.ts` — add cases for the backfill trigger being called after `autoAssignOrders`

### Known limitation: no deduplication

`createNotification` does not enforce uniqueness across `(targetUserId, entityId)`. If a user creates the same order twice (or runs a re-import), duplicate notifications will land. This is acceptable in v1 — the QA scenario (create-once) does not trigger duplicates. The limitation is documented in Resolved Questions.

---

## Implementation Steps

### Step 1 — RED: failing tests for `getDocumentsByPatientId` with `since` option

- **Files**: `apps/web/services/mongodb/document.test.ts` (modify — add cases)

- **What**: Extend the existing test file with a new describe block that covers the `since` option before any implementation exists. The test should import the function and the `Document` Mongoose model mock. Run `npm run test -- document.test` from `apps/web/` — the new cases fail because `since` is not yet handled.

  Test cases to add inside a new `describe('getDocumentsByPatientId — since option')` block:

  ```
  when since is not provided
    ✓ queries without a date filter (existing behavior, non-regression)
    ✓ returns all documents sorted by createdAt descending

  when since is a Date
    ✓ adds a createdAt >= since filter to the query
    ✓ does not affect the sort (still createdAt descending)
    ✓ returns only documents created on or after the since date

  when since is provided alongside limit
    ✓ applies both since filter and limit
  ```

  Mock boundary: `@/models/Document` — mock `Document.find()` to return a chainable query builder mock with `.sort().where().gte().limit().lean().exec()` methods.

---

### Step 2 — GREEN: add `since?: Date` to `getDocumentsByPatientId`

- **Files**: `apps/web/services/mongodb/document.ts` (modify)

- **What**: Extend the `options` parameter type and add a conditional `.where('createdAt').gte(since)` call to the existing query builder chain. Only the `getDocumentsByPatientId` function changes — no other functions in the file are touched.

  Updated signature:
  ```typescript
  export async function getDocumentsByPatientId(
    patientId: string,
    options?: { limit?: number; since?: Date }
  ): Promise<IDocument[]>
  ```

  Updated query chain (inside the existing `try` block):
  ```typescript
  let query = DocumentModel.find({ patientId }).sort({ createdAt: -1 });

  if (options?.since) {
    query = query.where('createdAt').gte(options.since) as typeof query;
  }

  if (options?.limit) {
    query = query.limit(options.limit);
  }

  return await query.lean().exec();
  ```

  Run `npm run test -- document.test` — all Step 1 cases go green, existing cases stay green.

---

### Step 3 — REFACTOR + RED: extract private primitive, add regression tests

- **Files**: `apps/web/services/notifications/documentNotification.ts` (modify), `apps/web/services/notifications/documentNotification.test.ts` (modify — add cases)

- **What**: Extract the per-(order, document) work from `notifyDocumentCreated`'s inner loop into a private async function `dispatchOrderDocumentNotification` in the same file. The existing public function delegates to it. All existing tests must stay green — if any break, fix the implementation, not the tests.

  **New private function signature:**
  ```typescript
  interface DispatchOrderDocumentNotificationParams {
    orderId: string;
    assignedTo: string;
    documentId: string;
    category: string;
    source: DocumentNotificationSource;
    signalr: Awaited<ReturnType<typeof getSignalRService>> | null;
  }

  async function dispatchOrderDocumentNotification(
    params: DispatchOrderDocumentNotificationParams
  ): Promise<void>
  ```

  The function body contains:
  1. `createNotification(...)` in a `try/catch` — log error on failure, continue
  2. SignalR `broadcastToAll(...)` in a `try/catch` — log error on failure, continue (only if `signalr` is non-null)

  **Refactored `notifyDocumentCreated`**: the inner loop body is replaced with a call to `dispatchOrderDocumentNotification`. SignalR init logic (lazy, once-per-call) stays in `notifyDocumentCreated` — the primitive receives the already-initialized `signalr` handle. This keeps `dispatchOrderDocumentNotification` synchronous with respect to SignalR init and testable in isolation by passing a mock handle directly.

  **New test cases** (add to `documentNotification.test.ts`, inside a new `describe('dispatchOrderDocumentNotification — via notifyDocumentCreated delegation')`):

  ```
  when notifyDocumentCreated dispatches to an assigned order
    ✓ calls dispatchOrderDocumentNotification (observed indirectly via createNotification mock)
    ✓ notification shape is unchanged after refactor — non-regression
    ✓ broadcastToAll payload is unchanged after refactor — non-regression
  ```

  These tests are behavior tests against the public function — they don't import the private primitive directly. They exist to document the refactor's intent and catch regressions. All existing tests in the file continue to test the same observable behaviors.

  Run `npm run test -- documentNotification.test` — all existing and new cases green.

---

### Step 4 — RED: failing tests for `notifyOrderOfRecentDocuments`

- **Files**: `apps/web/services/notifications/documentNotification.test.ts` (modify — add cases)

- **What**: Add a new top-level `describe('notifyOrderOfRecentDocuments')` block. The function does not exist yet, so importing it will cause the test file to fail at compile time. Add the import alongside `notifyDocumentCreated` and watch the suite fail.

  **Function signature to design toward (not implemented yet):**
  ```typescript
  export interface NotifyOrderOfRecentDocumentsParams {
    orderId: string;
    assignedTo: string | undefined;
    patientId: string | undefined;
  }

  export async function notifyOrderOfRecentDocuments(
    params: NotifyOrderOfRecentDocumentsParams
  ): Promise<void>
  ```

  **Test cases:**

  ```
  describe('notifyOrderOfRecentDocuments')
    when feature flag ORDER_DOCUMENT_NOTIFICATIONS is disabled
      ✓ returns without querying documents
      ✓ does not call createNotification
      ✓ does not call broadcastToAll

    when assignedTo is undefined
      ✓ returns without querying documents
      ✓ does not call createNotification

    when assignedTo is an empty string
      ✓ returns without querying documents

    when patientId is undefined
      ✓ returns without querying documents

    when patientId is an empty string
      ✓ returns without querying documents

    when no documents exist in the 30-day window
      ✓ does not call createNotification
      ✓ does not call broadcastToAll

    when one document exists in the 30-day window
      ✓ calls dispatchOrderDocumentNotification once (observed via createNotification mock)
      ✓ createNotification receives type 'user', targetUserId = assignedTo
      ✓ entities include { entityType: 'order', entityId: orderId } and { entityType: 'document', entityId: documentId }
      ✓ title includes the document category
      ✓ message includes 'Upload' (source = 'upload')
      ✓ broadcastToAll is called with event 'documentAdded' and payload { documentId, orderId, source: 'upload', category }
      ✓ broadcastToAll payload does not include patientId

    when three documents exist in the 30-day window
      ✓ calls createNotification three times (once per document)
      ✓ calls broadcastToAll three times (once per document)
      ✓ each notification targets the same assignedTo

    when getDocumentsByPatientId is called
      ✓ passes since = now minus BACKFILL_LOOKBACK_DAYS days (within 1 second tolerance)

    when dispatchOrderDocumentNotification throws for one document
      ✓ processing continues for remaining documents
      ✓ function resolves (never re-throws)

    overall error safety
      ✓ function resolves even if the top-level block throws unexpectedly
  ```

  **Mock boundaries**: `@/services/featureFlags/featureFlagService`, `@/services/mongodb/document` (mock `getDocumentsByPatientId`), `@/services/mongodb/notification` (mock `createNotification`), `@/services/signalr` (mock `getSignalRService`), `@/utils/logger`.

  Run `npm run test -- documentNotification.test` — new cases fail (function not defined), existing cases stay green.

---

### Step 5 — GREEN: implement `notifyOrderOfRecentDocuments`

- **Files**: `apps/web/services/notifications/documentNotification.ts` (modify)

- **What**: Implement the new exported function. The `BACKFILL_LOOKBACK_DAYS` constant is defined at the top of the file (not exported, not an env var):

  ```typescript
  const BACKFILL_LOOKBACK_DAYS = 30;
  ```

  **Logic sequence** (entire function body in a top-level `try/catch` that logs and swallows):
  1. Check `ORDER_DOCUMENT_NOTIFICATIONS` flag — return if disabled.
  2. Guard: return if `assignedTo` is falsy.
  3. Guard: return if `patientId` is falsy.
  4. Compute `since = new Date(Date.now() - BACKFILL_LOOKBACK_DAYS * 24 * 60 * 60 * 1000)`.
  5. Call `await getDocumentsByPatientId(patientId, { since })` — return if empty.
  6. Init SignalR lazily (same once-per-call pattern as `notifyDocumentCreated`): `let signalr = null; let signalrInitAttempted = false;` — inside the loop, after the first document, attempt init once.
  7. For each document:
     - Call `dispatchOrderDocumentNotification({ orderId, assignedTo, documentId: String(document._id), category: document.category, source: 'upload', signalr })` in a `try/catch` — log error, continue to next document.

  Run `npm run test -- documentNotification.test` from `apps/web/` — all Step 4 cases go green.

---

### Step 6 — RED: failing tests for the order creation trigger in `route.test.ts`

- **Files**: `apps/web/app/api/orders/new/route.test.ts` (modify — add cases)

- **What**: Add `jest.mock('@/services/notifications/documentNotification')` to the existing mock block at the top of the test file, and import `notifyOrderOfRecentDocuments` from the module. Add a new `describe('POST /api/orders/new — document backfill trigger')` block inside the existing `describe('Orders API Route')`.

  **Test cases:**

  ```
  describe('POST /api/orders/new — document backfill trigger')
    when createOrder returns one order with patient and assignedTo
      ✓ calls notifyOrderOfRecentDocuments once, with the correct orderId, assignedTo, and patientId
      ✓ orderId is String(order._id)
      ✓ assignedTo comes from the post-autoAssign order (not pre-assign)
      ✓ patientId is String(order.patient._id)

    when createOrder returns two orders (protocol pair) each with patient and assignedTo
      ✓ calls notifyOrderOfRecentDocuments twice (once per order)

    when order.patient is a string (not yet expanded)
      ✓ does not call notifyOrderOfRecentDocuments for that order (guard skips it)

    when order.patient is undefined
      ✓ does not call notifyOrderOfRecentDocuments for that order

    when order.assignedTo is absent
      ✓ calls notifyOrderOfRecentDocuments with assignedTo: undefined (helper handles the skip)

    when createOrder returns null
      ✓ notifyOrderOfRecentDocuments is not called

    when notifyOrderOfRecentDocuments throws (it should not, but guard test)
      ✓ response is still 200 (trigger failure does not fail the order create)
  ```

  Mock the notification module: `jest.mock('@/services/notifications/documentNotification', () => ({ notifyOrderOfRecentDocuments: jest.fn().mockResolvedValue(undefined) }))`.

  For the test cases that need `autoAssignOrders` to set `assignedTo`, mock `mockAutoAssignOrders` to return an order array with `assignedTo` present and assert the helper received that post-assign value (not the pre-assign one).

  Run `npm run test -- route.test` from `apps/web/app/api/orders/new/` — new cases fail (helper not yet imported/called in route), existing cases stay green.

---

### Step 7 — GREEN: wire `notifyOrderOfRecentDocuments` into the POST handler

- **Files**: `apps/web/app/api/orders/new/route.ts` (modify)

- **What**: Add the import and the fire-and-forget call immediately after the `autoAssignOrders` block (after line 144, before `postOrderCreatedSlackNotification`). Use `void` and never `await` so the route response is not blocked.

  **Import to add** (with the other service imports at the top):
  ```typescript
  import { notifyOrderOfRecentDocuments } from '@/services/notifications/documentNotification';
  ```

  **Code to add after the `autoAssignOrders` block** (inside the `if (newOrders)` block, after the reassignment):

  ```typescript
  for (const order of newOrders) {
    if (
      !order.patient ||
      typeof order.patient !== 'object' ||
      !('_id' in order.patient)
    ) {
      continue;
    }
    void notifyOrderOfRecentDocuments({
      orderId: String(order._id),
      assignedTo: order.assignedTo,
      patientId: String(order.patient._id),
    });
  }
  ```

  The `void` expression means the route handler does not await the trigger. The helper's top-level `try/catch` ensures it never propagates errors to the caller.

  Run `npm run test -- route.test` from `apps/web/app/api/orders/new/` — all Step 6 cases go green, all existing cases stay green.

  Run the full notification test suite to verify no regressions:
  ```
  npm run test -- documentNotification.test
  npm run test -- document.test
  npm run test -- route.test
  npm run types:check
  npm run lint:fix
  ```
  (All commands run from `apps/web/`.)

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/services/mongodb/document.ts` | Modify | Add `since?: Date` option to `getDocumentsByPatientId`; filters by `createdAt >= since` when provided |
| `apps/web/services/mongodb/document.test.ts` | Modify | Add test cases for `since` option in `getDocumentsByPatientId` |
| `apps/web/services/notifications/documentNotification.ts` | Modify | Extract private `dispatchOrderDocumentNotification` primitive from inner loop; add `BACKFILL_LOOKBACK_DAYS` constant; add exported `notifyOrderOfRecentDocuments` function and its `NotifyOrderOfRecentDocumentsParams` type |
| `apps/web/services/notifications/documentNotification.test.ts` | Modify | Add refactor regression cases and full test suite for `notifyOrderOfRecentDocuments` |
| `apps/web/app/api/orders/new/route.ts` | Modify | Import `notifyOrderOfRecentDocuments`; add per-order fire-and-forget loop after `autoAssignOrders` block inside the `if (newOrders)` guard |
| `apps/web/app/api/orders/new/route.test.ts` | Modify | Mock `documentNotification` module; add describe block for backfill trigger call behavior |

---

## Testing Strategy

### Unit tests — `document.test.ts` additions

- `getDocumentsByPatientId` with no `since`: query builder receives no date filter (non-regression)
- `getDocumentsByPatientId` with `since: new Date('2025-04-15')`: query builder receives `.where('createdAt').gte(new Date('2025-04-15'))`
- `getDocumentsByPatientId` with `since` + `limit`: both applied, in the correct chain order

### Unit tests — `documentNotification.test.ts` additions

Refactor regression (Step 3):
- `notifyDocumentCreated` with one order → `createNotification` shape unchanged after extracting primitive
- `notifyDocumentCreated` with one order → `broadcastToAll` payload unchanged after extracting primitive

`notifyOrderOfRecentDocuments` (Step 4):
- Feature flag off → no document query, no notification, no broadcast
- `assignedTo` falsy → no document query
- `patientId` falsy → no document query
- Zero docs in window → no notification, no broadcast
- One doc → `createNotification` called once with `type: 'user'`, `targetUserId: assignedTo`, `entities` referencing both `orderId` and `documentId`, `message` containing `'Upload'`
- One doc → `broadcastToAll` called with `'documentAdded'` and `{ documentId, orderId, source: 'upload', category }`, no `patientId` in payload
- Three docs → `createNotification` called 3 times, `broadcastToAll` called 3 times
- `since` cutoff: `getDocumentsByPatientId` receives `since` within 1 second of `now - 30 days`
- `dispatchOrderDocumentNotification` throws for doc 1 → doc 2 still processed, function resolves

### Unit tests — `route.test.ts` additions

- One order with `patient: { _id: 'patient-1' }` and `assignedTo: 'user-1'` → `notifyOrderOfRecentDocuments` called once with `{ orderId: 'order1', assignedTo: 'user-1', patientId: 'patient-1' }`
- Two orders (protocol pair) → `notifyOrderOfRecentDocuments` called twice
- `autoAssignOrders` sets `assignedTo` → helper called with the post-assign value
- `order.patient` is a plain string → helper not called for that order
- `order.patient` is undefined → helper not called for that order
- `createOrder` returns null → helper not called
- Helper rejects (mock) → response is still `200`

### Manual verification (QA repro scenario)

1. Upload two documents for a patient who has no active order yet (ensure the patient has a `we_infuse_id_IV` match available).
2. Create a new order in Order Tracker for that patient, assigned to a logged-in user.
3. Within a few seconds, the order badge should show `2`.
4. In MongoDB `notifications` collection, confirm two documents exist with:
   - `type: 'user'`
   - `targetUserId` equal to the assigned user's `_id`
   - `entities` arrays each containing one `entityType: 'order'` ref and one `entityType: 'document'` ref
   - `message: 'A new document was added via Upload.'`
5. Confirm a `documentAdded` SignalR event was broadcast for each document (two events total).
6. Upload a third document after the order exists — confirm the badge increments to `3` via the upload-time trigger (existing behavior, non-regression).
7. Disable `ORDER_DOCUMENT_NOTIFICATIONS` via admin panel, create another order — confirm no new notification documents are created.

---

## Security Considerations

- **Feature flag gating**: `FeatureFlag.ORDER_DOCUMENT_NOTIFICATIONS` is checked at the top of `notifyOrderOfRecentDocuments` before any DB query. The same flag already gates the upload-time trigger — both paths are controlled by one toggle.
- **PHI handling**: The `broadcastToAll` payload contains only `{ documentId, orderId, source, category }` — no `patientId`, no patient name, no clinical data. This matches the existing trigger's PHI minimization contract. The `since` date window query uses only IDs and dates — no PHI fields are logged.
- **Fire-and-forget safety**: `void notifyOrderOfRecentDocuments(...)` is used at the call site. The helper's top-level `try/catch` ensures it can never propagate an error that would fail the order creation response.
- **Server-side only**: `notifyOrderOfRecentDocuments` is a service-layer function. It is never called from client components or exposed via a public API endpoint.
- **Input guards**: `assignedTo` and `patientId` are both validated as truthy strings before any DB query is executed. The `orderId` is derived from `String(order._id)` — a safe coercion of a Mongoose ObjectId.
- **Authorization**: The trigger runs inside the already-authenticated POST handler. No additional auth check is needed inside the helper.
- **No new indexes**: The `getDocumentsByPatientId` query adds a `createdAt` range filter. The `Document` model's `patientId` index covers the equality filter; `createdAt` is a Mongoose-managed timestamp field. The 30-day window combined with per-page order counts (25–100) makes this a bounded, index-friendly query.

---

## Resolved Questions

1. **Lookback window duration**: 30 days, defined as `const BACKFILL_LOOKBACK_DAYS = 30;` inline in `documentNotification.ts`. Not an env var — env-var overrides cannot be wired into Azure without PE involvement. The constant can be promoted later.

2. **Source label for backfilled documents**: `'upload'`. The `Document` model stores no `source` field, so the fallback is `'upload'`, producing `"A new document was added via Upload."` in the notification body. If QA determines this label is confusing for documents that actually came from e-orders or fax (uploaded before the order existed), escalate to UX for a new variant.

3. **Deduplication**: None in v1. The QA scenario (create an order once) does not produce duplicates. If a user creates the same order twice or a re-import occurs, duplicate notifications will exist. If this becomes a problem post-launch, a `(targetUserId, entityType, entityId)` uniqueness check should be added to `createNotification`.

4. **Maintenance and lead order routes are out of scope**: The existing upload-time trigger (`notifyDocumentCreated`) only processes `neworders`. The backfill trigger mirrors that scope. `apps/web/app/api/orders/maintenance/route.ts` and `.../lead/route.ts` are not touched.

5. **Helper extraction (`dispatchOrderDocumentNotification`) is part of this PR**: Not a separate task. The refactor is necessary to avoid duplicating the notification body, error handling, and SignalR contract across the two triggers. The existing public API of `notifyDocumentCreated` is unchanged.

6. **`autoAssignOrders` return value handling**: When `autoAssignOrders` returns null (e.g., the order was intake-created without a site), the route keeps the original `newOrders` from `createOrder`. The per-order loop iterates whichever value `newOrders` holds at that point, so the guard `if (newOrders)` (already present) is the only condition needed before the loop.

---

## Open Questions / Follow-ups

1. **`'upload'` label accuracy**: If the backfill includes documents originally created via e-order or fax flows (uploaded before the order, so they fire the upload-time trigger with the wrong source label at upload time — a pre-existing issue, not introduced here), the `'upload'` label in the backfill notification will be consistent with what those documents already show. However, if UX decides the label should reflect the actual document origin rather than always defaulting to `'upload'` for the backfill path, a source field would need to be added to the `Document` model. Flag to UX if QA raises this.

2. **Duplicate notification dedup**: If the same patient-order-document triple appears in both the upload-time trigger (if the order happened to exist at upload time) and the backfill trigger (order created after), two notifications for the same document will exist. The 30-day window makes this scenario plausible. If badge overcounting is reported post-launch, design a `(targetUserId, entityType + entityId)` uniqueness constraint in `createNotification` or in the badge count query.
