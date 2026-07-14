# MLID-2450 — Propagate Notifications to Maintenance Orders on Intakes Upload

## Task Reference

- **Jira**: [MLID-2450](https://localinfusion.atlassian.net/browse/MLID-2450)
- **Epic**: [MLID-2417](https://localinfusion.atlassian.net/browse/MLID-2417) — epic plan at `docs/agomez/plans/MLID-2417-plan-progress.md`
- **Story Points**: 3
- **Branch**: `feature/MLID-2450-maintenance-intakes-upload-propagation`
- **Base Branch**: `epic/MLID-2417-document-notifications-maintenance`
- **Status**: To Do

---

## Summary

The `getOrdersByPatient` lean path in `apps/web/services/mongodb/ordersTracker.ts` currently queries only the `neworders` collection when called with `{ expandRefs: false }`. `notifyDocumentCreated` in `apps/web/services/notifications/documentNotification.ts` calls this lean path at upload time, so maintenance orders for the same patient receive no notification. This task widens the lean branch to accept a `collections` option and changes the one `notifyDocumentCreated` call-site to pass `['new', 'maintenance']`, which covers both the intakes-review split-pdf upload paths. The D4-T2 e-order/fax path routes through the identical `notifyDocumentCreated` call and therefore requires no additional code change — it is covered by this task as a side effect.

---

## Acceptance Criteria

- On intakes-review document upload (split-pdf path A and path B), every maintenance order whose patient matches the uploaded document receives a notification propagation for `(order, order.assignedTo, document)`.
- Maintenance orders whose `assignedTo` is empty are skipped (no notification row, no SignalR broadcast). The existing assignedTo guard already handles this inside `propagateDocumentNotificationToOrder`.
- The eligibility rule still applies: `category === 'Order'` documents are implicitly skipped at upload time because they are not linked to any order yet.
- When `ORDER_DOCUMENT_NOTIFICATIONS` is OFF, no propagation runs.
- Existing new-orders propagation behavior on intakes upload is unchanged. The default value of `collections` in the lean branch is `['new']`, which is a zero-change default for all callers that do not pass the option.

---

## Codebase Analysis

### `getOrdersByPatient` in `apps/web/services/mongodb/ordersTracker.ts`

The function has three overloads today. The lean branch (triggered by `expandRefs: false`) queries only `getOrdersCollection('new')` (`'neworders'`) and projects `{ _id: 1, assignedTo: 1 }`. The enriched branch (triggered by default or `expandRefs: true`) already queries both `getOrdersCollection('new')` and `getOrdersCollection('maintenance')` via a `Promise.all`. The private `getOrdersCollection` helper supports `'new'`, `'maintenance'`, and `'lead'` — nothing needs to change there.

`OrderRef` is defined as `{ _id: string; assignedTo?: string }`. The lean projection provides both `_id` and `assignedTo`, so maintenance orders need no extra projected fields.

The existing lean-branch overload signature is `(patientId: string, options: { expandRefs: false })`. This task extends the options type to `{ expandRefs: false; collections?: OrderCategory[] }`. The implementation signature's options type gains the same `collections` field.

### `notifyDocumentCreated` in `apps/web/services/notifications/documentNotification.ts`

This is the only caller of `getOrdersByPatient` with `{ expandRefs: false }`. It already gates on `ORDER_DOCUMENT_NOTIFICATIONS`, early-returns when `patientId` is absent, and iterates orders calling `propagateDocumentNotificationToOrder` per order. The orchestrator reads only `order._id` and `order.assignedTo` from the `OrderRef`, both of which the lean projection already provides. The change is exactly one line: add `collections: ['new', 'maintenance']` to the existing options object.

### Callers map — signature-change safety

- `apps/web/app/api/orders/by-patient/route.ts` calls `getOrdersByPatient(patientId)` — the enriched default, no options passed. Not affected.
- `apps/web/services/notifications/documentNotification.ts` is the only lean caller. It is the one to update.
- No other file passes `{ expandRefs: false }`.

### Trigger sites — no change needed

`apps/web/app/api/documents/split-pdf/route.ts` paths A (line-range around 358) and B (line-range around 624) both call `notifyDocumentCreated` with `{ documentId, patientId, category, source: 'upload', createdBy, actorId, auditSource: 'user', linkedOrderIds }`. These call sites make no collection assumption; they are unaffected.

### Why D4-T2 is also covered

The e-order/fax upload path in `apps/web/services/jobs/definitions/jotFormSubmission/weinfuseUploader.ts` calls `notifyDocumentCreated` with `source` resolved to `'fax'` or `'e-order'`. It routes through the exact same `notifyDocumentCreated` → `getOrdersByPatient({ expandRefs: false })` chain. Once `notifyDocumentCreated` passes `collections: ['new', 'maintenance']`, the background-job path fans out to maintenance orders automatically. D4-T2 (MLID-2451) therefore requires only verification and test coverage on the job path — no additional production code change.

### Existing test file patterns to mirror

- `apps/web/services/mongodb/__tests__/ordersTracker.test.ts` — mocks `connectToDatabase` via `jest.mock('../connect')`; builds a fake `db` with `collection: jest.fn()` returning `{ aggregate }` where aggregate returns `{ toArray }`. Functions are imported dynamically: `const { getOrdersByPatient } = await import('../ordersTracker')`. The lean test asserts the exact pipeline `[{ $match: { patient, deleted: { $ne: true } } }, { $project: { _id: 1, assignedTo: 1 } }]` and that `db.collection` was called with `'neworders'` only.
- `apps/web/services/notifications/documentNotification.test.ts` — mocks `@/services/mongodb/ordersTracker` (`getOrdersByPatient`, `getOrderById`); has a `mockOrdersFound(orders)` helper; tests feature-flag-off path (asserts `getOrdersByPatient` not called), `assignedTo`-empty skip (no `createNotification`), and per-order fan-out (`createNotification` called once per order).
- `apps/web/app/api/documents/split-pdf/__tests__/route.document.test.ts` — has a `document notification triggers` describe that verifies `notifyDocumentCreated` is called with the correct `documentId`, `patientId`, and `source`. It mocks `notifyDocumentCreated` as a no-op. No change to this file is needed — the route wiring is already verified there. Adding redundant route-level tests for maintenance fan-out would duplicate coverage that belongs at the `documentNotification` and `ordersTracker` layers.

---

## Implementation Steps

### Step 1 — Create the feature branch

- **Files**: (git operation only)
- **What**: From `epic/MLID-2417-document-notifications-maintenance`, create `feature/MLID-2450-maintenance-intakes-upload-propagation`. All commits in this task land on this branch. Commits happen only when the user explicitly asks — do not commit after each individual change; run tests first and wait for the go-ahead.

### Step 2 — RED: write failing unit tests for the `collections` option on `getOrdersByPatient`

- **Files**: `apps/web/services/mongodb/__tests__/ordersTracker.test.ts`
- **What**: Add a new `describe('getOrdersByPatient — lean path with collections option')` block inside the file. Write the following test cases:

  **Test case A — lean default still queries only `'neworders'` (regression guard)**

  `'calls db.collection with neworders only when collections is not provided'` — call `getOrdersByPatient(patientId, { expandRefs: false })`, assert `db.collection` was called exactly once with `'neworders'`, assert `db.collection` was NOT called with `'maintenanceorders'`, assert the returned array is the flattened result of the single collection.

  **Test case B — lean with `collections: ['new', 'maintenance']` queries both collections**

  `'calls db.collection with both neworders and maintenanceorders when collections includes maintenance'` — call `getOrdersByPatient(patientId, { expandRefs: false, collections: ['new', 'maintenance'] })`, assert `db.collection` was called with `'neworders'` and with `'maintenanceorders'`, assert the result is a concatenation of the two `toArray` return values (order: new first, then maintenance).

  **Test case C — same pipeline runs against each collection**

  `'runs the lean projection pipeline against each collection in the collections array'` — call with `collections: ['new', 'maintenance']`, assert that the `aggregate` spy was called twice and that both calls received the same pipeline `[{ $match: { patient: patientId, deleted: { $ne: true } } }, { $project: { _id: 1, assignedTo: 1 } }]`.

  Run from `apps/web/`:
  ```
  npm run test -- ordersTracker.test
  ```
  Expected: tests A, B, and C fail because the `collections` option does not exist yet. The existing lean test (default `'neworders'` only) continues to pass — if it fails, stop and investigate before proceeding.

- **Commit**: `[MLID-2450] - test(orders): RED — failing unit tests for getOrdersByPatient multi-collection lean path`

### Step 3 — GREEN: implement the `collections` option on `getOrdersByPatient`

- **Files**: `apps/web/services/mongodb/ordersTracker.ts`
- **What**: Extend the `{ expandRefs: false }` overload and the implementation signature's options type to include `collections?: OrderCategory[]`. Rewrite the lean branch to:

  ```typescript
  if (!expandRefs) {
    const targetCollections = options.collections ?? ['new'];
    const results = await Promise.all(
      targetCollections.map((category) =>
        db
          .collection(getOrdersCollection(category))
          .aggregate<OrderRef>([
            { $match: { patient: patientId, deleted: { $ne: true } } },
            { $project: { _id: 1, assignedTo: 1 } },
          ])
          .toArray()
      )
    );
    return results.flat();
  }
  ```

  The enriched branch (`expandRefs: true`) is not touched — it already queries both collections unconditionally. The `getOrdersCollection` helper is not touched. The `OrderRef` type is not touched.

  Run from `apps/web/`:
  ```
  npm run test -- ordersTracker.test
  ```
  Expected: all three new tests (A, B, C) pass. All pre-existing lean and enriched tests continue to pass.

- **Commit**: `[MLID-2450] - feat(orders): extend getOrdersByPatient lean path with collections option`

  Detailed commit body:

  The lean branch of getOrdersByPatient previously queried only neworders. A new
  optional `collections: OrderCategory[]` field on the expandRefs-false overload
  lets callers specify which collections to fan out across. The default is ['new'],
  which preserves existing behavior for every caller that does not pass the option.

  The enriched (expandRefs: true) branch already queries both neworders and
  maintenanceorders unconditionally and is unchanged.

### Step 4 — RED: write a failing unit test for `notifyDocumentCreated` calling with `collections`

- **Files**: `apps/web/services/notifications/documentNotification.test.ts`
- **What**: Add one focused test case to the existing `describe` block for `notifyDocumentCreated`. The test asserts the behavior change at this layer — that `getOrdersByPatient` is called with both `expandRefs: false` and `collections: ['new', 'maintenance']`.

  **Test case D — lean caller passes maintenance collection**

  `'calls getOrdersByPatient with expandRefs false and collections new and maintenance'` — arrange: `mockOrdersFound([])` (empty result is fine, the assertion is about the call args, not fan-out). Call `notifyDocumentCreated({ documentId: 'doc-1', patientId: 'patient-1', category: 'Referral Copy', source: 'upload', createdBy: 'user-1', actorId: 'user-1', auditSource: 'user', linkedOrderIds: [] })`. Assert: `getOrdersByPatient` was called with `'patient-1'` and `{ expandRefs: false, collections: ['new', 'maintenance'] }`.

  Run from `apps/web/`:
  ```
  npm run test -- documentNotification.test
  ```
  Expected: test D fails because `notifyDocumentCreated` still passes only `{ expandRefs: false }` — the `collections` key is absent. All existing `documentNotification.test.ts` tests continue to pass.

- **Commit**: `[MLID-2450] - test(notifications): RED — failing test asserting notifyDocumentCreated passes collections option`

### Step 5 — GREEN: update `notifyDocumentCreated` to pass `collections`

- **Files**: `apps/web/services/notifications/documentNotification.ts`
- **What**: In `notifyDocumentCreated`, locate the `getOrdersByPatient(patientId, { expandRefs: false })` call and change it to:

  ```typescript
  const orders = await getOrdersByPatient(patientId, { expandRefs: false, collections: ['new', 'maintenance'] });
  ```

  That is the entire production change in this file. No other lines in `notifyDocumentCreated` or the rest of the file are modified.

  Run from `apps/web/`:
  ```
  npm run test -- documentNotification.test
  ```
  Expected: test D passes. All existing tests continue to pass (flag-off early-return, assignedTo-empty skip, per-order fan-out — these use mock order fixtures regardless of which collection they notionally come from, so they are unaffected).

- **Commit**: `[MLID-2450] - feat(notifications): pass collections option to getOrdersByPatient in notifyDocumentCreated`

  Detailed commit body:

  notifyDocumentCreated previously called getOrdersByPatient({ expandRefs: false }),
  which only queried neworders. Passing collections: ['new', 'maintenance'] fans the
  lean lookup out to maintenanceorders as well.

  This single line change covers both D4-T1 (intakes-review split-pdf paths A and B)
  and D4-T2 (e-order/fax background job), because both entry points route through
  notifyDocumentCreated → getOrdersByPatient.

### Step 6 — Run the full quality gate

- **Files**: none modified in this step
- **What**: Run all checks from `apps/web/` (types:check and lint must run from `apps/web/`, not the repo root, due to `tsconfig.eslint.json` resolution):

  ```
  npm run test -- ordersTracker.test
  npm run test -- documentNotification.test
  npm run types:check
  npm run lint
  ```

  For coverage on the two modified source files (both already have test files — 80% threshold applies):

  ```
  npm run test -- --coverage ordersTracker.test
  npm run test -- --coverage documentNotification.test
  ```

  If `types:check` or `lint` surface issues, fix them in the same branch before committing. Do not create a separate formatting-only commit — fold any type or lint fix into the nearest relevant commit or create a focused fix commit: `[MLID-2450] - fix(notifications): resolve type annotation on collections option`.

### Step 7 — Manual verification

- **Files**: none
- **What**: End-to-end verification requires `ORDER_DOCUMENT_NOTIFICATIONS` to be ON. Toggle it at `/admin/feature-flags` before proceeding.

  **Test order and patient:**
  - Maintenance order display ID `0yn`, `_id` `6a34840be879c9374538c919`
  - Patient `_id` `6a15dda68c4205d331775425` (WeInfuse ID `9562`)
  - Order is assigned to test user `695c25a530f7a75a8b1ed8c8`

  **Upload a NON-Order-category document:**
  1. Navigate to the intakes-review split-pdf flow for patient `9562`.
  2. Upload a document with any category other than `'Order'` (for example, `'Referral Copy'`).
  3. After the upload completes, run the following mongosh query against the `notifications` collection to confirm the maintenance order received a notification row:

     ```javascript
     db.notifications.findOne({
       "entities.entityType": "order",
       "entities.entityId": "6a34840be879c9374538c919"
     })
     ```

     Expected: a document with `targetUserId: '695c25a530f7a75a8b1ed8c8'`, `entities` containing both the order tuple and the document tuple, and `title` matching `"New Document: Referral Copy"` (or equivalent).

  4. Navigate to `/orders-tracker/maintenance/6a34840be879c9374538c919/documents`. Confirm the Documents tab badge shows a non-zero count and a "New" chip appears next to the uploaded document row.

  5. Navigate to `/orders-tracker/maintenance/6a34840be879c9374538c919/order-history`. Confirm a log entry appears for the uploaded document.

  **Verify Order-category document is NOT propagated at upload time:**
  1. Upload a document with `category === 'Order'` for the same patient.
  2. Run `db.notifications.findOne(...)` as above.
  3. Confirm no new notification row is created for this document (eligibility gate still applies — `'Order'`-category docs only propagate when linked, not at upload time).

  **Verify new-orders behavior is unchanged:**
  4. Find a new order for any patient in the intakes review queue.
  5. Upload a non-Order-category document.
  6. Confirm the new order still receives a notification row (regression check).

  **Verify assignedTo-empty maintenance orders are skipped:**
  7. Find (or manually create) a maintenance order for patient `9562` with no `assignedTo` value.
  8. Upload a non-Order-category document for that patient.
  9. Confirm no notification row is created for the unassigned maintenance order.

  **Cleanup query** (run after verification to remove seeded notifications):
  ```javascript
  db.notifications.deleteMany({
    "entities.entityType": "order",
    "entities.entityId": "6a34840be879c9374538c919",
    createdBy: "system"
  })
  ```
  Also clean up the `notificationreads` collection for any rows referencing the deleted notification IDs.

  Create a companion testing doc at `docs/agomez/testing/MLID-2450.md` capturing the exact upload steps, the mongosh confirm query, the cleanup query, and the results of each verification point. This doc is for personal reference and is not committed to the feature branch.

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/services/mongodb/ordersTracker.ts` | Modify | Extend the `{ expandRefs: false }` overload and the lean branch to accept `collections?: OrderCategory[]`; rewrite the lean branch to fan out across all listed collections and concatenate results; default `['new']` preserves existing behavior |
| `apps/web/services/notifications/documentNotification.ts` | Modify | Change the `getOrdersByPatient` call in `notifyDocumentCreated` to pass `collections: ['new', 'maintenance']` |
| `apps/web/services/mongodb/__tests__/ordersTracker.test.ts` | Modify | Add `describe('getOrdersByPatient — lean path with collections option')` with three test cases: default queries only neworders (regression), `['new','maintenance']` queries both and concatenates, same pipeline runs against each collection |
| `apps/web/services/notifications/documentNotification.test.ts` | Modify | Add one test case asserting `notifyDocumentCreated` calls `getOrdersByPatient` with `{ expandRefs: false, collections: ['new', 'maintenance'] }` |

---

## Testing Strategy

### Automated tests

**`apps/web/services/mongodb/__tests__/ordersTracker.test.ts` — new lean-path block:**

| Test case | Behavior verified |
|---|---|
| Default `collections` queries only `'neworders'` | Zero behavior change for all existing callers |
| `collections: ['new','maintenance']` calls `db.collection` with both names | Both collections are queried in parallel |
| Same pipeline runs against each collection | No projection drift — both aggregations use `{ $match, $project }` with the same fields |

**`apps/web/services/notifications/documentNotification.test.ts` — new caller assertion:**

| Test case | Behavior verified |
|---|---|
| `notifyDocumentCreated` calls `getOrdersByPatient` with `collections: ['new','maintenance']` | The call-site change is present and wired correctly |

**Existing tests that must continue to pass (regression evidence):**

- `ordersTracker.test.ts` — all existing lean (neworders-only) and enriched (both collections) tests
- `documentNotification.test.ts` — flag-off early-return, assignedTo-empty skip, per-order fan-out
- `apps/web/app/api/documents/split-pdf/__tests__/route.document.test.ts` — `document notification triggers` describe; `notifyDocumentCreated` is mocked as a no-op here, so no change to this file is needed and no redundant trigger-path tests should be added for maintenance fan-out

**Coverage gate:**

Run `npm run test -- --coverage ordersTracker.test` and `npm run test -- --coverage documentNotification.test` from `apps/web/`. Both files have existing test suites; the 80% threshold must be maintained or improved.

### Manual verification

See Step 7 above. The key checkpoints:

1. A non-Order-category upload for patient `9562` creates a `notifications` row with the maintenance order `6a34840be879c9374538c919` in the entities array.
2. The maintenance order Documents tab badge and "New" chip reflect the new notification live.
3. The maintenance order History tab shows the upload audit entry.
4. An Order-category document upload does NOT create a notification row for the maintenance order (eligibility gate).
5. An unassigned maintenance order for the same patient does NOT receive a notification row.
6. A new order for any patient still receives its notification on upload (regression).

---

## Security Considerations

- **Input validation**: No new API inputs are introduced. The `collections` option is an internal type (`OrderCategory[]`) used only within the service layer — it is never derived from request data.
- **Authorization**: No change to authentication or authorization logic. The `getOrdersByPatient` function is called from behind the `ORDER_DOCUMENT_NOTIFICATIONS` feature flag gate in `notifyDocumentCreated`, which is itself called from authenticated API routes and background jobs.
- **Feature flag gating**: `ORDER_DOCUMENT_NOTIFICATIONS` already gates the entire `notifyDocumentCreated` function. When the flag is OFF, `notifyDocumentCreated` returns early and `getOrdersByPatient` is never called — the `collections` option is irrelevant.
- **PHI handling**: No new patient data fields are exposed. The lean projection (`{ _id: 1, assignedTo: 1 }`) returns only internal IDs — no PHI fields are introduced.

---

## Open Questions

None. The architecture is confirmed. The `collections` option is the agreed design. The `getOrdersCollection` helper already supports `'maintenance'`. The only change is wiring the option through the lean branch and updating the one caller.
