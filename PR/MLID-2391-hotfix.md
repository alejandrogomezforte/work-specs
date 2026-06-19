# [MLID-2391] fix(notifications): initial-assignment backfill + drop orphan audit rows on unassigned orders

> **Also fixes MLID-2439** — see Root Cause (b) below. Two bugs are addressed by a single coordinated change to the propagation orchestrator; one commit per bug, in dependency order.

## Summary

- **MLID-2391** — when an order is created with `assignedTo: ""` (the standard manual-creation flow at an IG-less location) and later assigned to a user, the non-Order documents that auto-attached at creation never had notifications written for the new assignee. The assignee saw the documents in the order's Documents tab but the unread-count badge and per-row "New" chip stayed at zero.
- **MLID-2439** — `propagateDocumentNotificationToOrder` was writing audit rows (and broadcasting `documentAdded`) even when the order had no assignee. The Order-History tab then showed "New Document received" entries that no user could ever act on. Combined with the MLID-2391 fix, those orphan rows turned into duplicates the moment the initial-assignment backfill ran.
- Single fix path: extend `handleOrderReassignment` with an "empty previous → backfill via `notifyOrderOfRecentDocuments`" branch (MLID-2391), AND gate all three propagator side-effects (notification + audit + SignalR broadcast) on `order.assignedTo` truthy (MLID-2439). The two changes are independent in code, but together they produce a coherent rule: **side-effects fire only when there is a real recipient.**

## Root Cause

Two independent code paths conspired to produce the original symptom and the duplicate-row follow-up.

### (a) MLID-2391 — no backfill on initial assignment

`handleOrderReassignment` was specced as a **retarget** of existing notifications (`updateMany` flipping `targetUserId`). Its guard was `if (!previousAssignedTo || !newAssignedTo) return;` — when `previousAssignedTo === ""`, the helper returned early. Even without the guard, the underlying `reassignUserNotificationsForOrder` is a retarget against existing rows; with zero pre-existing notifications (because the auto-attach notification write was gated out at creation time), there was nothing to retarget anyway.

The route handler `PUT /api/orders/new` had a matching guard `previousAssignedTo && reassignedTo && reassignedTo !== previousAssignedTo` that filtered out the empty-previous case before `handleOrderReassignment` was ever called.

Combined effect: orders created with `assignedTo: ""` and later assigned had their auto-attached non-Order documents permanently invisible on the assignee's badge / "New" chip surface.

### (b) MLID-2439 — orphan audit rows on unassigned orders

`propagateDocumentNotificationToOrder` gated only the notification write on `order.assignedTo`. Steps 3 (audit) and 4 (SignalR) ran unconditionally:

```ts
if (order.assignedTo) {
  await writeOrderDocumentNotification({ ... });
}
await writeOrderDocumentAuditEntry({ ... });     // ← always wrote
// then broadcastDocumentAddedEvent ran unconditionally too
```

When a document auto-attached to an unassigned order (via `notifyDocumentCreated` or `notifyOrderOfRecentDocuments`), the audit row was written even though no user could ever receive a notification for it. The Order-History tab then surfaced a "New Document received (Document type = X)" entry with no corresponding notification.

After (a) is fixed in this branch, the orphan rows from (b) become **duplicates**: one audit row written at auto-attach time (no assignee yet), one more written at backfill time when the assignee is set. The Order-History tab shows the same document twice. Fixing (b) restores the invariant that all three side-effects of the orchestrator are gated together — no recipient, no side-effects.

## Changes Overview

- **Files changed**: 6
- **Lines added / removed**: +330 / -78
- **New files**: 0
- **Commits**: 2 logical (1 per bug) on top of develop, plus a develop-into-branch merge.

## Files Changed

### Orchestrator + caller (MLID-2391 backfill)

