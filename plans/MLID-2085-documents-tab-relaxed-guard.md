# MLID-2085 — Documents Tab: Relaxed UI + Server Guard for Reassigned Orders

## Task Reference

- **Jira**: [MLID-2085](https://localinfusion.atlassian.net/browse/MLID-2085)
- **Story Points**: 5
- **Branch**: `feature/MLID-2085-documents-tab-relaxed-guard`
- **Base Branch**: `develop`
- **Status**: In Progress (QA failed on staging)
- **Follow-up**: [MLID-2391](https://localinfusion.atlassian.net/browse/MLID-2391) — Cascade `targetUserId` on order reassignment (the data-side fix is deferred to next sprint; this ticket ships permanent UI+server guard relaxation that covers the QA acceptance criteria without the heavier cascade)

---

## Summary

QA found that the new Infusion Guide assigned to an order cannot see "New" chips, the per-row "Mark as Read" button, or the "Mark all as Read" bulk button. The root cause is a stale `targetUserId` on document notifications: the field is snapshotted from `order.assignedTo` at write-time and is never updated when the order is reassigned.

This ticket relaxes the gating predicate in two places — client and server — so the current order assignee is authorized regardless of who the notification's `targetUserId` names:

1. **Client-side guards** (`isDocumentUnreadForUser` in `page.tsx`, `unreadNotificationIdsForCurrentUser` in the hook): allow when `summary.targetUserId === userId` OR `order.assignedTo === userId`.
2. **Server-side guards** (`markUserNotificationAsRead`, `bulkMarkUserNotificationsAsRead`): if strict `targetUserId === userId` fails, the server reads the notification's `entities` to find its order id, loads the order, and allows when `order.assignedTo === userId`.

The server derives the order id from the notification itself, NOT from client input. The notification's `entities` array is the authoritative record of which order it belongs to. This keeps the API contract unchanged (no new request body fields) and rules out any "trust the client's orderId" weakness.

The long-term data-side fix (cascade `targetUserId` on reassignment) is deferred to **MLID-2391**. The OR-branches added here are kept after MLID-2391 ships, as belt-and-suspenders against races.

---

## Codebase Analysis

**Two UI surfaces need the OR-condition.** The `isDocumentUnreadForUser` predicate in `apps/web/app/orders-tracker/[category]/[orderId]/documents/page.tsx` (line 30-39) gates both the "New" chip (line 135-136) and the per-row "Mark as Read" button (line 167-169). Separately, `useDocumentsTabNotifications.ts` derives `unreadNotificationIdsForCurrentUser` (line 129-131) which drives `hasUnreadForCurrentUser` and thus the visibility of the "Mark all as Read" bulk button (page.tsx:202). Both predicates currently require strict `targetUserId === userId`.

**Tab badge `unreadCount` is intentionally global.** The `unreadCount` computation in the hook (line 126-128) counts all unread notifications without filtering by `userId`. This is by design (it is the badge visible on the Documents tab header regardless of who is viewing) and must NOT receive the OR-condition.

**Server-side authz gate.** `markUserNotificationAsRead` (`apps/web/services/mongodb/notification.ts` lines 328-363) uses `Notification.findOne({ _id, targetUserId: userId })` and returns `false` when the caller is not the target. `bulkMarkUserNotificationsAsRead` (lines 374-409) filters via `Notification.find({ _id: { $in }, targetUserId: userId })`. The doc-comment at lines 320-323 explains the strict check is there to prevent notification-existence probing. Without server-side relaxation, the UI patch alone makes the chip visible but the PATCH endpoints silently no-op, so QA criteria 2 and 3 remain broken.

**Source of truth for the order id.** Every document notification is created with `entities: [{ entityType: 'order', entityId: orderId }, { entityType: 'document', entityId: documentId }]` (`apps/web/services/notifications/documentNotification.ts` line 85-88). The server reads this entry directly when the strict authz check fails — no client input is trusted.

**Existence-probing protection preserved.** The fallback still only succeeds when (a) the notification exists, (b) it has an order entity, AND (c) the caller is the order's current `assignedTo`. A caller who is not the assignee of any order the notification belongs to gets the same `false` result they got before. Routes continue to return 404 to keep the response signature uniform.

**Order lookup primitive.** `getOrderById<K extends OrderCategory>(id, category, options?)` at `apps/web/services/mongodb/ordersTracker.ts:740` is the existing service function. Because the notification's `entities` does NOT carry the order's category, the fallback probes `'new'` first and `'maintenance'` second. Lead orders are skipped — the Documents tab is not surfaced for leads. Worst-case extra cost: one additional Mongo lookup per click when the order is a maintenance order.

**Hook signature change required.** `useDocumentsTabNotifications(orderId: string)` is called in two places: `page.tsx` line 52 and `layout.tsx` line 162. Both have `data?.assignedTo` available. The hook will gain a second parameter `orderAssignedTo: string | null`.

**Data shape on the client.** `OrderDetailsData` (`apps/web/types/orders.ts` lines 231-271) includes `assignedTo: string | null` at line 270 — already populated by `/api/orders/details`. No new client fetch is required.

---

## Implementation Steps

### Step 1 — Relax `isDocumentUnreadForUser` in page.tsx

- **Files**:
  - `apps/web/app/orders-tracker/[category]/[orderId]/documents/page.tsx`
  - `apps/web/app/orders-tracker/[category]/[orderId]/documents/page.test.tsx`

- **What**:

  RED — Add a test case to the existing "does not show New chip when targetUserId does not match current user" describe block:

  ```
  it('shows New chip when targetUserId does not match current user but order is assigned to current user')
  ```

  The test sets `byDocumentId['doc-1'].targetUserId = 'user-other'`, ensures `docsData.assignedTo = 'user-1'` (matching the mocked session userId `'user-1'`), renders `<OrderDocumentsPage />`, and asserts `screen.getByTestId('new-chip')` is present.

  Matching test for "Mark as Read" button:

  ```
  it('shows Mark as Read button when targetUserId does not match but order is assigned to current user')
  ```

  GREEN — Change the `isDocumentUnreadForUser` signature to:

  ```typescript
  function isDocumentUnreadForUser(
    summary: DocumentNotificationSummary | undefined,
    userId: string | undefined,
    orderAssignedTo: string | null | undefined
  ): boolean
  ```

  The condition becomes:

  ```typescript
  return (
    !!summary &&
    !summary.isReadByCurrentUser &&
    !!userId &&
    (summary.targetUserId === userId || orderAssignedTo === userId)
  );
  ```

  Pass `data?.assignedTo ?? null` as the third argument at both call sites inside `OrderDocumentsPage` (lines 136 and 168 after the change).

  REFACTOR — No structural cleanup needed.

- **Commit**: `[MLID-2085] - fix(documents): relax isDocumentUnreadForUser to include current assignee`

---

### Step 2 — Extend `useDocumentsTabNotifications` with `orderAssignedTo` parameter

- **Files**:
  - `apps/web/app/orders-tracker/[category]/[orderId]/documents/useDocumentsTabNotifications.ts`
  - `apps/web/app/orders-tracker/[category]/[orderId]/documents/useDocumentsTabNotifications.test.tsx`
  - `apps/web/app/orders-tracker/[category]/[orderId]/documents/page.tsx`
  - `apps/web/app/orders-tracker/[category]/[orderId]/layout.tsx`

- **What**:

  RED — Add two test cases to the `unreadNotificationIdsForCurrentUser` describe block in the hook test:

  ```
  it('unreadNotificationIdsForCurrentUser includes unread notification when orderAssignedTo matches userId even though targetUserId does not')
  ```

  The test passes `renderHook(() => useDocumentsTabNotifications('order-1', 'user-1'))` where the notification has `targetUserId: 'user-other'` and `isReadByCurrentUser: false`. Asserts `result.current.unreadNotificationIdsForCurrentUser` contains the notification id.

  ```
  it('unreadNotificationIdsForCurrentUser excludes already-read notification even when orderAssignedTo matches userId')
  ```

  Same setup but `isReadByCurrentUser: true`. Asserts the array is empty.

  GREEN — Update the hook signature:

  ```typescript
  export function useDocumentsTabNotifications(
    orderId: string,
    orderAssignedTo: string | null
  ): UseDocumentsTabNotificationsResult
  ```

  Update the `unreadNotificationIdsForCurrentUser` filter at line 129-131:

  ```typescript
  const unreadNotificationIdsForCurrentUser = notifications
    .filter(
      (n) =>
        !n.isReadByCurrentUser &&
        (n.targetUserId === userId || orderAssignedTo === userId)
    )
    .map((n) => n._id);
  ```

  The `unreadCount` computation (line 126-128) is unchanged.

  Update the call site in `page.tsx` line 52:

  ```typescript
  const {
    byDocumentId,
    hasUnreadForCurrentUser,
    unreadNotificationIdsForCurrentUser,
  } = useDocumentsTabNotifications(data?.id ?? '', data?.assignedTo ?? null);
  ```

  Update the call site in `layout.tsx` line 162:

  ```typescript
  const { unreadCount: documentsUnreadCount } = useDocumentsTabNotifications(
    orderId ?? '',
    data?.assignedTo ?? null
  );
  ```

  REFACTOR — Existing tests that call `useDocumentsTabNotifications('order-1')` must be updated to pass a second argument (`null` is fine for old cases). Update all such call sites in the hook test file.

- **Commit**: `[MLID-2085] - fix(documents): extend useDocumentsTabNotifications with orderAssignedTo for unread filter`

---

### Step 3 — Relax server-side authz in `markUserNotificationAsRead` and `bulkMarkUserNotificationsAsRead`

- **Files**:
  - `apps/web/services/mongodb/notification.ts`
  - `apps/web/services/mongodb/notification.test.ts` (create or extend)

- **What**:

  No new parameters and no API contract change. The server reads the notification's `entities` to find the order id, then looks up the order to verify assignee.

  RED — Add tests in a `markUserNotificationAsRead — fallback authz via order assignee` describe block:

  - `returns true and writes NotificationRead when strict mismatch but caller is the current assignee of the notification's order (new orders)`
    Mock strict `findOne` (with `targetUserId` filter) to return null. Mock the full `findById` (notification lookup) to return a notification whose entities include `{ entityType: 'order', entityId: 'order-1' }`. Mock `getOrderById('order-1', 'new')` to return `{ assignedTo: 'user-1' }`. Assert `NotificationRead.create` was called and the function returned `true`.
  - `falls back to maintenance category when the order is not found under 'new'`
    Mock `getOrderById('order-1', 'new')` to return null. Mock `getOrderById('order-1', 'maintenance')` to return `{ assignedTo: 'user-1' }`. Assert the function returned `true`.
  - `returns false when caller is not the order's current assignee`
    Mock both category lookups to return orders whose `assignedTo !== 'user-1'`. Assert `false` and no write occurred.
  - `returns false when the notification has no order entity`
    Mock the notification to have only a `{ entityType: 'document', entityId: ... }` entry. Assert `false`.
  - `returns false when the notification does not exist`
    Mock `findById` to return null. Assert `false` and no order lookup occurred.
  - `does not call getOrderById when strict check succeeds (no wasted DB call)`
    Mock the strict `findOne` to return the notification. Assert `getOrderById` is never called.

  And in a `bulkMarkUserNotificationsAsRead — fallback authz via order assignee` describe block:

  - `returns union of strict-owned and fallback-owned ids`
    Three notifications: notif-A has `targetUserId = userId` (strict-owned), notif-B has `targetUserId = other` but its order entity points to an order the caller is assigned to (fallback-owned), notif-C has `targetUserId = other` and points to an order the caller is NOT assigned to (denied). Assert return value is `[notif-A, notif-B]` and `NotificationRead.insertMany` was called with both.
  - `dedupes order lookups when multiple notifications point at the same order`
    Two non-strict notifications, both pointing at the same `order-1`. Assert `getOrderById` is called at most once per (orderId, category) pair, not once per notification.
  - `skips fallback entirely when all supplied ids are strict-owned`
    Mock strict find to return all ids. Assert no notification-fetch and no order lookup happen.

  GREEN — Implementation outline for `markUserNotificationAsRead`:

  1. Existing strict check: `Notification.findOne({ _id: notificationId, targetUserId: userId }).lean()`. If found, proceed to write `NotificationRead` (existing path, including duplicate-key idempotency).
  2. If not found, attempt fallback:
     a. `notification = await Notification.findById(notificationId).lean()`. If null, return `false`.
     b. Find the order entity: `orderEntity = notification.entities?.find((e) => e.entityType === 'order')`. If absent, return `false`.
     c. Look up the order: `order = await getOrderById(orderEntity.entityId, 'new') ?? await getOrderById(orderEntity.entityId, 'maintenance')`. If null, return `false`.
     d. If `order.assignedTo !== userId`, return `false`.
     e. Write `NotificationRead` (same as path 1).
  3. Return `true`.

  Implementation outline for `bulkMarkUserNotificationsAsRead`:

  1. Existing strict filter: `Notification.find({ _id: { $in: notificationIds }, targetUserId: userId }).lean()`. Collect `strictIds`.
  2. `remaining = notificationIds.filter((id) => !strictIds.includes(id))`. If empty, jump to step 4 with `fallbackIds = []`.
  3. Fallback for remaining:
     a. `remainingNotifs = await Notification.find({ _id: { $in: remaining } }).lean()`.
     b. Build a unique set of `orderId`s from each notification's order entity.
     c. For each unique `orderId`, look up the order via `getOrderById(orderId, 'new')` then `'maintenance'`. Cache the result keyed by `orderId`.
     d. `fallbackIds = remainingNotifs.filter((n) => { const oid = n.entities.find(e => e.entityType==='order')?.entityId; const order = lookupCache[oid]; return order?.assignedTo === userId; }).map((n) => String(n._id))`.
  4. `allOwnedIds = [...strictIds, ...fallbackIds]`. If empty, return `[]`.
  5. Write `NotificationRead.insertMany` for `allOwnedIds` (existing path: `ordered: false`, duplicate-key tolerant).
  6. Return `allOwnedIds`.

  Per memory rule: do NOT add new `.select(...)` projections — use `.lean()` and read fields normally. Existing `.select('_id')` in the file is left as-is for legacy consistency.

  REFACTOR — Extract a private helper `isCallerOrderAssignee(notification, userId)` that encapsulates the entity-lookup + `getOrderById` probe so both functions share it.

- **Commit**: `[MLID-2085] - fix(notifications): allow current order assignee to mark notifications as read`

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/app/orders-tracker/[category]/[orderId]/documents/page.tsx` | Modify | Add `orderAssignedTo` parameter to `isDocumentUnreadForUser`; pass `data?.assignedTo` at both call sites; pass to hook |
| `apps/web/app/orders-tracker/[category]/[orderId]/documents/page.test.tsx` | Modify | Add OR-branch test cases for New chip and Mark as Read button |
| `apps/web/app/orders-tracker/[category]/[orderId]/documents/useDocumentsTabNotifications.ts` | Modify | Add `orderAssignedTo: string \| null` parameter; apply OR-condition to `unreadNotificationIdsForCurrentUser` filter |
| `apps/web/app/orders-tracker/[category]/[orderId]/documents/useDocumentsTabNotifications.test.tsx` | Modify | Add OR-branch test cases; update existing `renderHook` calls to pass second `null` argument |
| `apps/web/app/orders-tracker/[category]/[orderId]/layout.tsx` | Modify | Pass `data?.assignedTo ?? null` as second argument to `useDocumentsTabNotifications` |
| `apps/web/services/mongodb/notification.ts` | Modify | Add fallback authz in `markUserNotificationAsRead` and `bulkMarkUserNotificationsAsRead`: load the notification, derive orderId from `entities`, probe `getOrderById` ('new' then 'maintenance'), allow when caller matches `order.assignedTo` |
| `apps/web/services/mongodb/notification.test.ts` | Create or Modify | Unit tests for fallback authz on both single and bulk |

---

## Testing Strategy

**Unit tests — `isDocumentUnreadForUser`:**
- Returns `false` when `targetUserId !== userId` AND `orderAssignedTo !== userId`.
- Returns `true` when `targetUserId !== userId` BUT `orderAssignedTo === userId` and the notification is unread.
- Returns `false` when `isReadByCurrentUser` is true even if `orderAssignedTo === userId`.

**Unit tests — `useDocumentsTabNotifications`:**
- `unreadNotificationIdsForCurrentUser` includes notifications where `targetUserId !== userId` but `orderAssignedTo === userId` and unread.
- `unreadNotificationIdsForCurrentUser` excludes already-read notifications even when `orderAssignedTo === userId`.
- `unreadCount` is unaffected by the new parameter.

**Unit tests — `markUserNotificationAsRead`:**
- Strict-owned path: behaves exactly as before when `targetUserId === userId` (no order lookup).
- Fallback path: `true` + write when caller is the assignee of the notification's order (new or maintenance).
- Falls back from 'new' to 'maintenance' when the order isn't found under 'new'.
- Returns `false` when the caller is not the order's current assignee.
- Returns `false` when the notification has no order entity.
- Returns `false` when the notification doesn't exist.

**Unit tests — `bulkMarkUserNotificationsAsRead`:**
- Returns the union of strict-owned and fallback-owned ids.
- Dedupes `getOrderById` calls when multiple notifications point at the same order.
- Skips fallback path entirely when all supplied ids are strict-owned.
- Returns only strict-owned ids when the order's assignee doesn't match.

**Manual verification on staging (post-deploy):**
1. Log in as the current assignee (alejandro).
2. Navigate to order `6a073015c3a15ed77b2e2574` (18 stale notifications in Mongo).
3. Confirm "New" chips and "Mark as Read" buttons are visible on the document rows.
4. Click "Mark as Read" on one row: confirm the chip and button disappear and the badge decrements (the click now actually persists a `NotificationRead` via the fallback authz).
5. Click "Mark all as Read": confirm all remaining chips disappear and the bulk button becomes disabled.
6. Reload the page: confirm reads persist (badge stays at zero).

**Out of scope — covered by MLID-2391:**
- Cascade that retargets `targetUserId` on reassignment.
- SignalR `documentAdded` broadcast on reassignment.
- Backfill script for stale notifications.

---

## Security Considerations

**Source of truth.** The server derives the order id from the notification's `entities` field, never from client input. There is no new request body schema. A caller cannot trick the server into authorizing reads for notifications they aren't entitled to by lying in the body — there is no body to lie about.

**Existence-probing protection preserved.** The fallback succeeds only when (a) the notification exists, (b) it has an order entity, AND (c) the caller is the order's current `assignedTo`. A caller who is not the assignee of any order the notification belongs to gets the same `false` result they got before. Routes continue to return 404 on every authorization failure to keep the response signature uniform.

**Authorization invariants.**
- The session userId is sourced from `getServerSession(authOptions)` — clients cannot forge it.
- `getOrderById` is the only source of truth for `order.assignedTo`.
- The strict path remains the primary check; the fallback only runs when the strict path fails.

**PHI handling.** No patient data flows through the new code paths. Only `notificationId`, `orderId`, and `userId` strings are involved.

**Race conditions.** The UI and server fallbacks both consult the same source of truth (`order.assignedTo`), so they cannot diverge. If a reassignment happens between the UI check and the server check, the server's later read wins — there is no exploitable inconsistency.

---

## Open Questions

None. The two questions about lead-order cascade coverage and a backfill script for legacy stale notifications belong to **MLID-2391** and are documented in its plan file.
