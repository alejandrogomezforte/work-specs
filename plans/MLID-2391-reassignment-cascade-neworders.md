# MLID-2391 — [D0-T1] Reassignment cascades notifications to new assignee (neworders)

## Task Reference

- **Jira**: [MLID-2391](https://localinfusion.atlassian.net/browse/MLID-2391)
- **Parent Epic (tracking only)**: [MLID-2417](https://localinfusion.atlassian.net/browse/MLID-2417) — Orders: Document Notifications V2 — Maintenance & Order Linking
- **Story Points**: 3
- **Branch**: `feature/MLID-2391-reassignment-cascade-neworders`
- **Base Branch**: `develop` — D0 is a hotfix; per the epic delivery strategy, D0 tickets branch directly off `develop`, not off the epic branch
- **Status**: Done

> The Jira ticket status currently shows "IN QA" from a bot mis-transition (see author comment 2026-05-15). Work has not started. Do not retransition; the team policy is to leave Jira movement to QA/PO.

> **Approach change (2026-05-25).** Earlier revisions of this plan tried a "delete previous-user notifications and recreate for the new assignee" model. That model has two real problems: (a) the recreate path reuses `notifyOrderOfRecentDocuments`, which only looks back 30 days and would silently lose notifications for older orders; (b) it contradicts the existing team-shared-inbox product decision (the inline comment on the entity-count routes records the 2026-04-14 product call that badges are globally visible). The plan now uses an in-place retarget plus a read-predicate change that satisfies the PO requirement (new assignee sees everything as unread) without any of the recreate-path side effects. Three downstream simplifications drop out as part of the correctness fix.

---

## Summary

When a new order is reassigned via `PUT /api/orders/new`, document notifications targeting the previous assignee are not touched. `targetUserId` stays stale; the new assignee fails the strict `targetUserId === userId` predicate that gates "New" chips, "Mark as Read" buttons, and "Mark all as Read"; the previous assignee continues to see stale chips for an order they no longer own. MLID-2085 papered over the symptom by relaxing the read-side predicate (`targetUserId === userId || order.assignedTo === userId`) — this ticket closes the gap at the data layer and removes the workaround.

Three coordinated changes:

1. **Retarget**: a fire-and-forget cascade triggered from the route handler runs `Notification.updateMany({...prev-user-records...}, { $set: { targetUserId: newAssignee } })` for every order id returned by `updateOrder` (covers protocol-partner `-1` orders).
2. **Flip the read predicate**: the entity-count routes (`/api/notifications/counts`, `/api/notifications/by-order/:orderId`) currently use `userReadScope: 'any'` — a `notificationreads` row from any user clears the badge. With retarget in place, switch the predicate to "the **current target** must have the read receipt" (`notificationreads.userId === notification.targetUserId`). Past read receipts from prior assignees stay in the table for audit but no longer participate in the unread count. Net effect: the new assignee sees everything as unread; old receipts remain queryable.
3. **Strip the relaxed-predicate workarounds**: the `isCallerOrderAssignee` server-side fallback in `markUserNotificationAsRead` / `bulkMarkUserNotificationsAsRead`, and the `orderAssignedTo === userId` clause in the client chip filter, both become incorrect once `targetUserId` is always live. Remove them — only the user matching `targetUserId` can see and click "Mark as Read." This is part of the correctness fix, not a follow-up — leaving them in place would cause the client chip filter to over-show and the server auth to over-authorize.

A new SignalR event `order:reassigned { orderId }` triggers live refetch for both the previous and new assignee's open clients.

Un-assignment is **not in scope** — the UI cannot clear `assignedTo` on `PUT /api/orders/new`.

The retarget helper is exposed as a collection-agnostic orchestrator `handleOrderReassignment(order, previousAssignedTo, newAssignedTo)`; D4-T5 (MLID-2454) reuses it for `maintenanceorders` without modification.

---

## Codebase Analysis

### PUT `/api/orders/new` route handler

File: `apps/web/app/api/orders/new/route.ts:179-380`.

Relevant flow:

1. **Line 188-198** — `assignedToFields` is built only when `body.assignedTo` is truthy and `isAllowedToEditOrderAssignedTo(session.user.userObj, 'new')` passes. The shape is `{ assignedTo, lastAssigned, assignedBy }`. Because of the truthy guard, this route cannot clear an assignment — the un-assignment case has no code path.
2. **Line 199-202** — `currentOrder` is fetched via `db.collection(COLLECTION.NewOrders).findOne({ displayId })` before the update. This gives `currentOrder.assignedTo` (the previous assignee) at decision time.
3. **Line 353-360** — `updateOrder({ displayId, fieldsToUpdate, updateLinkedOrders }, 'new')` writes inside a Mongo transaction and returns `FullOrder<'new'>[]` for every modified document, including the protocol-partner (`-1`) order when `updateLinkedOrders === true`. Each item is enriched via the standard orders aggregate — `_id` and `assignedTo` are guaranteed present.
4. **Line 367-377** — `auditOrderUpdateWithOptionalSystemRtsDate` is awaited.
5. **Line 379** — `return NextResponse.json(result)`.

The cascade trigger goes between steps 4 and 5, fire-and-forget.

### `updateOrder` return shape

File: `apps/web/services/mongodb/ordersTracker.ts:1035-1100`. Inside a transaction, updates the primary `displayId`, then the partner `displayId` from `getOrderProtocolPartnerDisplayId(displayId)` (`apps/web/types/orders.ts:27-31` — `"XYZ"` ↔ `"XYZ-1"`), then aggregates both `displayId`s. Every `_id` in the returned array got the new `assignedTo`.

### Current read-side architecture (what we are simplifying)

Three retrofits exist today to compensate for stale `targetUserId`:

- **`getCountForEntity` with `userReadScope: 'any'`** (`apps/web/services/mongodb/notification.ts:168-206`) — `buildUserUnreadLookup('any')` joins `notificationreads` by `notificationId` alone, so any reader's receipt clears the unread count. Both consumer routes call it this way: `apps/web/app/api/notifications/counts/route.ts:84` and `apps/web/app/api/notifications/by-order/[orderId]/route.ts:97`.
- **`isCallerOrderAssignee` fallback** in `markUserNotificationAsRead` and `bulkMarkUserNotificationsAsRead` (`apps/web/services/mongodb/notification.ts:356-385`, used at `:418` and `:488`) — when strict `targetUserId === userId` fails, falls back to "caller is the order's current `assignedTo`."
- **Client chip filter relaxed clause** (`apps/web/app/orders-tracker/[category]/[orderId]/documents/useDocumentsTabNotifications.ts:136-142`) — `!isReadByCurrentUser && (targetUserId === userId || orderAssignedTo === userId)`.

All three exist to grant the new assignee read-side rights on records whose `targetUserId` still names the previous assignee. With retarget in place, `targetUserId` is always live and all three become incorrect (the client would over-show; the server would over-authorize) or dead (the `'any'` `userReadScope` would let any prior reader's receipt suppress the new assignee's chip).

The `'target'` branch of `buildUserUnreadLookup` already implements the predicate this plan needs:

```javascript
{
  $match: {
    $expr: {
      $and: [
        { $eq: ['$notificationId', '$$notifId'] },
        { $eq: ['$userId', '$$targetUserId'] },
      ],
    },
  },
}
```

Switching the two consumer routes to `'target'` makes the `'any'` branch dead code — it has no other call sites in the repo.

### `notificationreads` rows from prior assignees

Per the rule above, prior-assignee `notificationreads` rows are retained. They no longer participate in the unread join (their `userId` does not match the new `targetUserId`), but they remain queryable for "did user A read notification X at time T?" audits. No cleanup job is required.

### Per-row `isReadByCurrentUser`

`apps/web/app/api/notifications/by-order/[orderId]/route.ts:82-87` computes `isReadByCurrentUser` via `isReadByUser(notif._id, userId)` — a per-current-viewer check, independent of `targetUserId`. This field stays as-is: under the stripped chip filter, it only matters when the viewer is the target (because `targetUserId === userId` is required for the chip to render), and in that case "current viewer" and "current target" are the same person.

### Existing SignalR client surfaces that must react to reassignment

Two hooks consume the affected APIs and must refetch when an order is reassigned:

- `apps/web/app/orders-tracker/[category]/[orderId]/documents/useDocumentsTabNotifications.ts` — Documents-tab badge + chips
- `apps/web/app/orders-tracker/hooks/useOrderNotificationCounts.ts` — orders-tracker table row badge

Both currently listen for `documentAdded` and `notification:read`. Add an `order:reassigned` listener (refetch handler is the same — just call the existing `fetchNotifications` / equivalent).

### Test patterns observed

- **Route tests** (`apps/web/app/api/orders/new/route.test.ts:767+`) mock dependencies with `jest.mock(...)` at module top, type-cast with `as jest.MockedFunction<...>`, stub the session with `mockGetServerSession.mockResolvedValue(mockSession)`, and stub `connectToDatabase` to return a `findOne`-capable collection. Existing PUT cases at `:782-836` are the closest templates.
- **Service tests** (`apps/web/services/mongodb/notification.test.ts`) mock the Mongoose `Notification` model at the module level with `jest.mock('@/models/Notification', () => ({...}))`. This is the template for the new helper's unit tests.
- **Documents-tab hook tests** (`apps/web/app/orders-tracker/[category]/[orderId]/documents/useDocumentsTabNotifications.test.tsx`) cover the existing event listeners.

### Branch state

- Local `develop` and `origin/develop` are at the same commit (`759fc8829`).
- No existing local branch matches `*MLID-2391*`.

### Out of scope (explicit)

- **Maintenance-order PUT route** (`apps/web/app/api/orders/maintenance/route.ts`) — D4-T5 (MLID-2454). The helper is built to be reusable there without modification.
- **Un-assignment** — the UI cannot clear `assignedTo` on `PUT /api/orders/new`, so there is no code path to handle.

---

## Implementation Steps

Each step is one TDD cycle (RED → GREEN → REFACTOR) and one commit. All commits follow `[MLID-2391] - type(scope): description`.

Order matters: steps 1-3 add the retarget and trigger; steps 4-6 strip the workarounds. The workaround removals are sequenced after the retarget so that at no intermediate commit does the read-side over-show or over-authorize.

### Step 1 — Add `reassignUserNotificationsForOrder` service helper

- **Files**:
  - `apps/web/services/mongodb/notification.ts`
  - `apps/web/services/mongodb/notification.test.ts`

- **What**:

  **RED** — Add a `reassignUserNotificationsForOrder` describe block. Cases:

  - `returns 0 and does not call the database when orderId is empty`
  - `returns 0 and does not call the database when fromUserId is empty`
  - `returns 0 and does not call the database when toUserId is empty`
  - `calls Notification.updateMany with the exact retarget filter and $set, returns modifiedCount`
    - Arrange: `Notification.updateMany` mocked to resolve `{ modifiedCount: 4, matchedCount: 4 }`.
    - Assert the filter is exactly `{ type: 'user', targetUserId: 'user-old', entities: { $elemMatch: { entityType: 'order', entityId: 'order-1' } } }` and the update is exactly `{ $set: { targetUserId: 'user-new' } }`.
    - Assert the return value is `4`.
  - `re-throws when Notification.updateMany rejects` (orchestration helper above swallows; this helper logs and re-throws, mirroring `createNotification` at `notification.ts:48-64`)

  **GREEN** — In `notification.ts`, export:

  ```typescript
  export const reassignUserNotificationsForOrder = async (
    orderId: string,
    fromUserId: string,
    toUserId: string
  ): Promise<number> => {
    if (!orderId || !fromUserId || !toUserId) {
      return 0;
    }
    logger.info(
      'Notification Service: Reassigning user notifications for order',
      { orderId, fromUserId, toUserId }
    );
    try {
      const result = await Notification.updateMany(
        {
          type: 'user',
          targetUserId: fromUserId,
          entities: {
            $elemMatch: { entityType: 'order', entityId: orderId },
          },
        },
        { $set: { targetUserId: toUserId } }
      );
      return result.modifiedCount ?? 0;
    } catch (error) {
      logger.error(
        'Notification Service: Error reassigning user notifications for order',
        { error, orderId, fromUserId, toUserId }
      );
      throw error;
    }
  };
  ```

  **REFACTOR** — None expected.

- **Commit**: `[MLID-2391] - feat(notifications): add reassignUserNotificationsForOrder service helper`

---

### Step 2 — Add `handleOrderReassignment` orchestration helper

- **Files**:
  - `apps/web/services/notifications/handleOrderReassignment.ts` (create)
  - `apps/web/services/notifications/handleOrderReassignment.test.ts` (create)

- **What**:

  **RED** — `/** @jest-environment node */`. Mock `FeatureFlagService.getFeatureFlag`, `reassignUserNotificationsForOrder`, `getSignalRService`, and `logger`. Cases:

  - `feature flag off — returns without retargeting or broadcasting`
  - `previousAssignedTo empty — returns without retargeting or broadcasting` (defensive — the route guards against this, but the helper protects future callers)
  - `newAssignedTo empty — returns without retargeting or broadcasting` (defensive — un-assignment is out of scope)
  - `previousAssignedTo === newAssignedTo — returns without retargeting or broadcasting` (no-op)
  - `reassignment — calls retarget with (orderId, prev, next), then broadcasts order:reassigned`
    - Arrange `order = { _id: 'order-1', assignedTo: 'user-new' }`, `previousAssignedTo = 'user-old'`, `newAssignedTo = 'user-new'`.
    - Assert `reassignUserNotificationsForOrder('order-1', 'user-old', 'user-new')` called.
    - Assert `signalr.broadcastToAll('order:reassigned', { orderId: 'order-1' })` called after the retarget resolves (mock invocation order).
  - `retarget rejects — does not throw, logs error, does not broadcast`
  - `signalr unavailable — does not throw, logs error, retarget still ran`
  - `broadcast rejects — does not throw, logs error`

  **GREEN** — Create `handleOrderReassignment.ts`:

  ```typescript
  import { FeatureFlagService } from '@/services/featureFlags/featureFlagService';
  import { reassignUserNotificationsForOrder } from '@/services/mongodb/notification';
  import { getSignalRService } from '@/services/signalr';
  import { FeatureFlag } from '@/utils/featureFlags/featureFlags';
  import { logger } from '@/utils/logger';

  export interface ReassignableOrder {
    _id: unknown;
    assignedTo?: string | null;
  }

  /**
   * Fire-and-forget cascade for order reassignment. Retargets every
   * notification scoped to (order, previousAssignedTo) onto newAssignedTo,
   * then broadcasts an order:reassigned SignalR event so open clients
   * refetch. Never throws.
   *
   * D4-T5 (MLID-2454) reuses this helper for maintenance orders without
   * modification — the helper does not branch on collection.
   */
  export async function handleOrderReassignment(
    order: ReassignableOrder,
    previousAssignedTo: string,
    newAssignedTo: string
  ): Promise<void> {
    try {
      const isEnabled = await FeatureFlagService.getFeatureFlag(
        FeatureFlag.ORDER_DOCUMENT_NOTIFICATIONS
      );
      if (!isEnabled) {
        return;
      }
      if (!previousAssignedTo || !newAssignedTo) {
        return;
      }
      if (previousAssignedTo === newAssignedTo) {
        return;
      }

      const orderId = String(order._id);

      try {
        await reassignUserNotificationsForOrder(
          orderId,
          previousAssignedTo,
          newAssignedTo
        );
      } catch (error) {
        logger.error(
          'handleOrderReassignment: failed to retarget notifications — skipping broadcast',
          { error, orderId, previousAssignedTo, newAssignedTo }
        );
        return;
      }

      try {
        const signalr = await getSignalRService();
        await signalr.broadcastToAll('order:reassigned', { orderId });
      } catch (error) {
        logger.error(
          'handleOrderReassignment: failed to broadcast order:reassigned — notifications were retargeted',
          { error, orderId }
        );
      }
    } catch (error) {
      logger.error('handleOrderReassignment: unexpected error', {
        error,
        orderId: String(order._id),
        previousAssignedTo,
        newAssignedTo,
      });
    }
  }
  ```

  **REFACTOR** — None expected. Skipping the broadcast when the retarget fails is intentional: a failure leaves the records in their pre-cascade state, and emitting `order:reassigned` would cause clients to refetch a state that does not reflect the broadcast.

- **Commit**: `[MLID-2391] - feat(notifications): add handleOrderReassignment helper`

---

### Step 3 — Wire the cascade into `PUT /api/orders/new`

- **Files**:
  - `apps/web/app/api/orders/new/route.ts`
  - `apps/web/app/api/orders/new/route.test.ts`

- **What**:

  **RED** — Add `jest.mock('@/services/notifications/handleOrderReassignment', () => ({ handleOrderReassignment: jest.fn().mockResolvedValue(undefined) }))` to the module mocks. Add cases to the `describe('PUT /api/orders/new')` block at `route.test.ts:767`:

  - `triggers handleOrderReassignment for each updated order when assignedTo changes`
    - Arrange `mockFindOne` to return `{ _id: 'order-abc', displayId: 'N-0042', assignedTo: 'user-old' }`. `mockUpdateNewOrder` returns `[{ _id: 'order-abc', assignedTo: 'user-new' }, { _id: 'order-partner', assignedTo: 'user-new' }]`.
    - Body: `{ displayId: 'N-0042', received: '...', assignedTo: 'user-new' }`.
    - Assert `handleOrderReassignment` called twice — once per item in `result` — with `(updatedOrder, 'user-old', 'user-new')`.
  - `does not trigger when body.assignedTo is absent`
  - `does not trigger when assignedTo is unchanged` (`currentOrder.assignedTo === body.assignedTo`)
  - `does not trigger when previous assignedTo was empty` (first-time assignment is handled by the existing `notifyOrderOfRecentDocuments` call at line 154-159, not by reassignment)
  - `does not trigger when the editor lacks permission` (session whose `userObj.positions` fails `isAllowedToEditOrderAssignedTo` — `assignedToFields` stays `{}`, condition is false)
  - `response is 200 even when handleOrderReassignment rejects` (fire-and-forget guarantee)

  **GREEN** — In `route.ts` PUT handler, between the audit await at `:367-377` and `return NextResponse.json(result)` at `:379`:

  ```typescript
  const previousAssignedTo =
    typeof currentOrder?.assignedTo === 'string' && currentOrder.assignedTo
      ? currentOrder.assignedTo
      : null;
  const reassignedTo =
    'assignedTo' in assignedToFields
      ? (assignedToFields as { assignedTo: string }).assignedTo
      : null;

  if (
    result &&
    previousAssignedTo &&
    reassignedTo &&
    reassignedTo !== previousAssignedTo
  ) {
    for (const updatedOrder of result) {
      handleOrderReassignment(
        updatedOrder,
        previousAssignedTo,
        reassignedTo
      ).catch(() => {
        // Belt-and-suspenders: handleOrderReassignment is no-throw by
        // contract; this catch guarantees the route response is never
        // delayed or surfaced as 500 if that contract is violated.
      });
    }
  }
  ```

  Add `import { handleOrderReassignment } from '@/services/notifications/handleOrderReassignment';`.

  **REFACTOR** — None. The block is small enough to live inline.

- **Commit**: `[MLID-2391] - feat(orders/new): trigger reassignment notification cascade fire-and-forget`

---

### Step 4 — Flip badge predicate from `'any'` to current-target join

The next three steps drop the workarounds the retarget makes obsolete. Each step is its own commit; the order is intentional — the read predicate flips first so that the unread count goes from "team-wide" to "current-target" before the mark-as-read workarounds are removed.

- **Files**:
  - `apps/web/services/mongodb/notification.ts`
  - `apps/web/services/mongodb/notification.test.ts`
  - `apps/web/app/api/notifications/counts/route.ts`
  - `apps/web/app/api/notifications/counts/route.test.ts`
  - `apps/web/app/api/notifications/by-order/[orderId]/route.ts`
  - `apps/web/app/api/notifications/by-order/[orderId]/route.test.ts`

- **What**:

  **RED** — Update existing test cases that asserted the `'any'`-mode behavior to instead assert the current-target join:

  - `getCountForEntity` no longer accepts `userReadScope`; counts treat a notification as "read" only when its current `targetUserId` has a `notificationreads` row for it.
  - Add a regression case: a `notificationreads` row whose `userId` does not match the notification's current `targetUserId` does **not** clear the count.
  - Counts route returns the new predicate's counts.
  - By-order route returns the new predicate's `unreadCount`.

  **GREEN** —

  In `notification.ts`:
  - Remove the `userReadScope` parameter from `getCountForEntity`.
  - Remove the `'any'` branch from `buildUserUnreadLookup`; it has no remaining call site. Inline the `'target'` branch as the only behavior, or simplify to `buildUserUnreadLookup()`.
  - In the counts route, remove `userReadScope: 'any'` from the `getCountForEntity` call.
  - In the by-order route, remove `userReadScope: 'any'` from the `getCountForEntity` call.

  **REFACTOR** — `buildUserUnreadLookup` simplifies from "switch on scope" to a single literal lookup object — fold it into the call site if that reads more cleanly than a one-line helper.

- **Commit**: `[MLID-2391] - refactor(notifications): unread count joins on current targetUserId (remove userReadScope:any)`

---

### Step 5 — Remove `isCallerOrderAssignee` fallback from mark-as-read paths

- **Files**:
  - `apps/web/services/mongodb/notification.ts`
  - `apps/web/services/mongodb/notification.test.ts`

- **What**:

  **RED** — Update existing tests for `markUserNotificationAsRead` and `bulkMarkUserNotificationsAsRead`:

  - Cases that exercised the fallback ("caller is order's assignedTo but not targetUserId, mark succeeds") should now fail. Update them to assert the request is rejected (returns `false` / not in `markedIds`).
  - Cases asserting strict-target-match continue to pass unchanged.
  - Delete cases that purely existed to verify the fallback (e.g. `orderAssigneeCache` dedupe behavior in the bulk path).

  **GREEN** —
  - Delete `isCallerOrderAssignee` (`notification.ts:356-385`).
  - In `markUserNotificationAsRead` (`:399-441`): remove the fallback branch; if strict `findOne({ _id: notificationId, targetUserId: userId })` returns null, return `false` immediately.
  - In `resolveOwnedNotificationIds` (`:461-511`): remove the fallback block — `ownedIds = strictIds`. Simplify the return shape (drop `fallbackCount`).
  - Update `bulkMarkUserNotificationsAsRead` log to drop `fallbackOwned`.

  **REFACTOR** — The simplification collapses `resolveOwnedNotificationIds` into a single `find({ _id: { $in }, targetUserId: userId })`. Inline it into `bulkMarkUserNotificationsAsRead` if it becomes a thin wrapper.

- **Commit**: `[MLID-2391] - refactor(notifications): remove order-assignee fallback from mark-as-read auth`

---

### Step 6 — Strip the relaxed client chip filter and add `order:reassigned` listener

- **Files**:
  - `apps/web/app/orders-tracker/[category]/[orderId]/documents/useDocumentsTabNotifications.ts`
  - `apps/web/app/orders-tracker/[category]/[orderId]/documents/useDocumentsTabNotifications.test.tsx`
  - `apps/web/app/orders-tracker/hooks/useOrderNotificationCounts.ts`
  - `apps/web/app/orders-tracker/hooks/useOrderNotificationCounts.test.tsx`

- **What**:

  **RED** — Update existing hook tests:

  - Remove cases that asserted "chip appears when current user is order's `assignedTo` but not `targetUserId`."
  - Add a case: chip does **not** appear when current user is order's `assignedTo` but not `targetUserId`.
  - Add a case: `connection.on('order:reassigned', ...)` is registered and triggers a refetch.

  **GREEN** —

  In `useDocumentsTabNotifications.ts`:
  - Remove the `orderAssignedTo` parameter (and its callers' supplied value). The signature becomes `useDocumentsTabNotifications(orderId)`.
  - Update `unreadNotificationIdsForCurrentUser` filter to drop the `|| orderAssignedTo === userId` clause:
    ```typescript
    const unreadNotificationIdsForCurrentUser = notifications
      .filter((n) => !n.isReadByCurrentUser && n.targetUserId === userId)
      .map((n) => n._id);
    ```
  - In the SignalR effect, add a third listener: `connection.on('order:reassigned', fetchNotifications)` (and a matching `off` in cleanup).

  In `useOrderNotificationCounts.ts`:
  - Add `connection.on('order:reassigned', refetch-equivalent)` (and matching `off`).

  Update all call sites to drop the second positional arg.

  **REFACTOR** — None expected. The signature change is mechanical; the filter is now a one-line predicate.

- **Commit**: `[MLID-2391] - refactor(documents-tab): drop relaxed chip predicate, add order:reassigned listener`

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/services/mongodb/notification.ts` | Modify | Add `reassignUserNotificationsForOrder`; remove `userReadScope` param and `'any'` lookup branch; remove `isCallerOrderAssignee`; simplify `resolveOwnedNotificationIds` |
| `apps/web/services/mongodb/notification.test.ts` | Modify | Tests for retarget helper; update mark-as-read and count tests for stripped fallbacks |
| `apps/web/services/notifications/handleOrderReassignment.ts` | Create | Fire-and-forget orchestrator (retarget + SignalR broadcast) |
| `apps/web/services/notifications/handleOrderReassignment.test.ts` | Create | Unit tests for the orchestrator |
| `apps/web/app/api/orders/new/route.ts` | Modify | Fire-and-forget trigger between audit and response |
| `apps/web/app/api/orders/new/route.test.ts` | Modify | Integration tests for trigger conditions |
| `apps/web/app/api/notifications/counts/route.ts` | Modify | Drop `userReadScope: 'any'` |
| `apps/web/app/api/notifications/counts/route.test.ts` | Modify | Update expectations for current-target predicate |
| `apps/web/app/api/notifications/by-order/[orderId]/route.ts` | Modify | Drop `userReadScope: 'any'` |
| `apps/web/app/api/notifications/by-order/[orderId]/route.test.ts` | Modify | Update expectations for current-target predicate |
| `apps/web/app/orders-tracker/[category]/[orderId]/documents/useDocumentsTabNotifications.ts` | Modify | Drop `orderAssignedTo` parameter and relaxed predicate; add `order:reassigned` listener |
| `apps/web/app/orders-tracker/[category]/[orderId]/documents/useDocumentsTabNotifications.test.tsx` | Modify | Update tests for new signature and listener |
| `apps/web/app/orders-tracker/hooks/useOrderNotificationCounts.ts` | Modify | Add `order:reassigned` listener |
| `apps/web/app/orders-tracker/hooks/useOrderNotificationCounts.test.tsx` | Modify | Test the new listener |
| Call sites of `useDocumentsTabNotifications` | Modify | Drop the second arg |

---

## Testing Strategy

### Unit tests — `reassignUserNotificationsForOrder`

- Early-returns `0` when any of `orderId` / `fromUserId` / `toUserId` is empty (no DB call).
- Issues a `Notification.updateMany` with the exact filter shape (`type: 'user'`, `targetUserId: fromUserId`, `entities.$elemMatch`) and `$set: { targetUserId: toUserId }`.
- Returns Mongo `modifiedCount`.
- Re-throws on DB rejection (orchestration layer above catches it).

### Unit tests — `handleOrderReassignment`

- Feature-flag-off early-return.
- Empty previous, empty new, or equal previous/new — all early-return.
- Reassignment: retargets, then broadcasts `order:reassigned { orderId }`. Broadcast happens after retarget (mock invocation order).
- Retarget rejects: no broadcast, error logged, no throw.
- SignalR unavailable: no broadcast, error logged, no throw; retarget already ran.
- Broadcast rejects: error logged, no throw.

### Integration tests — `PUT /api/orders/new`

- Cascade fires once per item in `updateOrder` result when `assignedTo` changes from previous truthy to different truthy (covers protocol-partner orders).
- Cascade does not fire when: body omits `assignedTo`; `assignedTo` unchanged; previous `assignedTo` was empty; editor lacks `isAllowedToEditOrderAssignedTo`.
- Response is HTTP 200 even when the orchestrator rejects.

### Unit tests — predicate flip and workaround removal

- `getCountForEntity` returns the count using the current-target join. Regression: a `notificationreads` row whose `userId !== notification.targetUserId` does not clear the count.
- `markUserNotificationAsRead` returns `false` when caller is the order's `assignedTo` but not the notification's `targetUserId`.
- `bulkMarkUserNotificationsAsRead` returns only strict-owned ids; fallback-owned ids no longer surface.
- `useDocumentsTabNotifications` does not show the chip when current user is order's `assignedTo` but not `targetUserId`.
- Both hooks subscribe to `order:reassigned` and refetch on the event.

### Manual verification on staging (post-deploy)

1. Sign in as user A. Open an order assigned to A; confirm Documents tab shows unread badges and "New" chips.
2. Mark a few as read as A; verify they disappear for A.
3. Reassign the order to user B.
4. Confirm A's open Documents tab refetches (via `order:reassigned`) and all chips / mark-as-read buttons disappear for A. The tab badge drops to 0 for A.
5. Sign in as B in a separate session; open the same order's Documents tab. Confirm every document shows a "New" chip and the badge equals the document count — including the ones A had marked as read (PO requirement).
6. Mark a few as read as B; verify they disappear for B.
7. Mongo check: notification rows for the reassigned order now have `targetUserId = userB`. Prior `notificationreads` rows from A are still present (audit retention). New `notificationreads` rows from B exist for the ones B marked.
8. Repeat with a protocol-partner order (`-1` suffix) to confirm both the primary and partner cascade.

---

## Security Considerations

- **Authorization at trigger site**: gated by `isAllowedToEditOrderAssignedTo(session.user.userObj, 'new')` (already in place). If the check fails, `assignedToFields` stays `{}`, the cascade condition is false, the helper is never invoked.
- **Authorization tightens at read site**: mark-as-read no longer accepts the `assignedTo`-fallback. The only path to mark a notification as read is `targetUserId === userId`. This is a narrowing of authority and was reviewed by the team as the correct semantics under retarget.
- **PHI handling**: the helper operates only on `orderId`, `userId`, and notification documents. No patient data is touched. SignalR payload (`{ orderId }`) carries no PHI.
- **Fire-and-forget safety**: orchestrator invoked without `await`, with a `.catch` guard. Failure cannot affect HTTP response.
- **Feature-flag gating**: `ORDER_DOCUMENT_NOTIFICATIONS` (default `false`) gates the orchestrator and matches the rest of the notification surface.
- **Audit retention**: prior-assignee `notificationreads` rows survive — a "who read what when" audit query against the collection remains answerable.

---

## Open Questions

- None at planning time.