| File | Action | Description |
|---|---|---|
| `apps/web/services/notifications/handleOrderReassignment.ts` | Modify | Branch on `!previousAssignedTo`: empty previous → call `notifyOrderOfRecentDocuments` for the backfill (covering MLID-2391), populated previous → existing `reassignUserNotificationsForOrder` retarget path (unchanged). Added an `extractPatientId` helper that handles both the raw-string and populated-object shapes of `order.patient` (the aggregation pipeline in `updateOrder` resolves the patient reference). Defensive `!newAssignedTo` early-return kept against direct API misuse. |
| `apps/web/services/notifications/handleOrderReassignment.test.ts` | Modify | New `initial assignment (previousAssignedTo empty, newAssignedTo populated)` describe block with 5 cases (happy path, no-retarget on initial path, backfill rejection, populated-object patient shape, missing patient). Existing retarget cases unchanged. 17 tests total. |
| `apps/web/app/api/orders/new/route.ts` | Modify | `previousAssignedTo` now defaults to `""` instead of `null` when the underlying field is missing/non-string. Route guard simplified to `reassignedTo && reassignedTo !== previousAssignedTo` — empty-previous transitions now reach `handleOrderReassignment`. |
| `apps/web/app/api/orders/new/route.test.ts` | Modify | Replaced the "does not trigger when previous was empty" test with two positive cases (missing `assignedTo` field + explicit `assignedTo: ""`), both asserting `handleOrderReassignment` fires with `previousAssignedTo: ""`. |

### Propagator gate (MLID-2439 orphan rows + duplicate guard)

| File | Action | Description |
|---|---|---|
| `apps/web/services/notifications/documentNotification.ts` | Modify | `propagateDocumentNotificationToOrder` now skips ALL THREE side-effects (notification + audit + SignalR) when `order.assignedTo` is falsy. Added a log line ("Document propagation skipped — no assignee") at the new early-return point. The docstring captures the rationale for both MLID-2439 and the MLID-2391 duplicate-row interaction. |
| `apps/web/services/notifications/documentNotification.test.ts` | Modify | Flipped 7 existing tests in `notifyDocumentCreated`, `notifyOrderOfRecentDocuments`, and `propagateDocumentNotificationToOrder` from "still writes audit / still broadcasts" to "does NOT write / does NOT broadcast" when the order is unassigned. Added one new case covering `assignedTo: ""` (the realistic Mongo shape, vs. `undefined`). |

## What Changed and Why

### MLID-2391 — initial-assignment backfill

`handleOrderReassignment` now distinguishes two cases instead of one:

```ts
if (!newAssignedTo) return;                       // defensive (un-assignment impossible via UI)
if (previousAssignedTo === newAssignedTo) return; // no-op

const isInitialAssignment = !previousAssignedTo;
if (isInitialAssignment) {
  await notifyOrderOfRecentDocuments({
    orderId,
    assignedTo: newAssignedTo,
    patientId: extractPatientId(order.patient),
    auditSource: 'user',
  });
} else {
  await reassignUserNotificationsForOrder(orderId, previousAssignedTo, newAssignedTo);
}
// then broadcast `order:reassigned` (both branches)
```

`extractPatientId` was added after the first end-to-end test failed: `updateOrder`'s aggregation pipeline returns orders with `patient` resolved to a populated object (`{ _id, fullName, ... }`), not the raw string stored in Mongo. The helper accepts either shape, mirroring the existing successful caller pattern at `apps/web/app/api/orders/new/route.ts:158` (`String(createdOrder.patient._id)`).

The route's `previousAssignedTo` was changed from a nullable type (`string | null`) to a string defaulting to `""`. The guard then fires on `reassignedTo !== previousAssignedTo`, which is `true` for both empty-to-populated and populated-to-populated. The empty-to-populated case now reaches `handleOrderReassignment`, which routes it to the backfill branch.

### MLID-2439 — gate all three side-effects on `order.assignedTo`

```ts
if (!isDocumentEligibleForOrder(document, order)) {
  // unchanged eligibility gate (Order-category docs must be linked)
  return;
}

if (!order.assignedTo) {
  logger.info('Document propagation skipped — no assignee (audit + notification + broadcast deferred until the order is assigned)', { ... });
  return;
}

// All three side-effects now fire together:
await writeOrderDocumentNotification({ ... });
await writeOrderDocumentAuditEntry({ ... });
const signalr = await getSignalR();
if (signalr) {
  await broadcastDocumentAddedEvent(signalr, { ... });
}
```

The orchestrator is called from four trigger sites:

