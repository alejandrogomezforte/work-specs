# MLID-2451 — [D4-T2] - feat(orders): propagate notifications to maintenance orders on e-order/fax upload

## Task Reference

- **Jira**: [MLID-2451](https://localinfusion.atlassian.net/browse/MLID-2451)
- **Story Points**: 1
- **Branch**: `feature/MLID-2451-eorder-fax-maintenance-propagation`
- **Base Branch**: `epic/MLID-2417-document-notifications-maintenance`
- **Status**: To Do

---

## Summary

D4-T1 (MLID-2450) already extended the production fan-out chain to include maintenance orders on every document-notification path, including e-order and fax. This task adds the integration tests that verify the e-order/fax background-job path feeds that chain correctly. No production code changes are expected.

---

## Codebase Analysis

**Trigger site — `apps/web/services/jobs/definitions/jotFormSubmission/weinfuseUploader.ts`:**

Inside `WeInfuseUploader.uploadPDFToWeInfuse`, after a successful WeInfuse upload and `createDocumentRecord` call, the code fire-and-forgets `notifyDocumentCreated` with:
- `source: intake.source === 'spruce' ? 'fax' : 'e-order'`
- `actorId: ''` (always empty on this path)
- `createdBy: 'system'`
- `auditSource: 'webhook'`
- `patientId: patientDbId` (the Mongo `_id` string returned by `Patient.findOne`)
- `linkedOrderIds: createdDoc.linkedOrderIds ?? []`

The call is inside an inner `try/catch` that logs `docError` and does not rethrow. The outer `try/catch` handles WeInfuse upload failures. Both are separate so a failing `notifyDocumentCreated` never surfaces to the job runner.

The exported surface used in tests is `uploadPDFToWeInfuse` (the legacy function wrapper at the bottom of the file), which constructs a `WeInfuseUploader` instance and delegates to `uploadPDFToWeInfuse`.

**No existing test file for `weinfuseUploader.ts`.** `index.test.ts` mocks the entire `./weinfuseUploader` module wholesale (`jest.mock('./weinfuseUploader')`), so it provides no coverage of the uploader internals.

**What must be mocked to reach the `notifyDocumentCreated` call:**

The method has three async dependencies before the notify call:
1. `weinfuseService` (`@/services/integrations/weinfuse`) — must resolve `uploadDocument` successfully.
2. `Storage.downloadFromBlob` (`@/services/storage`) — must resolve a `Buffer`.
3. `Patient.findOne` (`@/models/Patient`) — must resolve with `{ _id: 'patient-db-id', we_infuse_location_id: '...' }`.
4. `createDocumentRecord` (`@/services/mongodb/document`) — must resolve with `{ _id: new ObjectId(), linkedOrderIds: [] }`.
5. `notifyDocumentCreated` (`@/services/notifications/documentNotification`) — the subject of assertion; mock so it can be inspected.
6. `weinfuseService.configure` is called via `ensureConfigured`; env vars `WEINFUSE_API_GATEWAY_URL` and `WEINFUSE_API_GATEWAY_KEY` must be set.
7. `logger` (`@/utils/logger`) — mock to suppress output.

**`documentNotification.test.ts` coverage review:**

The existing 1536-line test file already covers `notifyDocumentCreated` generically with `source: 'e-order'` and `source: 'fax'` values (lines 330-338 region), and it already asserts `getOrdersByPatient` is called with `{ expandRefs: false, collections: ['new', 'maintenance'] }` (the D4-T1 parity test at line 144-151). There is no gap to fill in `documentNotification.test.ts` — adding tests there would duplicate already-covered behavior. All new tests belong in the uploader test file.

**`intake.source` resolution:**
- `intake.source === 'spruce'` → `source: 'fax'`
- Any other value (or undefined) → `source: 'e-order'`

**MLID-2434 self-upload gate:** `propagateDocumentNotificationToOrder` suppresses the in-app notification only when `!!actorId && actorId === assignedTo`. Because `actorId` is always `''` on this path, `!!actorId` is `false`, so the gate never fires. Tests should confirm notifications ARE written even when `assignedTo` is set (the gate is irrelevant here, but it is worth a comment in the test).

---

## Implementation Steps

### Step 1 — RED: Write the failing integration test file

- **Files**: `apps/web/services/jobs/definitions/jotFormSubmission/weinfuseUploader.test.ts` (create)
- **What**: Create a new `@jest-environment node` test file. Mock all boundaries: `@/models/Patient`, `@/services/integrations/weinfuse`, `@/services/storage`, `@/services/mongodb/document`, `@/services/notifications/documentNotification`, and `@/utils/logger`. Set `WEINFUSE_API_GATEWAY_URL` and `WEINFUSE_API_GATEWAY_KEY` in `process.env` inside `beforeEach`. Write tests that call `uploadPDFToWeInfuse` (the exported wrapper function) and assert on the `notifyDocumentCreated` mock. Before any production code exists that makes these tests pass (they already exist), run once to confirm the import resolves correctly and the mock structure is valid.

  Test cases to write:

  **describe: `uploadPDFToWeInfuse — notifyDocumentCreated propagation`**

  - `when intake.source is not 'spruce', calls notifyDocumentCreated with source 'e-order'`
    - Arrange: intake with `source: 'jotform'` (or any non-spruce value), a PDF document in `intake.documents`, patient with `patientId`, `createDocumentRecord` resolves with `{ _id: ObjectId(), linkedOrderIds: [] }`.
    - Assert: `mockNotifyDocumentCreated` was called with `expect.objectContaining({ source: 'e-order' })`.

  - `when intake.source is 'spruce', calls notifyDocumentCreated with source 'fax'`
    - Same setup with `source: 'spruce'`.
    - Assert: `mockNotifyDocumentCreated` was called with `expect.objectContaining({ source: 'fax' })`.

  - `passes patientId (MongoDB _id) from Patient.findOne result`
    - `Patient.findOne` resolves with `{ _id: 'patient-db-id', we_infuse_location_id: 'loc-1' }`.
    - Assert: `mockNotifyDocumentCreated` called with `expect.objectContaining({ patientId: 'patient-db-id' })`.

  - `passes empty actorId and system createdBy`
    - Assert: `mockNotifyDocumentCreated` called with `expect.objectContaining({ actorId: '', createdBy: 'system' })`.

  - `passes auditSource webhook`
    - Assert: `expect.objectContaining({ auditSource: 'webhook' })`.

  - `passes linkedOrderIds from the created document record`
    - `createDocumentRecord` resolves with `{ _id: ObjectId(), linkedOrderIds: ['order-abc'] }`.
    - Assert: `expect.objectContaining({ linkedOrderIds: ['order-abc'] })`.

  - `passes empty linkedOrderIds when createdDoc.linkedOrderIds is undefined`
    - `createDocumentRecord` resolves with `{ _id: ObjectId() }` (no `linkedOrderIds` field).
    - Assert: `expect.objectContaining({ linkedOrderIds: [] })`.

  - `does not await notifyDocumentCreated (fire-and-forget — function returns before notify completes)`
    - Make `mockNotifyDocumentCreated` return a promise that never resolves (`new Promise(() => {})`).
    - Assert: `uploadPDFToWeInfuse` still resolves with `{ success: true }` within a reasonable timeout. This documents the fire-and-forget contract.

  - `when createDocumentRecord throws, does not call notifyDocumentCreated and does not rethrow`
    - `createDocumentRecord` rejects.
    - Assert: `mockNotifyDocumentCreated` not called; `uploadPDFToWeInfuse` still resolves with `{ success: true }`.

Run: `npm run test -- weinfuseUploader.test.ts`
Expected: tests fail only because the mock wiring is wrong (not yet proven) or the module cannot be imported. Fix wiring, confirm tests fail for the right reason (i.e., `notifyDocumentCreated` was not called, or source is wrong) — but since D4-T1 is already merged, tests should actually pass on first run after the wiring is correct. If they all pass immediately, that is the GREEN signal.

### Step 2 — GREEN: Confirm all tests pass against existing production code

- **Files**: No production files changed.
- **What**: Run `npm run test -- weinfuseUploader.test.ts` and confirm all tests in Step 1 pass. Since D4-T1 already landed the fan-out, the tests should be green immediately after the mocks are wired correctly. If any test fails, this is a signal that the production path differs from the documented behavior — surface the discrepancy as a finding before modifying any production code.

Run: `npm run test -- weinfuseUploader.test.ts`
Expected: all tests pass.

### Step 3 — REFACTOR: Run full check suite and clean up

- **Files**: `apps/web/services/jobs/definitions/jotFormSubmission/weinfuseUploader.test.ts`
- **What**: Run type-check and lint. Clean up any test-helper duplication (extract a shared `buildIntake(overrides)` factory if the intake object is repeated across cases). Ensure `beforeEach` calls `jest.clearAllMocks()`. Add a top-level comment explaining that this test file covers only the `notifyDocumentCreated` propagation contract — the WeInfuse upload mechanics themselves are outside this file's scope.

Run from `apps/web/`: `npm run types:check && npm run lint:fix`
Expected: no errors.

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/services/jobs/definitions/jotFormSubmission/weinfuseUploader.test.ts` | Create | Integration tests asserting that `uploadPDFToWeInfuse` calls `notifyDocumentCreated` with the correct params for both e-order and fax source paths |

---

## Testing Strategy

**Unit / integration tests (new file):**

All tests mock `notifyDocumentCreated` at the module boundary and assert on the call arguments. The WeInfuse service, Storage, Patient model, and document service are also mocked so the test does not touch the network or database.

Specific behaviors tested:
- `source` mapping: non-spruce → `'e-order'`; spruce → `'fax'`
- `patientId` threaded from `Patient.findOne` result
- `actorId: ''` and `createdBy: 'system'` always set
- `auditSource: 'webhook'` always set
- `linkedOrderIds` from `createDocumentRecord` result (populated and undefined/fallback cases)
- Fire-and-forget contract: main function resolves even when notify never resolves
- Error isolation: `createDocumentRecord` failure does not call notify and does not rethrow

**Existing tests — no changes needed:**
- `documentNotification.test.ts` already covers the `['new', 'maintenance']` collections call and the `source: 'e-order'` / `source: 'fax'` notification content. Adding duplicates there would create maintenance burden without adding value.

**Manual verification:**

Because `actorId` is always `''` on this path, the MLID-2434 self-upload gate (`!!actorId && actorId === assignedTo`) is always false. Manual verification for this task is:

1. Trigger a fax or e-order intake upload through the background job (use the `/admin/jobs` UI to replay an existing intake, or submit a new JotForm entry with `source !== 'spruce'` for e-order or `source === 'spruce'` for fax).
2. Confirm that after the job completes, every maintenance order whose patient matches shows the document notification in the in-app notification list.
3. Confirm that maintenance orders without an `assignedTo` value do not produce notifications (no error, just silently skipped).
4. Confirm the notification message contains the correct source label (`E-Order` or `Fax`).

Note: the end-to-end plan for MLID-2417 documents a fax/e-order test using patient order `0yn`. On this path `actorId` is always `''`, so the self-upload gate is not a concern — there is no need for the uploader to differ from the assignee to see notifications fire.

---

## Security Considerations

- **Input validation**: None required — this task adds no new production code.
- **Authorization**: None required — test file only.
- **PHI handling**: Test data uses synthetic patient IDs (`'patient-db-id'`, `'patient-mongo-1'`). No real patient data used in tests. The existing production code already avoids logging `patientId` in the broadcast payload (PHI minimization) — tests confirm this contract remains intact via the `notifyDocumentCreated` mock assertion pattern (the `source: 'e-order'` and `patientId` assertions use `expect.objectContaining` on the notify call, not on the broadcast).

---

## Open Questions

None.
