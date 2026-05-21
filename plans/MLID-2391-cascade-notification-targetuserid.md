# MLID-2391 — Cascade notification targetUserId when an order is reassigned

## Task Reference

- **Jira**: [MLID-2391](https://localinfusion.atlassian.net/browse/MLID-2391)
- **Parent Epic**: [MLID-2011](https://localinfusion.atlassian.net/browse/MLID-2011) — Orders: Document Notifications
- **Predecessor**: [MLID-2085](https://localinfusion.atlassian.net/browse/MLID-2085) — Documents Tab: relaxed UI guard (ships first as short-term mitigation)
- **Story Points**: 3
- **Branch**: `feature/MLID-2391-cascade-notification-targetuserid`
- **Base Branch**: `develop`
- **Status**: To Do (planning complete, scheduled for next sprint — created 2026-05-15, close to release date so the heavier server-side work was deferred)

---

## Summary

Document notifications snapshot `targetUserId` from `order.assignedTo` at write time and are never updated when the order is reassigned. MLID-2085 patched both the UI symptom (relaxed gating predicate) AND the server-side mark-as-read authz (fallback path via order-assignee verification), so QA acceptance is already met without this cascade. This ticket closes the *data-side* gap: when an order is reassigned, all unread notifications tied to that order are retargeted to the new assignee, and a `documentAdded` SignalR broadcast tells the new assignee's UI to refetch live.

After this ships, the data is self-consistent (every notification's `targetUserId` matches the current assignee), the fallback authz path becomes a rarely-taken legacy/race code path, and the existing OR-branches in the UI guard and server-side mark-as-read are intentionally kept as belt-and-suspenders for races (cascade fails mid-loop, SignalR arrives before the cascade write).

---

## Codebase Analysis

**Order reassignment write path.** Both `apps/web/app/api/orders/new/route.ts` and `apps/web/app/api/orders/maintenance/route.ts` follow the same pattern: the PUT handler builds `assignedToFields = { assignedTo, lastAssigned, assignedBy }` (gated by `isAllowedToEditOrderAssignedTo(session.user.userObj)`), fetches `currentOrder` via raw `db.collection().findOne()`, then calls `updateOrder({ displayId, fieldsToUpdate, ... }, category)` from `apps/web/services/mongodb/ordersTracker.ts`. After `updateOrder` resolves, the route calls the audit helper, then `return NextResponse.json(result)`. The cascade block belongs between the audit call and the return.

**Authorization.** `isAllowedToEditOrderAssignedTo` is checked when building `assignedToFields`. If it fails, the field is omitted, the cascade detection condition is false, and the cascade is never reached. No additional authorization logic is required inside the cascade.

**Partner-order coverage.** `updateOrder` propagates the same `fieldsToUpdate` to a protocol-partner order (`-1` suffix). It returns `FullOrder<K>[]` containing every modified document. Every `_id` in that array must be cascaded, not just the one identified by `body.displayId`.

**Raw driver in `updateOrder`.** `updateOrder` uses `connectToDatabase()` → `db.collection().updateOne()` — Mongoose middleware would be bypassed. The cascade must be invoked explicitly from the route handler, not via a Mongoose hook.

**"Unread" semantics for `type: 'user'` notifications.** The `isRead` boolean on `INotification` is for `type: 'task'` only. A `type: 'user'` notification is unread when no `NotificationRead` record exists for `(notificationId, userId)`. The cascade must exclude notifications that the prior assignee has already read (audit preservation).

**No bulk-update primitive exists.** `apps/web/services/mongodb/notification.ts` has `createNotification`, `getNotificationsForEntity`, etc., but nothing that bulk-updates `targetUserId`. The new function `cascadeOrderNotificationTargetUserId` will be added to that file.

**SignalR pattern.** `apps/web/services/notifications/documentNotification.ts` lines 99-128 show the established pattern: lazy init, try/catch around `broadcastToAll`, never-throw. The cascade reuses the same primitive (`broadcastToAll('documentAdded', { orderId })`) so the existing `useDocumentsTabNotifications` hook listener (which already subscribes to `documentAdded`) picks it up without UI changes.

---

## Implementation Steps

### Step 1 — Add `cascadeOrderNotificationTargetUserId` service function

- **Files**:
  - `apps/web/services/mongodb/notification.ts`
  - `apps/web/services/mongodb/notification.test.ts` (create if absent, otherwise extend)

- **What**:

  RED — Create or extend `notification.test.ts` with `@jest-environment node` and module-level mocks for `Notification`, `NotificationRead`, and `logger`. Add a `cascadeOrderNotificationTargetUserId` describe block with the following test cases:

  - `updates targetUserId on unread notifications for the given order ids`
    Arrange three notifications on an order — two unread (no `NotificationRead` record for prior assignee), one already read by prior assignee. Mock `NotificationRead.find` to return the read notification's id for the prior assignee. Mock `Notification.updateMany` to resolve `{ modifiedCount: 2 }`. Assert `Notification.updateMany` is called once with a filter that excludes the already-read notification id.
  - `returns early without touching the database when newTargetUserId is null`
    Assert `Notification.find` is never called.
  - `returns early without touching the database when newTargetUserId is an empty string`
    Assert `Notification.find` is never called.
  - `returns early without touching the database when orderIds array is empty`
    Assert `Notification.find` is never called.
  - `does not throw when Notification.updateMany rejects — logs the error and resolves void`
    Mock `Notification.updateMany` to reject. Assert the returned promise resolves (does not throw) and `logger.error` was called.

  GREEN — Export the function from `notification.ts`:

  ```typescript
  export const cascadeOrderNotificationTargetUserId = async (
    orderIds: string[],
    priorTargetUserId: string,
    newTargetUserId: string
  ): Promise<void>
  ```

  Implementation logic:
  1. Guard: return immediately if `newTargetUserId` is falsy or `orderIds.length === 0`.
  2. Fetch all `type: 'user'` notifications whose `entities` array contains an entry `{ entityType: 'order', entityId: $in orderIds }` and whose `targetUserId === priorTargetUserId`. Use `Notification.find(...).lean()`.
  3. If no notifications found, return early.
  4. Fetch `NotificationRead` records where `notificationId` is in the found set AND `userId === priorTargetUserId`. These have been read by the prior assignee and must not be retargeted.
  5. Compute `eligibleIds` = found notification ids minus the already-read set.
  6. If `eligibleIds.length === 0`, return early.
  7. Call `Notification.updateMany({ _id: { $in: eligibleIds } }, { $set: { targetUserId: newTargetUserId } })`.
  8. Log info on success, log error on catch (never rethrow).

  REFACTOR — Extract a named helper `buildAlreadyReadIds(notifications, priorUserId)` if the find + diff logic exceeds one readable block.

- **Commit**: `[MLID-2391] - feat(notifications): add cascadeOrderNotificationTargetUserId service function`

---

### Step 2 — Wire cascade + SignalR broadcast into the `new` order PUT route

- **Files**:
  - `apps/web/app/api/orders/new/route.ts`
  - `apps/web/app/api/orders/new/route.test.ts`

- **What**:

  RED — Add tests in the `PUT /api/orders/new` describe block:

  - `fires cascade when assignedTo changes to a truthy value`
    `mockFindOne` returns `{ assignedTo: 'user-old', ... }`. Request body has `assignedTo: 'user-new'`. Mock `updateOrder` to return `[{ _id: 'order-obj-id-1' }, { _id: 'order-obj-id-2' }]`. Assert `cascadeOrderNotificationTargetUserId` is called with `(['order-obj-id-1', 'order-obj-id-2'], 'user-old', 'user-new')`.
  - `does not fire cascade when assignedTo is unchanged`
    `mockFindOne` returns `{ assignedTo: 'user-same' }`. Body has `assignedTo: 'user-same'`. Assert the cascade is NOT called.
  - `does not fire cascade when new assignedTo is absent or empty`
    Body has no `assignedTo` field. Assert the cascade is NOT called.
  - `returns 200 even when cascade rejects`
    Mock the cascade to reject. Assert response status is 200.
  - `broadcasts documentAdded per affected order id after cascade`
    Assert `broadcastToAll` is called with `'documentAdded'` for each order id in the result.

  GREEN — Add to the PUT handler in `route.ts`, between the audit call and `return NextResponse.json(result)`:

  ```typescript
  const priorAssignedTo =
    typeof currentOrder?.assignedTo === 'string'
      ? currentOrder.assignedTo
      : null;
  const newAssignedTo =
    'assignedTo' in assignedToFields
      ? (assignedToFields as { assignedTo: string }).assignedTo
      : null;

  if (newAssignedTo && priorAssignedTo && newAssignedTo !== priorAssignedTo) {
    const affectedOrderIds = result.map((o) => String(o._id));
    cascadeOrderNotificationTargetUserId(
      affectedOrderIds,
      priorAssignedTo,
      newAssignedTo
    )
      .then(async () => {
        const signalr = await getSignalRService().catch(() => null);
        if (signalr) {
          await Promise.allSettled(
            affectedOrderIds.map((orderId) =>
              signalr.broadcastToAll('documentAdded', { orderId })
            )
          );
        }
      })
      .catch((err: unknown) => {
        logger.error(
          '[MLID-2391] Cascade or SignalR broadcast failed for new order reassignment',
          { err }
        );
      });
  }
  ```

  Import `cascadeOrderNotificationTargetUserId` from `@/services/mongodb/notification` and `getSignalRService` from `@/services/signalr`.

  REFACTOR — Extract a named helper `fireCascadeAndBroadcast(affectedOrderIds, priorAssignedTo, newAssignedTo)` if the inline block makes the handler hard to read. This helper will be reused in the maintenance route (Step 3), justifying the extraction.

- **Commit**: `[MLID-2391] - feat(orders/new): fire notification cascade and SignalR broadcast on reassignment`

---

### Step 3 — Wire cascade + SignalR broadcast into the `maintenance` order PUT route

- **Files**:
  - `apps/web/app/api/orders/maintenance/route.ts`
  - `apps/web/app/api/orders/maintenance/route.test.ts`

- **What**:

  RED — Mirror the five test cases from Step 2 in the maintenance route test file's `PUT /api/orders/maintenance` describe block. The mock setup is structurally identical — swap `COLLECTION.NewOrders` references for `COLLECTION.MaintenanceOrders` and import the maintenance `PUT` handler.

  GREEN — Apply the identical injection block from Step 2 to `apps/web/app/api/orders/maintenance/route.ts` at the same relative position (after the audit call, before `return NextResponse.json(result)`).

  If `fireCascadeAndBroadcast` was extracted in Step 2, import and reuse it here.

  REFACTOR — Confirm no duplication remains between the two route files for this logic.

- **Commit**: `[MLID-2391] - feat(orders/maintenance): fire notification cascade and SignalR broadcast on reassignment`

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/services/mongodb/notification.ts` | Modify | Add `cascadeOrderNotificationTargetUserId` export |
| `apps/web/services/mongodb/notification.test.ts` | Create or Modify | Unit tests for `cascadeOrderNotificationTargetUserId` |
| `apps/web/app/api/orders/new/route.ts` | Modify | Inject cascade + SignalR broadcast block after audit call |
| `apps/web/app/api/orders/new/route.test.ts` | Modify | Add cascade and broadcast test cases for PUT handler |
| `apps/web/app/api/orders/maintenance/route.ts` | Modify | Inject cascade + SignalR broadcast block after audit call |
| `apps/web/app/api/orders/maintenance/route.test.ts` | Modify | Add cascade and broadcast test cases for PUT handler |

---

## Testing Strategy

**Unit tests — `cascadeOrderNotificationTargetUserId`:**
- Returns early (no DB calls) when `newTargetUserId` is falsy.
- Returns early when `orderIds` is empty.
- Fetches unread `type: 'user'` notifications scoped to the given order ids and prior assignee.
- Excludes notifications where a `NotificationRead` record exists for the prior assignee.
- Calls `Notification.updateMany` with only the eligible notification ids.
- Does not rethrow when `updateMany` rejects — resolves void and logs the error.

**Integration tests — route PUT handlers (new + maintenance):**
- Cascade fires with correct `(orderIds, priorAssignedTo, newAssignedTo)` arguments when reassignment changes the assignee to a truthy value.
- Cascade does not fire when `assignedTo` is unchanged or absent.
- Response returns 200 even when the cascade promise rejects.
- `broadcastToAll('documentAdded', ...)` is called per affected order id after cascade resolves.

**Manual verification on staging (post-deploy):**
1. Log in as user A. Reassign an order to user B.
2. Log in as user B. Confirm the Documents tab shows correct unread counts and "Mark as Read" controls without page reload.
3. Confirm via Mongo query that the notifications' `targetUserId` was updated to user B.
4. Confirm any notifications the previous assignee (user A) had already marked as read still have `targetUserId === user A` (audit preservation).
5. Reassign to user A: confirm the cycle works in the other direction.
6. Un-assign the order (clear `assignedTo`): confirm no notifications are touched.

---

## Security Considerations

**Authorization.** The cascade runs only after `isAllowedToEditOrderAssignedTo(session.user.userObj)` has been enforced by the route. If the check fails, `assignedToFields` is empty, the detection condition (`newAssignedTo && priorAssignedTo && newAssignedTo !== priorAssignedTo`) is false, and the cascade is never reached. No extra authorization is required.

**PHI handling.** The cascade operates only on `notificationId`, `userId` (Mongo ObjectId strings), and `orderId` strings — no patient data. The `documentAdded` SignalR broadcast carries only `orderId`, consistent with the established PHI minimization in `documentNotification.ts`.

**Fire-and-forget safety.** The cascade is not awaited by the route response, so a failure cannot affect the order-update outcome. The `.catch` handler logs the error for observability.

**Input validation.** `newTargetUserId` is sourced from the already-validated and authorized `assignedToFields.assignedTo`. `orderIds` comes from `updateOrder`'s return value. No additional sanitization is required.

---

## Open Questions

1. **Lead orders (`apps/web/app/api/orders/lead/route.ts`).** The lead route builds `assignedToFields` with the same pattern. Documents are not currently surfaced for lead orders, so the cascade is not wired there in this task. Confirm with the team whether a follow-up ticket is needed proactively or "skip until documents are surfaced for leads" is acceptable.

2. **Backfill script for existing stale notifications on staging.** Order `6a073015c3a15ed77b2e2574` has 18 notifications still targeting cavila despite the order now being assigned to alejandro. MLID-2085's relaxed UI guard already makes them functionally visible to the current assignee. Decide whether to add a one-shot script to this PR that updates `targetUserId` on those records (and any other historically-affected orders), or rely on the relaxed UI guard for legacy data.
