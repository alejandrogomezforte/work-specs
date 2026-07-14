# MLID-2448 — Documents Tab Notifications on Maintenance Order Detail

## Task Reference

- **Jira**: [MLID-2448](https://localinfusion.atlassian.net/browse/MLID-2448)
- **Epic**: [MLID-2417](https://localinfusion.atlassian.net/browse/MLID-2417) — epic plan at `docs/agomez/plans/MLID-2417-plan-progress.md`
- **Story Points**: 3
- **Branch**: `feature/MLID-2448-maintenance-documents-tab-notifications`
- **Base Branch**: `epic/MLID-2417-document-notifications-maintenance`
- **Status**: To Do
- **Sprint**: MLID Sprint 30 (active)

---

## Summary

The notification UI for the order-detail Documents tab (tab badge, per-row "New" chip, per-row "Mark as read" button, and "Mark all as read" bulk action) is fully implemented under the shared `[category]` dynamic route and is not gated on `category === 'new'` anywhere. Maintenance orders at `/orders-tracker/maintenance/[id]/documents` therefore already receive all of these affordances with no production code changes.

This task is verification plus maintenance-specific test coverage. Every existing test file pins `mockCategory = 'new'` (or hardcodes `category: 'new'` in `useParams`), leaving zero automated evidence that the code works for `category = 'maintenance'`. The deliverable is a set of maintenance-category test cases in `layout.test.tsx` and `documents/page.test.tsx` that close that gap.

---

## Codebase Analysis

### Route structure

All files live under `apps/web/app/orders-tracker/[category]/[orderId]/`. The single `[category]` segment resolves for `new`, `maintenance`, and `lead` alike. The real URL prefix is `/orders-tracker/`, not `/orders/` (Jira ticket text uses the wrong prefix).

### layout.tsx — tab badge

`useDocumentsTabNotifications(orderId)` is called with a single argument (the `assignedTo` parameter was removed in MLID-2391 / D0-T1). The badge renders when `isNotificationsEnabled && documentsUnreadCount > 0`. `isNotificationsEnabled` comes from `useFeatureFlag(FeatureFlag.ORDER_DOCUMENT_NOTIFICATIONS)`. The `category` value from `useParams` is used only for status editing and the edit-order chevron — it is never consulted for any notification decision.

### documents/page.tsx — chip, mark-as-read, mark-all-as-read

The "New" chip renders per row when `isNotificationsEnabled && isDocumentUnreadForUser(summary, userId)`. Per-row "Mark as read" fires `PATCH /api/notifications/${notificationId}/read`. "Mark all as read" fires `PATCH /api/notifications/bulk-read`. The only `orderCategory` check in this file is `showAddDocument = isManualUploadEnabled && orderCategory !== 'lead'`, which is unrelated to notifications.

### documents/useDocumentsTabNotifications.ts

Signature: `useDocumentsTabNotifications(orderId: string)`. Listens to SignalR events `documentAdded`, `documentUnlinked`, `notification:read`, and `order:reassigned`. Fetches `GET /api/notifications/by-order/${orderId}`. Never receives or reads `category`. Returns `{ unreadCount, byDocumentId, notificationIds, unreadNotificationIdsForCurrentUser, hasUnreadForCurrentUser, isLoading }`.

### documents/orderDocumentHelpers.ts

`isDocumentUnreadForUser(summary, userId)` returns `!!summary && !summary.isReadByCurrentUser && !!userId && summary.targetUserId === userId`. This is the strict rule — there is no `assignedTo` fallback. The Jira ticket text (`targetUserId === userId OR order.assignedTo === userId`) describes a superseded version of the helper that was updated in MLID-2391. The tests in this plan use the strict rule.

### API routes

All three routes (`GET /api/notifications/by-order/[orderId]/route.ts`, `PATCH /api/notifications/[id]/read/route.ts`, `PATCH /api/notifications/bulk-read/route.ts`) are session-authenticated and gated on `ORDER_DOCUMENT_NOTIFICATIONS` (returning 404 when off). None gate on order category.

### Existing test files

- `layout.test.tsx` — `mockCategory` variable is set to `'new'` in `beforeEach`; a `describe('Documents tab badge')` block covers badge-on, badge-off-count-zero, badge-off-flag-off but all run under `category = 'new'`. No maintenance badge test exists.
- `documents/page.test.tsx` — `useParams` is hardcoded to `{ category: 'new', orderId: 'order-1' }` via `jest.mock('next/navigation', ...)`. Covers "New" chip, actions column, mark-as-read, mark-all-as-read, and PDF viewer wiring — all for `category = 'new'`. No maintenance equivalent.
- `documents/useDocumentsTabNotifications.test.tsx` — hook tests; hook never receives category, so these already provide category-agnostic coverage of the data layer.
- `documents/orderDocumentHelpers.test.tsx` — pure unit tests of the strict predicate; already category-agnostic.

### Known double-fetch

`useDocumentsTabNotifications` is called from both `layout.tsx` (tab badge, consumes `unreadCount`) and `documents/page.tsx` (per-row state), producing two `GET /api/notifications/by-order/:orderId` requests on mount. There is no shared context provider today — a search for `DocumentsNotificationsContext` returns zero matches. The existing `NotificationsContext` is an unrelated toast context.

---

## Double-Fetch / Context-Provider Decision

The epic plan and the Jira ticket both mention lifting state to a Context provider to eliminate the double-fetch.

DECISION (2026-06-19): **out of scope for this task.** The double-fetch is a pre-existing performance characteristic introduced in MLID-2011 with the original new-orders implementation — it is not a bug and not a maintenance regression. It has run in production for new orders since notifications first shipped, with correct functional behavior. Reasons to keep it out:

1. The double-fetch pre-exists on new orders and is not a maintenance regression. Fixing it changes behavior for new orders too.
2. Introducing a Context provider touches `layout.tsx` and `documents/page.tsx` together — the same files that carry the "existing new-orders behavior is unchanged" criterion. Any mistake risks breaking the new-orders tests.
3. Every acceptance criterion in this task is satisfiable without the refactor.

If the redundant fetch is to be eliminated, it should be filed as a separate, dedicated performance ticket covering both new and maintenance orders.

---

## Implementation Steps

### Step 1 — Maintenance badge tests in layout.test.tsx (RED then verify green)

- **Files**: `apps/web/app/orders-tracker/[category]/[orderId]/layout.test.tsx`
- **What**: Add a `describe('Documents tab badge — maintenance category')` block alongside the existing `describe('Documents tab badge')` block. The new block sets `mockCategory = 'maintenance'` and `mockPathname = '/orders-tracker/maintenance/order-123'` in its `beforeEach`. Write three test cases:

  1. `'renders badge with count when unreadCount > 0 on a maintenance order'` — set `mockDocNotifications.unreadCount = 3`, render, assert `screen.getByTestId('badge-Documents')` has text content `'3'`.
  2. `'does not render badge when unreadCount is 0 on a maintenance order'` — set `mockDocNotifications.unreadCount = 0`, render, assert `screen.queryByTestId('badge-Documents')` is not in the document.
  3. `'does not render badge when notifications flag is off on a maintenance order'` — set `mockNotificationsFlag = false` and `mockDocNotifications.unreadCount = 5`, render, assert `screen.queryByTestId('badge-Documents')` is not in the document.

  Also add a `'existing new-orders badge behavior unchanged'` guard inside the existing `describe('Documents tab badge')` block that confirms the badge still renders with `mockCategory = 'new'` and `mockDocNotifications.unreadCount = 2`.

  Run `npm run test -- layout.test` from `apps/web/`. Because the implementation is already category-agnostic, all three maintenance tests are expected to pass immediately. A green result on first run IS the correct verification outcome — it is proof the affordance works for maintenance. If any test fails, that would indicate a category gate that needs to be removed; fix that before proceeding.

- **Commit**: `[MLID-2448] - test(orders): add maintenance-category Documents tab badge tests`

### Step 2 — Maintenance page tests in documents/page.test.tsx (RED then verify green)

- **Files**: `apps/web/app/orders-tracker/[category]/[orderId]/documents/page.test.tsx`
- **What**: The `jest.mock('next/navigation', ...)` at the top of this file hardcodes `category: 'new'`. Change the mock to use a mutable variable, following the same pattern as `layout.test.tsx`:

  ```typescript
  let mockCategory = 'new';
  jest.mock('next/navigation', () => ({
    useParams: () => ({ category: mockCategory, orderId: 'order-1' }),
  }));
  ```

  Add `mockCategory = 'new'` to the top-level `beforeEach` so existing tests are unaffected.

  Then add a `describe('OrderDocumentsPage — maintenance category')` block at the bottom of the file with its own `beforeEach` that sets `mockCategory = 'maintenance'`. Inside this block write the following test cases (each mirrors the existing new-orders tests):

  **New chip sub-describe:**

  1. `'shows New chip for unread document matching strict targetUserId on maintenance order'` — set `mockDocNotifications.byDocumentId` with `targetUserId: 'user-1'`, `isReadByCurrentUser: false`; render; assert `screen.getByTestId('new-chip')` has text `'New'`.
  2. `'does not show New chip when targetUserId does not match current user on maintenance order'` — set `targetUserId: 'user-other'`; assert no chip.
  3. `'does not show New chip when feature flag is off on maintenance order'` — set `mockNotificationsFlag = false` with `targetUserId: 'user-1'`; assert no chip.

  **Actions column sub-describe:**

  4. `'renders Mark as Read button for unread row matching current user on maintenance order'` — same setup as the new-orders equivalent; assert button is present.
  5. `'does not show Mark as Read when targetUserId does not match current user on maintenance order'` — `targetUserId: 'user-other'`; assert no button.
  6. `'calls PATCH /api/notifications/:id/read when Mark as Read is clicked on maintenance order'` — fire click; `waitFor` `mockFetch` called with `'/api/notifications/n1/read'` and `method: 'PATCH'`.

  **Mark all as Read sub-describe:**

  7. `'Mark all as Read button is present when flag is on for maintenance order'` — assert button in document.
  8. `'Mark all as Read button is disabled when hasUnreadForCurrentUser is false on maintenance order'` — assert disabled.
  9. `'Mark all as Read button is enabled when hasUnreadForCurrentUser is true on maintenance order'` — assert not disabled.
  10. `'clicking Mark all as Read calls PATCH /api/notifications/bulk-read on maintenance order'` — fire click; `waitFor` `mockFetch` called with `'/api/notifications/bulk-read'`, correct body, `method: 'PATCH'`.
  11. `'does not render Mark all as Read button when flag is off on maintenance order'` — `mockNotificationsFlag = false`; assert no button.

  **New-orders guard:**

  12. `'new-orders chip and mark-as-read behavior unchanged'` — inside the top-level `describe('OrderDocumentsPage')`, add one assertion block that explicitly sets `mockCategory = 'new'` and confirms a chip appears for `targetUserId: 'user-1'`.

  Run `npm run test -- documents/page.test` from `apps/web/`. As with Step 1, all tests are expected to pass immediately. Green on first run is correct. If a test fails, diagnose before committing.

- **Commit**: `[MLID-2448] - test(orders): add maintenance-category documents page notification tests`

### Step 3 — DROPPED (out of scope)

The Context-provider refactor to eliminate the double-fetch is **not part of this task** (resolved 2026-06-19 — see Open Questions). The double-fetch is a pre-existing performance characteristic from MLID-2011, not a bug and not a maintenance regression. If it is to be addressed, it should be a separate dedicated performance ticket covering both new and maintenance orders. This task is Steps 1 and 2 only.

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/app/orders-tracker/[category]/[orderId]/layout.test.tsx` | Modify | Add `describe('Documents tab badge — maintenance category')` block with 3 maintenance tests; add new-orders guard assertion |
| `apps/web/app/orders-tracker/[category]/[orderId]/documents/page.test.tsx` | Modify | Make `useParams` mock use a mutable `mockCategory` variable; add `describe('OrderDocumentsPage — maintenance category')` block with 12 test cases; add new-orders guard |
No production source files require changes. This task modifies only the two test files above.

---

## Testing Strategy

### Automated tests (primary acceptance evidence)

The automated tests are the primary evidence for this task. Live staging data from maintenance orders may not be available until D4 (MLID-2450, the notification-creation job for maintenance) lands. The test suite is authoritative.

**layout.test.tsx — maintenance badge block:**
- Badge renders with correct count when `unreadCount > 0` and flag ON, `category = 'maintenance'`
- Badge absent when `unreadCount = 0` and `category = 'maintenance'`
- Badge absent when flag OFF and `category = 'maintenance'`
- Guard: badge still renders correctly when `category = 'new'` (regression protection)

**documents/page.test.tsx — maintenance page block:**
- "New" chip present when `targetUserId === userId` (strict match), flag ON, `category = 'maintenance'`
- "New" chip absent when `targetUserId !== userId`, `category = 'maintenance'`
- "New" chip absent when flag OFF, `category = 'maintenance'`
- "Mark as Read" button present for unread row matching strict `targetUserId`, `category = 'maintenance'`
- "Mark as Read" absent when `targetUserId` does not match, `category = 'maintenance'`
- Click on "Mark as Read" calls `PATCH /api/notifications/:id/read`, `category = 'maintenance'`
- "Mark all as Read" button present when flag ON, `category = 'maintenance'`
- "Mark all as Read" disabled when `hasUnreadForCurrentUser = false`, `category = 'maintenance'`
- "Mark all as Read" enabled when `hasUnreadForCurrentUser = true`, `category = 'maintenance'`
- Click on "Mark all as Read" calls `PATCH /api/notifications/bulk-read` with correct body, `category = 'maintenance'`
- "Mark all as Read" absent when flag OFF, `category = 'maintenance'`
- Guard: chip and mark-as-read still work correctly when `category = 'new'` (regression protection)

**Coverage gate:**
Run `npm run test -- --coverage --collectCoverageFrom='app/orders-tracker/\[category\]/\[orderId\]/layout.tsx' --collectCoverageFrom='app/orders-tracker/\[category\]/\[orderId\]/documents/page.tsx'` from `apps/web/`. Both files must remain at or above the 80% threshold. Since no new branches are added to these files, coverage should stay where it is.

### Manual / staging verification

Full end-to-end verification on staging requires maintenance order notification data, which is produced by D4 (MLID-2450). Until D4 is deployed, manual verification requires seeding rows directly:

1. Insert a document notification record into the `notifications` collection targeting a maintenance order and a specific `targetUserId`.
2. Navigate to `/orders-tracker/maintenance/[that-orderId]/documents` while logged in as that user.
3. Confirm the badge count in the Documents tab label.
4. Confirm the "New" chip on the matching document row.
5. Click "Mark as Read" and confirm the chip disappears.
6. Click "Mark all as Read" and confirm all chips disappear.

This manual step is secondary. It can be deferred until D4 lands or done via seeded data — the automated tests are the acceptance gate for this task.

---

## Quality Checks

Run all of the following from `apps/web/` before marking the task done:

```bash
npm run test -- layout.test documents/page.test
npm run types:check
npm run lint
```

For coverage on the modified test files:
```bash
npm run test -- --coverage layout.test documents/page.test
```

---

## Security Considerations

- **Input validation**: No new API calls or inputs are introduced in Steps 1 and 2.
- **Authorization**: All three notification API routes enforce session authentication and gating on `ORDER_DOCUMENT_NOTIFICATIONS`. None gate on category. No changes to authorization logic.
- **Feature flag gating**: `ORDER_DOCUMENT_NOTIFICATIONS` gates all notification UI at both API (404 when off) and component levels. Tests verify both ON and OFF states for `category = 'maintenance'`.
- **PHI handling**: No PHI is exposed by notification read-state operations. The `targetUserId` is an internal user ID, not patient data.

---

## Open Questions

1. **Jira criteria vs. code reality on the `assignedTo` fallback** — RESOLVED 2026-06-19: follow the code reality. The strict `targetUserId === userId` rule in `isDocumentUnreadForUser` is intentional — the `OR order.assignedTo === userId` fallback was deliberately removed in MLID-2391 once retarget kept `targetUserId` always-live. The Jira ticket text is stale; do not re-introduce the fallback. No production code change to `orderDocumentHelpers.ts` or `useDocumentsTabNotifications.ts`. Tests assert the strict rule.

2. **Context-provider refactor** — RESOLVED 2026-06-19: out of scope for this task. The double-fetch (`useDocumentsTabNotifications` called from both `layout.tsx` and `documents/page.tsx`) is a pre-existing performance characteristic introduced in MLID-2011 with the original new-orders implementation — not a bug and not a maintenance regression. It has run in production for new orders since notifications first shipped with no functional problem. Step 3 is dropped from this task. If the redundant fetch is to be eliminated, it should be filed as a separate, dedicated performance ticket covering both new and maintenance orders. This task is Steps 1 and 2 only.
