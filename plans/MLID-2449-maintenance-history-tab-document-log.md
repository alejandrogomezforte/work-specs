# MLID-2449 — Show Document Upload Log Entries on Maintenance Order History Tab

## Task Reference

- **Jira**: [MLID-2449](https://localinfusion.atlassian.net/browse/MLID-2449)
- **Epic**: [MLID-2417](https://localinfusion.atlassian.net/browse/MLID-2417) — epic plan at `docs/agomez/plans/MLID-2417-plan-progress.md`
- **Story Points**: 2
- **Branch**: `feature/MLID-2449-maintenance-history-tab-document-log`
- **Base Branch**: `epic/MLID-2417-document-notifications-maintenance`
- **Status**: To Do

---

## Summary

The History tab for order detail (`apps/web/app/orders-tracker/[category]/[orderId]/order-history/page.tsx`) lives under the shared `[category]` dynamic route and has no code path that reads or gates on the category segment. The `useAuditLog` hook, the API route, the `getAuditLog` service, and the `documents` formatter in `auditLog.config.ts` are all category-agnostic. This means maintenance orders at `/orders-tracker/maintenance/[id]/order-history` already render document-log entries correctly given the right audit data.

This task is verification plus maintenance-specific test coverage. Every existing test file for the order-history surface uses a plain `orderId` with no category involvement, so adding a `describe('OrderHistoryPage — maintenance order')` block with a maintenance-style orderId is the entire deliverable. No production code change is expected. If during implementation any category gate is discovered, the scope must be widened to remove it before proceeding.

---

## Acceptance Criteria

- On `/orders/maintenance/[id]/order-history`, the history log shows an entry for each document propagated to this order.
- Documents with `category === 'Order'` appear only when explicitly linked to this order (`document.linkedOrderIds` includes this orderId).
- Documents with any other category appear unconditionally (matches propagation eligibility).
- Each log entry shows at minimum: document name/category, source (Upload / E-Order / Fax), and timestamp.
- Layout, sorting, and pagination match the equivalent new-orders History tab.
- Existing `/orders/new/[id]/order-history` behavior is unchanged.

---

## Codebase Analysis

### Route and page structure

All files live under `apps/web/app/orders-tracker/[category]/[orderId]/order-history/`. The single `[category]` route segment resolves for `new`, `maintenance`, and `lead` alike.

`order-history/page.tsx` reads `params.orderId` via `useParams` — it never reads `params.category`. It calls `useAuditLog({ entityType: 'order', entityId })` and `useAuditLogFilterOptions('order', entityId)` with no category awareness.

### Data flow

`useAuditLog` (`hooks/useAuditLog.ts`) fetches `GET /api/audit-log?entityType=order&entityId=<orderId>`. The API route (`apps/web/app/api/audit-log/route.ts`) performs `getServerSession` auth (any authenticated user) and forwards to `getAuditLog()`. The `getAuditLog` service (`apps/web/services/mongodb/auditLog.ts`) builds a Mongo match on `{ entityType: 'order', entityId }` against a single `auditLogs` collection. There is no field in the collection that discriminates new orders from maintenance orders.

### Formatter

`apps/web/services/mongodb/auditLog.config.ts` contains the `documents` formatter whose `renderChange` reads the document category from `change.from?.category ?? change.to?.category`. It emits:

- `A Document has been added (Document type = Order)` when `from === null` and `category === 'Order'`
- `A Document has been removed (Document type = Order)` when `to === null` and `category === 'Order'`
- `New Document received (Document type = <category>)` for all other categories

This is the format agreed upon in MLID-2413 (D0-T2). No category gate exists.

### Audit entry writer

`apps/web/services/audit/documentAudit.ts` exports `writeOrderDocumentAuditEntry` and `writeOrderDocumentUnlinkAuditEntry`. Both write rows with `{ entityType: 'order', entityId: orderId (string), action: 'update', actorId, source: auditSource, changes: [{ field: 'documents', from, to }] }`. The `entityType` is always `'order'` regardless of which collection the order belongs to. This means maintenance order audit entries sit in the same `auditLogs` collection with the same shape as new-order entries, and the History tab page picks them up by the same `entityType + entityId` match.

### Eligibility and the "Order-category only when linked" rule

The eligibility gate lives in `propagateDocumentNotificationToOrder` (in `apps/web/services/notifications/documentNotification.ts`). `category === 'Order'` documents only pass eligibility when `document.linkedOrderIds` includes the orderId at write time. All other categories pass unconditionally. Because `writeOrderDocumentAuditEntry` is called inside `propagateDocumentNotificationToOrder` only after the eligibility gate passes, no extra read-time filtering is required in the History tab — the entries that exist in `auditLogs` are already those that passed eligibility.

### Existing test files (mirror these)

- `apps/web/app/orders-tracker/[category]/[orderId]/order-history/page.test.tsx` — mocks `next/navigation` `useParams` returning `{ orderId: 'order-123' }` with NO `category` key. Mocks `useAuditLog` and `useAuditLogFilterOptions` as module mocks returning full objects. RTL render and `screen`. Covers title render, hook call params, table content, and loading spinner. No `mockCategory` variable exists or is needed — the page never reads the category param.
- `apps/web/app/orders-tracker/[category]/[orderId]/order-history/hooks/useAuditLog.test.tsx` — `global.fetch` mock, `renderHook + waitFor + act`. Already category-agnostic.
- `apps/web/app/orders-tracker/[category]/[orderId]/order-history/components/AuditLogTable.test.tsx` — table renderer tests including document `renderedChange` strings; covers Order-category removal side.
- `apps/web/services/mongodb/auditLog.config.test.ts` and `apps/web/services/mongodb/__tests__/auditLog.config.test.ts` — formatter tests, category-agnostic.
- `apps/web/services/audit/documentAudit.test.ts` — tests `writeOrderDocumentAuditEntry` and `writeOrderDocumentUnlinkAuditEntry` audit row shape.

### Key difference from D2-T1 (MLID-2448)

D2-T1 required making the `useParams` mock in `documents/page.test.tsx` use a mutable `mockCategory` variable because that page does read `category` from params (for the `showAddDocument` gate). The History tab page never reads `category` at all, so no mock refactor is needed here — just add the new `describe` block with a maintenance orderId.

---

## Implementation Steps

### Step 1 — Create the feature branch

- **Files**: (git operation only)
- **What**: From `epic/MLID-2417-document-notifications-maintenance`, create `feature/MLID-2449-maintenance-history-tab-document-log`. This is the working branch for all commits in this task.

### Step 2 — Add maintenance-category test coverage to order-history/page.test.tsx

- **Files**: `apps/web/app/orders-tracker/[category]/[orderId]/order-history/page.test.tsx`
- **What**: Add a `describe('OrderHistoryPage — maintenance order')` block at the bottom of the file. The block needs its own `beforeEach` that sets `mockOrderId = 'maint-order-456'` (or a local constant) and resets all module mocks with fresh return values appropriate for a maintenance orderId.

  Because `useParams` in this file already returns `{ orderId: 'order-123' }` with no category key, the only change to mock setup is swapping the orderId value — no structural refactor of the mock is needed.

  Write the following test cases inside the new `describe` block:

  **Sub-describe: `'page renders for a maintenance order'`**

  1. `'calls useAuditLog with entityType order and the maintenance orderId'` — render the page with `useParams` returning `{ orderId: 'maint-order-456' }`, assert that the `useAuditLog` mock was called with `{ entityType: 'order', entityId: 'maint-order-456' }`.
  2. `'calls useAuditLogFilterOptions with order and the maintenance orderId'` — same render, assert `useAuditLogFilterOptions` mock called with `('order', 'maint-order-456')`.
  3. `'renders the page title'` — assert `screen.getByRole('heading', { name: /order history/i })` (or whatever the existing title assertion uses) is present.
  4. `'renders a document-added log entry for Order category on a maintenance order'` — set the `useAuditLog` mock to return an entry whose `changes[0]` has `field: 'documents'`, `from: null`, `to: { _id: 'doc-1', category: 'Order', source: 'upload' }`; assert the rendered text contains `A Document has been added (Document type = Order)`.
  5. `'renders a non-Order category document entry (unconditional) on a maintenance order'` — set the `useAuditLog` mock to return an entry whose `changes[0]` has `field: 'documents'`, `from: null`, `to: { _id: 'doc-2', category: 'Referral Copy', source: 'fax' }`; assert the rendered text contains `New Document received (Document type = Referral Copy)`.
  6. `'renders a document-removed log entry for Order category on a maintenance order'` — set the mock to return an entry with `from: { _id: 'doc-1', category: 'Order', source: 'upload' }`, `to: null`; assert the rendered text contains `A Document has been removed (Document type = Order)`.
  7. `'shows loading spinner when isLoading is true for a maintenance order'` — set `useAuditLog` mock to return `{ isLoading: true, auditLogs: [] }`; assert loading indicator is present (mirror the existing new-orders loading test).
  8. `'shows empty state when there are no audit entries for a maintenance order'` — set the mock to return `{ isLoading: false, auditLogs: [] }`; assert the empty-state message is present (mirror existing test).

  **New-orders regression guard** (add inside the existing top-level `describe` block, not in the new maintenance block):

  9. `'existing new-orders behavior unchanged — renders document-added entry for new order'` — assert that with `orderId: 'order-123'` and an Order-category-added entry, the correct text renders. This guard will catch any accidental breakage to the existing mock setup.

  Run the tests from `apps/web/`:

  ```
  npm run test -- order-history/page.test
  ```

  Expected outcome: all tests pass immediately. Because the implementation is already category-agnostic, green on first run is the correct and intended result — it proves the History tab works for maintenance orders. If any test fails, that failure reveals a real category gate that must be removed before committing; do not proceed to Step 3 with a failing test.

- **Commit**: `[MLID-2449] - test(orders): add maintenance-category order-history page tests proving audit log parity`

### Step 3 — Verify the quality gate and run all changed-file checks

- **Files**: none modified — this is a verification step
- **What**: Run the full quality gate from `apps/web/` (types:check and lint must run from `apps/web/`, not the repo root, due to `tsconfig.eslint.json` resolution):

  ```
  npm run test -- order-history/page.test
  npm run types:check
  npm run lint
  ```

  For coverage on the modified test file:

  ```
  npm run test -- --coverage order-history/page.test
  ```

  The coverage gate for `order-history/page.tsx` must remain at or above 80%. Since no new branches are added to production source files in this task, coverage should be stable or improve.

  If `types:check` or `lint` surface issues in the test file (for example, a type annotation on a mock return value), fix them in the same branch before committing. Do not create a separate formatting-only commit.

- **Commit**: no separate commit for this step — only commit if a lint or type fix was required, in which case fold it into the Step 2 commit or create a focused fix commit: `[MLID-2449] - fix(orders): resolve type annotation in maintenance order-history test`

### Step 4 — Manual verification with seeded audit data

- **Files**: none
- **What**: Since D4 (MLID-2450 through MLID-2454) has not yet landed, there are no live maintenance order audit entries produced by the system. Manual verification requires seeding rows directly into the `auditLogs` collection.

  Find a real maintenance order `_id` from the `maintenanceorders` collection. Insert the following two documents into `auditLogs` (replace `<maintenanceOrderId>` with the actual ObjectId string):

  **Example 1 — Order-category linked document (explicit link required)**

  ```json
  {
    "entityType": "order",
    "entityId": "<maintenanceOrderId>",
    "action": "update",
    "actorId": "system",
    "source": "upload",
    "changes": [
      {
        "field": "documents",
        "from": null,
        "to": {
          "_id": "seeded-doc-001",
          "category": "Order",
          "source": "upload"
        }
      }
    ],
    "createdAt": { "$date": "2026-06-22T12:00:00.000Z" }
  }
  ```

  This represents a document with `category === 'Order'` that was explicitly linked to this order (eligibility enforced at write time — no read-time filtering needed). The History tab should render: `A Document has been added (Document type = Order)`.

  **Example 2 — Non-Order category document (unconditional)**

  ```json
  {
    "entityType": "order",
    "entityId": "<maintenanceOrderId>",
    "action": "update",
    "actorId": "system",
    "source": "fax",
    "changes": [
      {
        "field": "documents",
        "from": null,
        "to": {
          "_id": "seeded-doc-002",
          "category": "Referral Copy",
          "source": "fax"
        }
      }
    ],
    "createdAt": { "$date": "2026-06-22T11:00:00.000Z" }
  }
  ```

  This represents a document with any category other than `'Order'` (Referral Copy, Other, etc.) that was auto-attached unconditionally. The History tab should render: `New Document received (Document type = Referral Copy)`.

  Navigate to `/orders-tracker/maintenance/<maintenanceOrderId>/order-history` while logged in. Verify:

  - Both seeded entries appear in the table.
  - Entry 1 shows `A Document has been added (Document type = Order)`.
  - Entry 2 shows `New Document received (Document type = Referral Copy)`.
  - Timestamps render correctly.
  - Pagination and sorting match the new-orders History tab at `/orders-tracker/new/<newOrderId>/order-history`.
  - Feature flag `ORDER_DOCUMENT_NOTIFICATIONS` state does not gate the History tab itself — the audit log renders regardless of the flag (the flag gates notification records and UI affordances on the Documents tab, not the audit log on the History tab). Verify this by toggling the flag in the admin panel and confirming the History tab entries still appear when the flag is OFF.

  Note: if the History tab requires `ORDER_DOCUMENT_NOTIFICATIONS` to be ON for some reason discovered during manual verification, that would be a bug — raise it before closing the ticket.

  No commit is produced by this step.

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/app/orders-tracker/[category]/[orderId]/order-history/page.test.tsx` | Modify | Add `describe('OrderHistoryPage — maintenance order')` block with 8 test cases covering page render, hook call params, all three formatter render paths (added Order, removed Order, non-Order), loading state, and empty state; add new-orders regression guard |

No production source files require changes. This task modifies only the one test file above.

---

## Testing Strategy

### Automated tests (primary acceptance evidence)

The automated tests are the primary acceptance evidence for this task. Live staging audit data from maintenance orders is not available until D4 (MLID-2450 through MLID-2454) lands. The test suite is authoritative.

**order-history/page.test.tsx — maintenance order block:**

| Test case | Behavior verified |
|---|---|
| `useAuditLog` called with maintenance `entityId` | Hook receives correct orderId, not the new-order fixture |
| `useAuditLogFilterOptions` called with maintenance `entityId` | Same — filter options hook also receives the maintenance orderId |
| Page title renders | Page renders for maintenance category without crash |
| Order-category document-added entry | Formatter emits `A Document has been added (Document type = Order)` |
| Non-Order category entry (Referral Copy) | Formatter emits `New Document received (Document type = Referral Copy)` |
| Order-category document-removed entry | Formatter emits `A Document has been removed (Document type = Order)` |
| Loading state for maintenance orderId | Loading spinner appears while `isLoading: true` |
| Empty state for maintenance orderId | Empty-state message appears when `auditLogs: []` |
| Regression guard — new-orders entry still renders | Existing behavior unchanged when `orderId: 'order-123'` |

**Coverage gate:**
Run `npm run test -- --coverage order-history/page.test` from `apps/web/`. The `order-history/page.tsx` file must remain at or above the 80% threshold.

### Manual verification

Navigate to `/orders-tracker/maintenance/<maintenanceOrderId>/order-history` after seeding the two `auditLogs` documents described in Step 4. Confirm:

1. Both seeded entries render with the correct `renderChange` text.
2. Timestamps and pagination match the new-orders History tab.
3. The History tab renders regardless of the `ORDER_DOCUMENT_NOTIFICATIONS` feature flag state (the flag should not gate audit log display).

Full end-to-end verification with real maintenance audit data (produced by the system rather than seeded manually) is deferred until D4 lands.

---

## Security Considerations

- **Input validation**: No new API calls or inputs are introduced. The audit-log API route already performs `getServerSession` authentication.
- **Authorization**: `GET /api/audit-log` requires a valid session (any authenticated user). No category-based permission split exists or is needed. No changes to authorization logic.
- **Feature flag gating**: `ORDER_DOCUMENT_NOTIFICATIONS` gates notification records and Documents-tab affordances. It does not gate the audit log. The History tab renders for all authenticated users regardless of flag state — confirm this during manual verification.
- **PHI handling**: Audit log entries contain document category and source, not patient-identifiable fields. `actorId` is an internal user ID, not patient data. No PHI exposure risk in the log entries or test fixtures.

---

## Open Questions

None. The architecture is confirmed category-agnostic at every layer. The only contingency is: if manual verification reveals that the History tab is gated on `ORDER_DOCUMENT_NOTIFICATIONS` being ON, widen scope to remove that gate and add a test asserting the History tab renders when the flag is OFF.