| Caller | Behavior on unassigned order — before | Behavior on unassigned order — after |
|---|---|---|
| `notifyDocumentCreated` (upload → patient's orders) | Wrote audit row, broadcast, no notification | No-op (deferred) |
| `notifyOrderOfRecentDocuments` (order-creation backfill) | Same | No-op (deferred) |
| `handleOrderLink` (link-orders route — formerly `firePropagationForNewlyLinkedOrders`) | Same | No-op (deferred) |
| `handleOrderReassignment` initial-assignment backfill (new in MLID-2391) | n/a — only fires once `assignedTo` is being set | All three side-effects fire (one audit row per eligible document, no duplicates) |

The "deferred" cases land their audit row + notification at the moment the order gets its first assignee (via the MLID-2391 backfill path). No work is lost — it's just consolidated to a single point in the order's lifecycle, when there is a real recipient.

### Side-effect to confirm acceptable in review

Linking an Order-category document to an **unassigned** order no longer writes any audit row at link time. The audit row appears when the order is later assigned (the backfill picks up the now-linked Order-category doc through the eligibility gate). This aligns with the PO's stated rule ("audit only when transitioning to a real user") but is a slightly different surface from the auto-attach case — flagging explicitly.

## Commits

| Hash | Message |
|---|---|
| `4a4a7016d` | `[MLID-2391] - fix(notifications): backfill notifications on initial order assignment` |
| `bbd477a74` | `[MLID-2391] - fix(notifications): gate audit + SignalR on assignedTo (also fixes MLID-2439)` |

## Test Plan

### Automated (all passing)

- [x] Unit tests for `handleOrderReassignment` covering the initial-assignment branch (5 new cases) and the existing retarget branch (unchanged). 17 tests pass total.
- [x] Unit tests for `propagateDocumentNotificationToOrder` and the two fan-out helpers (`notifyDocumentCreated`, `notifyOrderOfRecentDocuments`) updated to assert the new "no audit / no broadcast when unassigned" behavior. 98 tests pass in `documentNotification.test.ts`.
- [x] Integration tests for `PUT /api/orders/new` covering the empty → populated `assignedTo` transition (2 new cases: missing field and explicit `""`). All 56 route tests pass.
- [x] Broader sweep across 14 related test suites (handleOrderUnlink, link-orders, useDocumentsTabNotifications, useOrderNotificationCounts, AuditLogTable, audit-config, notification service helpers): 376 tests pass.
- [x] `npx tsc --noEmit` clean on the changed files.
- [x] `npx eslint` clean on the 6 changed files.
- [x] `npx prettier --check` clean on the 6 changed files.
- [x] Line coverage ≥ 80% on every changed source file (`route.ts`: 100%, `documentNotification.ts`: 98.91%, `handleOrderReassignment.ts`: 93.02%).

### Manual verification

Locally exercised against order `05Wv` (`6a171b8b16035e07bc75199f`) on the fix branch:

- Order created at an IG-less location → `neworders.assignedTo === ""`.
- `auditLogs` for the order at creation time: only the `create` row (no orphan document audits — MLID-2439 fix verified).
- `notifications`: 0 at creation time (gate held).
- Assigned the order to `Alejandro Gomez FO` via Order Tracker → Order Details.
- After assignment: `notifications` gained 2 rows (one per auto-attached non-Order document, targeting the new assignee, `isRead: false`). `auditLogs` gained 3 new rows: the `assignedTo` change (route's existing audit pipeline) + 2 document-attachment rows from the backfill. **No duplicates.** MLID-2391 fix verified.
- Order's Documents tab badge counter reflects the 2 new notifications live (SignalR broadcast triggered the client refetch).

### Reviewer-side verification suggestions

- Create a fresh order at any location without an assigned IG → confirm the order lands with `assignedTo: ""` and that the Order-History tab shows ONLY "Order Created" (no document rows) before assignment.
- Assign the order → confirm two new History rows appear ("New Document received (Document type = Other)" and "New Document received (Document type = Referral Copy)" or whichever non-Order docs auto-attached), the badge counter increments, and the "New" chip + "Mark as Read" button render on the affected document rows in the Documents tab.
- Re-assign the order to a different user → confirm the existing retarget path still works: `targetUserId` on the 2 notifications flips, no new rows are written, the new assignee sees both as unread.

## Jira

- [MLID-2391](https://localinfusion.atlassian.net/browse/MLID-2391) — primary tracking ticket. The branch name uses MLID-2391 because that's where the planning + postmortem context lives. Original ticket is already `Done`; the QA team can review this PR and mark it Done a second time on landing.
- [MLID-2439](https://localinfusion.atlassian.net/browse/MLID-2439) — the orphan-audit-rows bug. Resolves alongside MLID-2391 since the same `assignedTo` gate change fixes it.

D4-T5 (MLID-2454), the maintenance-orders equivalent of the reassignment cascade, inherits the updated `handleOrderReassignment` behavior without modification (the helper does not branch on collection).
