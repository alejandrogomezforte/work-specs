# MLID-2083 — Investigate Document Entry Points + Emit SignalR Events

## Task Reference

- **Jira**: [MLID-2083](https://localinfusion.atlassian.net/browse/MLID-2083)
- **Story Points**: 5
- **Branch**: `feature/MLID-2083-document-notification-triggers`
- **Base Branch**: `epic/MLID-2011-order-document-notifications`
- **Epic Plan**: `docs/agomez/plans/MLID-2011-plan-progress.md`
- **Status**: Done

---

## Summary

Wire document-creation notification triggers into the two places in the codebase where documents are saved: the split-pdf API route (user uploads and splits) and the JotForm/WeInfuse background job uploader. After each successful document save, a `user`-type notification is created for every active `neworders` order assigned to the matched patient, and a `documentAdded` SignalR broadcast is fired. All trigger logic lives in a single dedicated helper so the notification semantics are defined once and are impossible to miss at new entry points.

---

## Codebase Analysis

### The single document creation function

`apps/web/services/mongodb/document.ts` — `createDocumentRecord()` (line 28) is the only place that calls `DocumentModel.create()` in production code. It does not receive `source` metadata (`'upload'` vs `'e-order'`), so the notification trigger cannot live inside it without threading source through every caller. The alternative — calling the helper at each call site after `createDocumentRecord()` returns — keeps source semantics at the boundary where they are naturally available and avoids touching the document service contract.

### Entry points and available data

| # | Call site | File | `patientId` (Mongo `_id`) | `weInfusePatientId` | `source` |
|---|-----------|------|--------------------------|--------------------|----|
| 1A | Full upload | `apps/web/app/api/documents/split-pdf/route.ts` ~line 338 | `patient?._id?.toString()` | `weInfusePatientId` (request body) | `'upload'` |
| 1B | Split upload | `apps/web/app/api/documents/split-pdf/route.ts` ~line 573 | `patient?._id?.toString()` | `patientId` (request variable, WeInfuse ID) | `'upload'` |
| 2 | JotForm job | `apps/web/services/jobs/definitions/jotFormSubmission/weinfuseUploader.ts` ~line 330 | `patientDbId` | `intake.patient.patientId` | `'e-order'` |

`patientId` may be `undefined` at Entry Points 1A and 1B when no patient has been matched. The helper must skip silently in that case.

### Order-to-patient link

There is no `orderId` on documents. The chain is:

```
Document.patientId (Mongo _id)
  → Patient.we_infuse_id_IV  (Number)
    → neworders.patient.we_infuse_id_IV  (embedded)
      → neworders.assignedTo  (user _id string)
```

The `neworders` collection stores patient data as an embedded sub-document. Querying active orders requires matching `patient.we_infuse_id_IV` and filtering by `deleted: { $ne: true }`. The raw MongoDB driver is currently used in `ordersTracker.ts` for aggregation-heavy pipelines, but for a lightweight `find` that returns only `_id` and `assignedTo`, the raw driver is also the pragmatic path since there is no Mongoose model for `neworders`. **Use the raw driver** for this specific query — scoped to the helper only — since no Mongoose model wraps `neworders`.

### Existing services to use

- `createNotification()` — `apps/web/services/mongodb/notification.ts`
- `getSignalRService()` — `apps/web/services/signalr/index.ts` (returns a `Promise<SignalRService>`)
- `FeatureFlagService.getFeatureFlag(FeatureFlag.ORDER_DOCUMENT_NOTIFICATIONS)` — feature flag guard
- `Patient` Mongoose model — `apps/web/models/Patient.ts` (has `we_infuse_id_IV: Number`, indexed)
- `connectToDatabase()` — `apps/web/services/mongodb/connect.ts` (raw driver access for `neworders`)
- `COLLECTION` constant — `apps/web/utils/constants/collections.ts` (`COLLECTION.NewOrders = 'neworders'`)

### Feature flag

`FeatureFlag.ORDER_DOCUMENT_NOTIFICATIONS` is already defined. The helper must check it first and return immediately if disabled.

### Existing tests

`apps/web/services/mongodb/document.test.ts` exists and has full coverage of `createDocumentRecord()`. The split-pdf route has `apps/web/app/api/documents/split-pdf/route.test.ts`. The weinfuseUploader has `apps/web/services/jobs/definitions/jotFormSubmission/__tests__/weinfuseUploader.test.ts`. All three files will need new test cases for the notification trigger call.

---

## Implementation Steps

### Step 1 — RED: Write failing tests for `notifyDocumentCreated()`

- **Files**:
  - `apps/web/services/notifications/documentNotification.test.ts` (create)

- **What**: Write the full test suite for `notifyDocumentCreated()` before writing any implementation. The tests define the contract: feature flag off → no-op; `patientId` missing → no-op; patient not found → no-op; no active orders → no-op; one order → one notification + one broadcast; two orders → two notifications + two broadcasts; notification failure → error logged, broadcast still attempted; broadcast failure → error logged, function resolves (never throws).

  Test cases:

  ```
  describe('notifyDocumentCreated')
    when feature flag ORDER_DOCUMENT_NOTIFICATIONS is disabled
      ✓ returns without querying patient or orders
    when patientId is undefined
      ✓ returns without querying patient or orders
    when patientId is an empty string
      ✓ returns without querying patient or orders
    when patient is not found in the database
      ✓ returns without querying orders
      ✓ logs a warn
    when patient has no we_infuse_id_IV
      ✓ returns without querying orders
    when no active neworders exist for the patient
      ✓ does not call createNotification
      ✓ does not call broadcastToAll
    when one active order is found
      ✓ calls createNotification with type 'user', targetUserId = order.assignedTo
      ✓ notification entities include { entityType: 'order', entityId: order._id } and { entityType: 'document', entityId: documentId }
      ✓ notification title includes the category
      ✓ notification message includes the source label ('upload' → 'Upload', 'e-order' → 'E-Order')
      ✓ calls broadcastToAll with event name 'documentAdded' and payload containing documentId, orderId, patientId, source, category
      ✓ calls broadcastToAll after createNotification
    when two active orders are found for the same patient
      ✓ calls createNotification twice (once per order)
      ✓ calls broadcastToAll twice (once per order)
    when createNotification throws
      ✓ logs the error
      ✓ does not re-throw (function resolves)
      ✓ still attempts broadcastToAll for the remaining orders
    when broadcastToAll throws
      ✓ logs the error
      ✓ does not re-throw (function resolves)
  ```

  Mock boundaries: `@/services/featureFlags/featureFlagService`, `@/models/Patient`, `@/services/mongodb/connect` (raw driver), `@/services/mongodb/notification`, `@/services/signalr`, `@/utils/logger`.

  The tests fail at this point because the implementation file does not exist.

---

### Step 2 — GREEN + REFACTOR: Implement `notifyDocumentCreated()`

- **Files**:
  - `apps/web/services/notifications/documentNotification.ts` (create)

- **What**: Implement the helper to make Step 1 tests pass.

  **Type definition** (in the same file, not a separate types file — the module is small):

  ```typescript
  export type DocumentNotificationSource = 'upload' | 'e-order';

  export interface NotifyDocumentCreatedParams {
    documentId: string;
    patientId: string | undefined;
    category: string;
    source: DocumentNotificationSource;
    createdBy: string;
  }
  ```

  **Logic sequence**:
  1. Check `ORDER_DOCUMENT_NOTIFICATIONS` flag — return if disabled.
  2. Guard: return if `patientId` is falsy.
  3. Look up `Patient.findById(patientId).select('we_infuse_id_IV').lean()` — return with `logger.warn` if not found or `we_infuse_id_IV` is absent.
  4. Query `neworders` via raw driver: `db.collection(COLLECTION.NewOrders).find({ 'patient.we_infuse_id_IV': patient.we_infuse_id_IV, deleted: { $ne: true } }, { projection: { _id: 1, assignedTo: 1 } }).toArray()` — return if empty.
  5. For each order: call `createNotification()` + call `(await getSignalRService()).broadcastToAll('documentAdded', payload)`. Wrap each in `try/catch` — log errors, never throw.

  **Notification payload**:
  - `type: 'user'`
  - `priority: 'normal'`
  - `targetUserId: order.assignedTo`
  - `title: 'New Document: ${category}'`
  - `message: 'A new document was added via ${sourceLabel}.'` where `sourceLabel` is `'Upload'` for `'upload'` and `'E-Order'` for `'e-order'`
  - `entities: [{ entityType: 'order', entityId: String(order._id) }, { entityType: 'document', entityId: documentId }]`
  - `createdBy: 'system'`

  **SignalR broadcast payload**:
  ```typescript
  {
    event: 'documentAdded',
    documentId,
    orderId: String(order._id),
    patientId,
    source,
    category,
  }
  ```

  The entire function body is wrapped in a top-level `try/catch` that logs and swallows — the helper is designed to never surface errors to callers.

  Run `npm run test -- documentNotification` from `apps/web/` — all Step 1 tests should go green.

---

### Step 3 — RED → GREEN: Wire trigger into the split-pdf route (Entry Points 1A + 1B)

- **Files**:
  - `apps/web/app/api/documents/split-pdf/route.test.ts` (modify — add test cases)
  - `apps/web/app/api/documents/split-pdf/route.ts` (modify — add trigger calls)

- **What**:

  **Tests first** — add two new describe blocks to the existing test file. Each block tests one entry point call site:

  ```
  describe('POST /api/documents/split-pdf — full upload (Entry Point 1A)')
    when document creation succeeds and patient is matched
      ✓ calls notifyDocumentCreated with source 'upload', correct patientId, category, and documentId
    when document creation succeeds but patient is not matched (patientId undefined)
      ✓ calls notifyDocumentCreated with patientId undefined (helper handles the skip)
    when notifyDocumentCreated throws (it should not, but guard test)
      ✓ response is still 200 (trigger errors do not fail the upload)

  describe('POST /api/documents/split-pdf — split upload (Entry Point 1B)')
    when document creation succeeds and patient is matched
      ✓ calls notifyDocumentCreated with source 'upload', correct patientId, category, and documentId
    when notifyDocumentCreated throws
      ✓ response is still 200
  ```

  Mock `@/services/notifications/documentNotification` — assert `notifyDocumentCreated` is called with the right arguments.

  **Implementation** — at each of the two `createDocumentRecord()` call sites in `route.ts`, fire the trigger immediately after:

  ```typescript
  // Entry Point 1A — inside the pdfDocuments.forEach loop, after await createDocumentRecord(...)
  void notifyDocumentCreated({
    documentId: String(document._id),
    patientId: patient?._id?.toString(),
    category: uploadData.category,
    source: 'upload',
    createdBy: session.user?.id || session.user?.email || 'system',
  });

  // Entry Point 1B — after await createDocumentRecord(...) in the split path
  void notifyDocumentCreated({
    documentId: String(createdDoc._id),
    patientId: mongoPatientId,
    category: uploadData.category,
    source: 'upload',
    createdBy: session.user?.id || session.user?.email || 'system',
  });
  ```

  Note: `createDocumentRecord()` returns `IDocument` so the document `_id` is available from the return value. The existing call sites discard the return value — they need to capture it to pass `documentId` to the trigger. Update the local variable assignment accordingly (`const createdDoc = await createDocumentRecord(...)`).

  Use `void` (fire-and-forget) to match the existing non-blocking pattern used for OCR job scheduling. The helper never throws, so no wrapping try/catch is required at the call site.

  Run `npm run test -- route.test` from `apps/web/app/api/documents/split-pdf/` — new tests should go green.

---

### Step 4 — RED → GREEN: Wire trigger into weinfuseUploader (Entry Point 2)

- **Files**:
  - `apps/web/services/jobs/definitions/jotFormSubmission/__tests__/weinfuseUploader.test.ts` (modify — add test cases)
  - `apps/web/services/jobs/definitions/jotFormSubmission/weinfuseUploader.ts` (modify — add trigger call)

- **What**:

  **Tests first** — add a new describe block to the existing uploader test file:

  ```
  describe('weinfuseUploader — document notification trigger')
    when document creation succeeds and patientDbId is set
      ✓ calls notifyDocumentCreated with source 'e-order', patientDbId, category, and documentId
    when document creation succeeds but patientDbId is undefined
      ✓ calls notifyDocumentCreated with patientId undefined
    when notifyDocumentCreated throws
      ✓ upload result is still { success: true } (trigger errors do not fail the job)
  ```

  Mock `@/services/notifications/documentNotification`.

  **Implementation** — after the `await createDocumentRecord(...)` call at line ~330 in `weinfuseUploader.ts`, capture the return value and fire the trigger:

  ```typescript
  const createdDoc = await createDocumentRecord({
    intakeId: intake._id.toString(),
    patientId: patientDbId,
    category: uploadData.category,
    description: uploadData.description,
    blobPath: pdfDocument.pdfDocument,
    pageCount: pdfDocument.imageCount,
    createdBy: intake.audit?.createdBy,
  });

  void notifyDocumentCreated({
    documentId: String(createdDoc._id),
    patientId: patientDbId,
    category: uploadData.category,
    source: 'e-order',
    createdBy: 'system',
  });
  ```

  The existing call site already discards the return value inside a try/catch. Capture it as `createdDoc` and add the `void notifyDocumentCreated(...)` call inside the same try block, after the document record is created, before the catch.

  Run `npm run test -- weinfuseUploader` — new tests should go green.

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/services/notifications/documentNotification.ts` | Create | Helper function `notifyDocumentCreated()` — feature flag guard, patient lookup, order lookup, notification creation, SignalR broadcast |
| `apps/web/services/notifications/documentNotification.test.ts` | Create | Full unit test suite for `notifyDocumentCreated()` covering all paths |
| `apps/web/app/api/documents/split-pdf/route.ts` | Modify | Capture return value of `createDocumentRecord()` at both call sites; fire `notifyDocumentCreated()` after each successful create |
| `apps/web/app/api/documents/split-pdf/route.test.ts` | Modify | Add test cases verifying `notifyDocumentCreated` is called with correct arguments at both call sites |
| `apps/web/services/jobs/definitions/jotFormSubmission/weinfuseUploader.ts` | Modify | Capture return value of `createDocumentRecord()`; fire `notifyDocumentCreated()` after successful create |
| `apps/web/services/jobs/definitions/jotFormSubmission/__tests__/weinfuseUploader.test.ts` | Modify | Add test cases verifying `notifyDocumentCreated` is called with correct arguments |

---

## Testing Strategy

### Unit tests — `documentNotification.test.ts`

All mocked at the boundary. Tests assert behavior, not internals:

- Feature flag disabled → function returns, zero calls to patient model, notification service, or SignalR
- `patientId` falsy → same
- Patient not in DB → warn logged, no further calls
- Patient has no `we_infuse_id_IV` → warn logged, no further calls
- No active orders → no notification, no broadcast
- One order → `createNotification` called once with correct shape; `broadcastToAll` called once with correct payload; `targetUserId` equals `order.assignedTo`
- Two orders → both called twice, once per order
- `createNotification` rejects → error logged, no re-throw, broadcast still attempted for remaining orders
- `broadcastToAll` rejects → error logged, no re-throw
- `source: 'upload'` → `message` contains `'Upload'`
- `source: 'e-order'` → `message` contains `'E-Order'`

### Unit tests — split-pdf route additions

- Assert `notifyDocumentCreated` mock is called once per successfully created document in the full-upload loop
- Assert `notifyDocumentCreated` mock is called once after the split-upload create
- Assert response is 200 even when `notifyDocumentCreated` mock rejects (regression guard)
- Assert `patientId: undefined` is passed when no patient is matched

### Unit tests — weinfuseUploader additions

- Assert `notifyDocumentCreated` is called with `source: 'e-order'`
- Assert `notifyDocumentCreated` is called with `patientId = patientDbId`
- Assert upload result is `{ success: true }` when `notifyDocumentCreated` rejects

### Manual verification

1. Upload a document for a patient who has an active `neworders` order. In MongoDB, verify a new `notifications` document appears with `targetUserId = order.assignedTo`, `entities` referencing both the order and the document, and `type = 'user'`.
2. Confirm a SignalR `documentAdded` event is broadcast (visible in Azure Portal → SignalR → Metrics → Messages Sent, or via a temporary console.log in the client).
3. Upload a document for an intake with no matched patient — verify no notification document is created and no error appears in application logs.
4. Trigger a JotForm job manually (or simulate) — verify the same notification creation behavior as step 1 with `source = 'e-order'`.
5. Disable `ORDER_DOCUMENT_NOTIFICATIONS` flag via admin panel — upload a document, verify no notification is created.

---

## Security Considerations

- **Input validation**: `patientId` is validated as truthy before any DB query. No user-supplied string is used raw in a query without the Patient model's ObjectId coercion.
- **Authorization**: The helper runs server-side only (service layer), never in a browser context. No auth check is needed inside the helper itself — callers are already inside authenticated API routes or background jobs.
- **PHI handling**: The notification `title` and `message` contain only document `category` (e.g., `'Labs'`) and source label — no patient name, no SSN, no DOB. The SignalR broadcast payload contains only IDs and metadata. No PHI surfaces in logs from this helper beyond what `createNotification()` already logs.
- **Feature flag gating**: Checked at the top of the helper. If the flag is missing from MongoDB the `FeatureFlagService` returns the code-defined default (disabled for new flags until explicitly enabled).
- **Fire-and-forget safety**: The helper is wrapped in a top-level try/catch and uses `void` at call sites. A failure in notification delivery can never propagate to fail a document upload or background job.

---

## Resolved Questions

1. **Multiple orders per patient**: **Resolved.** One document added = one notification per matching order. Each notification links to both the document and the specific order. Target user is the order's `assignedTo`. If a patient has two active orders, two notifications are created — one per order, each targeting that order's assigned user.

2. **`assignedTo` may be absent**: **Resolved.** Skip orders with no `assignedTo` — unassigned orders don't represent a notification because there's no user to send to.

3. **Fax document timing**: **Resolved.** Fax documents only reach the `documents` collection when a user manually processes them via the split-pdf route. Notification fires at manual processing time, not at fax receipt. This is expected behavior.

4. **`neworders` query approach**: **Resolved.** Use raw MongoDB driver consistent with existing `ordersTracker.ts`. We don't want to be intrusive with the existing code — adapt to what's there.
