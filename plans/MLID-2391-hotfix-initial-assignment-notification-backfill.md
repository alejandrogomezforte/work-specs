# MLID-2391 hotfix — Initial-assignment notification backfill

## Task Reference

- **Jira**: [MLID-2391](https://localinfusion.atlassian.net/browse/MLID-2391) — `[D0-T1] Reassignment cascades notifications to new assignee (neworders)` (already merged + Done — this hotfix reuses the same ticket key on a separate branch; no Jira reopen)
- **Parent epic**: [MLID-2417](https://localinfusion.atlassian.net/browse/MLID-2417)
- **Branch**: `fix/MLID-2391-initial-assignment-notification-backfill`
- **Base branch**: `develop` (D0 hotfixes branch off develop directly per the epic's delivery strategy)
- **Status**: **Done** — implementation complete, quality gates passed, local UI verification confirmed. PR doc at `docs/agomez/PR/MLID-2391-hotfix.md`. Mid-session scope grew to also cover MLID-2439 (orphan audit rows on unassigned orders) — both bugs fixed together. Ready to push and open PR.
- **Sprint**: current active sprint
- **Postmortem**: `docs/agomez/postmortem/MLID-2391-unassigned-to-assigned-skips-notification-backfill.md` (full root-cause + design analysis)

---

## Summary

A pre-existing gap in MLID-2391's reassignment cascade was surfaced during MLID-2413's local UI testing. When an order is created with `assignedTo: ""` (the standard manual-creation flow at an IG-less location) and later assigned to a user, the non-Order documents that auto-attached at order-creation time never get notifications written for the new assignee. The reassignment cascade was specced as a **retarget** of existing notifications from a previous assignee to a new one, so it early-returns when `previousAssignedTo` is empty — there is nothing to retarget. As a result, the assignee sees the documents listed in the order's Documents tab but the unread-count badge and per-row "New" chip stay at zero.

This hotfix extends `handleOrderReassignment` with a second branch: when the transition is empty → populated (initial assignment), call `notifyOrderOfRecentDocuments(order)` to backfill notifications instead of running the retarget. Everything else stays the same — same SignalR `order:reassigned` broadcast, same fire-and-forget guarantees, same external API to the route handler.

---

## Background — why the gap exists

Two independent code paths conspire. Neither is wrong in isolation; together they fail to cover the initial-assignment case.

**Path 1 — auto-attach at order creation skips notification writes when unassigned.** `propagateDocumentNotificationToOrder` writes a `notifications` row only when `order.assignedTo` is truthy. The audit row is written unconditionally. So at creation time with `assignedTo: ""`, audit entries exist but notifications do not.

**Path 2 — `handleOrderReassignment` early-returns when `previousAssignedTo` is empty.** The guard `if (!previousAssignedTo || !newAssignedTo) return;` exits before the retarget runs. Even without the early return, the underlying `reassignUserNotificationsForOrder` is an `updateMany` flipping `targetUserId` — a retarget operation against existing rows. With zero pre-existing notifications, it would have nothing to flip anyway.

Domain fact: un-assignment (populated → empty) is impossible by UI design. The only `assignedTo: ""` case is the initial-creation state for IG-less locations. Once assigned, the value stays populated.

---

## Implementation steps (Option B from the postmortem — reuse `notifyOrderOfRecentDocuments`)

Each step is one TDD cycle (RED → GREEN → REFACTOR) and one commit. Commit format: `[MLID-2391] - type(scope): description`.

### Step 1 — `handleOrderReassignment`: branch on initial vs. standard reassignment

**Files**:
- `apps/web/services/notifications/handleOrderReassignment.test.ts` (modify — new describe block)
- `apps/web/services/notifications/handleOrderReassignment.ts` (modify — split into two paths)

**RED — failing tests added to the existing file**:

A new `describe('initial assignment (empty → populated)')` block, covering:

- `calls notifyOrderOfRecentDocuments with the order's id + assignedTo + patientId`
- `does NOT call reassignUserNotificationsForOrder when previousAssignedTo is empty` (we're backfilling, not retargeting)
- `broadcasts order:reassigned after a successful backfill` (same event as the standard path so client hooks refetch)
- `does NOT broadcast when the backfill rejects` (failure-isolated, mirrors the existing retarget-rejection test)
- `logs and swallows when notifyOrderOfRecentDocuments rejects` (no throw)
- `auditSource forwarded as 'user'`, `actorId` forwarded if supplied (parameter shape — see Open Questions below)

Update the existing `'when previousAssignedTo is empty'` describe so its "returns without retargeting or broadcasting" assertion is replaced — that case is no longer a no-op; it now triggers the backfill path. The standard-reassignment cases (`previousAssignedTo` populated → `newAssignedTo` populated, different users) remain unchanged.

**GREEN — modify `handleOrderReassignment.ts`** to:

1. Keep the feature-flag check and the `!newAssignedTo` early-return (defensive — un-assignment is impossible via UI, but keep the guard against direct API misuse).
2. Keep the `previousAssignedTo === newAssignedTo` no-op.
3. Add the new branch: if `previousAssignedTo === ''` (initial assignment), call `notifyOrderOfRecentDocuments` with the order's data. If it rejects, log + skip the broadcast (mirroring the existing retarget-rejection behavior).
4. Else (standard reassignment, both populated), keep the existing `reassignUserNotificationsForOrder` retarget path unchanged.
5. After either DB-side step succeeds, broadcast `order:reassigned { orderId }`. The broadcast is the same in both branches because the client refetch behavior is the same.

Sketch:

```typescript
if (!isEnabled) return;
if (!newAssignedTo) return; // defensive — UI prevents this
if (previousAssignedTo === newAssignedTo) return;

const orderId = String(order._id);

try {
  if (!previousAssignedTo) {
    // Initial assignment — backfill via notifyOrderOfRecentDocuments
    await notifyOrderOfRecentDocuments({
      orderId,
      assignedTo: newAssignedTo,
      patientId: /* see Open Questions */,
      auditSource: 'user',
    });
  } else {
    // Standard reassignment — retarget
    await reassignUserNotificationsForOrder(orderId, previousAssignedTo, newAssignedTo);
  }
} catch (error) {
  logger.error('handleOrderReassignment: DB step failed — skipping broadcast', {
    error, orderId, previousAssignedTo, newAssignedTo,
  });
  return;
}

// broadcast in both successful cases
try {
  const signalr = await getSignalRService();
  await signalr.broadcastToAll('order:reassigned', { orderId });
} catch (error) {
  logger.error('handleOrderReassignment: broadcast failed — DB step already applied', {
    error, orderId,
  });
}
```

**REFACTOR** — only structural cleanup if the two branches share enough to factor out a helper. Likely not needed; the two calls have different shapes.

**Commit**: `[MLID-2391] - feat(notifications): backfill notifications on initial order assignment`

---

### Step 2 — Integration-level test on the route handler

**Files**:
- `apps/web/app/api/orders/new/route.test.ts` (modify — add one case)

**Why**: the orchestrator tests in Step 1 mock `notifyOrderOfRecentDocuments` and `reassignUserNotificationsForOrder` directly. The route test verifies the full call chain when the route handler triggers the helper.

**RED — add one case** to the existing `PUT /api/orders/new` test:

- `triggers handleOrderReassignment when assignedTo transitions from "" to a user id` — arrange an order with `assignedTo: ""`, send a PUT that sets `assignedTo: "user-1"`, assert that `handleOrderReassignment` was called with `(order, "", "user-1")`. The existing tests cover the populated → populated transition; this new case covers empty → populated.

**GREEN** — no production code change. The route handler already calls `handleOrderReassignment` whenever `assignedTo` changes (per MLID-2391's original implementation). The new path inside the helper handles the empty-previous case correctly.

**REFACTOR** — none.

**Commit**: `[MLID-2391] - test(orders): cover initial-assignment route trigger`

---

### Step 3 — Run `/verify` quality gates

Run jest/types/prettier/eslint on the touched files. Coverage check on `handleOrderReassignment.ts` should stay ≥ 80%. No new files created; one source file modified.

If any quality issues surface, fix them in a separate `[MLID-2391] - chore(quality)` commit per the project pattern.

---

## Files Affected

| File | Action | Description |
|---|---|---|
| `apps/web/services/notifications/handleOrderReassignment.ts` | Modify | Add the empty-previous branch that calls `notifyOrderOfRecentDocuments`; keep the existing retarget branch for the populated-previous case |
| `apps/web/services/notifications/handleOrderReassignment.test.ts` | Modify | Replace the "empty previousAssignedTo returns no-op" expectations with the new backfill behavior; add an initial-assignment describe block with happy path + failure cases |
| `apps/web/app/api/orders/new/route.test.ts` | Modify | One new integration test confirming the empty → populated transition triggers `handleOrderReassignment` |

No new files. No client-side changes (the existing `order:reassigned` SignalR listener already refetches on the event).

---

## Testing strategy

### Unit tests — `handleOrderReassignment`

- Feature flag off → no-op (unchanged).
- `newAssignedTo` empty → no-op (unchanged, defensive).
- `previousAssignedTo === newAssignedTo` → no-op (unchanged).
- **NEW** Initial assignment (`previousAssignedTo === ''`, `newAssignedTo` populated):
  - Calls `notifyOrderOfRecentDocuments` exactly once with the right params.
  - Does NOT call `reassignUserNotificationsForOrder`.
  - Broadcasts `order:reassigned` after the backfill resolves.
  - On backfill rejection: logs, skips broadcast, does not throw.
- **UNCHANGED** Standard reassignment (`previousAssignedTo` and `newAssignedTo` both populated, different users):
  - Calls `reassignUserNotificationsForOrder` exactly once.
  - Does NOT call `notifyOrderOfRecentDocuments`.
  - Broadcasts `order:reassigned` after the retarget resolves.
  - On retarget rejection: logs, skips broadcast, does not throw.
- SignalR resolution failure: logs, no throw, DB step already ran (both branches).
- Broadcast rejection: logs, no throw (both branches).

### Integration test — `PUT /api/orders/new`

- New case: `assignedTo: "" → "user-1"` triggers `handleOrderReassignment("", "user-1")` exactly once.
- Existing reassignment + no-change cases must still pass.

### Manual verification

1. Enable `ORDER_DOCUMENT_NOTIFICATIONS` in `/admin/feature-flags`.
2. Create a fresh order via the standard UI flow on an IG-less location → confirm `assignedTo: ""` and zero notifications.
3. Confirm at least one non-Order document auto-attached (audit log on the order should show entries for `Referral Copy`, `Other`, etc.).
4. Assign the order to yourself via Order Tracker → Order Details.
5. **Expected after fix**: notifications appear for each auto-attached non-Order document, targeting the new assignee. The Documents tab badge counter on the order matches the number of auto-attached non-Order docs.
6. Re-assign the order to a different user → confirm the existing retarget path still works (notifications flip target, no duplicates).

Optional Mongo spot-check after Step 5: `db.notifications.find({ entities: { $elemMatch: { entityType: 'order', entityId: '<orderId>' } } })` returns one row per non-Order doc, all with `targetUserId` = the new assignee, `isRead: false`.

---

## Out of scope

- **Partner-order propagation details** — the existing reassignment cascade tests already cover how partner orders (the `-1` suffix companions) flow through. No change to that behavior.
- **Audit-log replay (postmortem Option A)** — reading the order's `field: 'documents'` audit entries to figure out which documents to backfill. Not needed because `notifyOrderOfRecentDocuments` already walks the patient's recent documents and applies the eligibility filter. Same result, less code.

Un-assignment is **not** listed here because it cannot happen by UI design. The defensive `!newAssignedTo` guard in the helper exists only to protect against direct API misuse.

---

## Open questions to settle during implementation

1. **How does `handleOrderReassignment` obtain `patientId` for the `notifyOrderOfRecentDocuments` call?** The `ReassignableOrder` interface today only exposes `_id` and `assignedTo`. Options:
   - **A** — extend `ReassignableOrder` to include `patient` (the route handler has the full order; widening the interface adds zero work at the call site).
   - **B** — fetch the order from the DB inside the helper to get `patient`. Adds a query for the cost of keeping the interface narrow. Less clean.
   - Recommendation: **A**. Pass the patient field straight through.

2. **`actorId` for the audit entries written by `notifyOrderOfRecentDocuments`** — its `auditSource` parameter accepts a default of `'system'`. For initial assignment fired from the route, this is a user action, so the helper should forward `auditSource: 'user'`. Whether to also forward an `actorId` (the user who performed the assignment) is the question — it adds a small parameter, but it captures who triggered the backfill in the resulting audit rows. Recommendation: forward `actorId` if the route already has `session.user?.id` handy at the call site (it does in MLID-2413's parallel pattern). If not trivially available, default to empty / system.

3. **Do we need a new SignalR event?** No. `order:reassigned { orderId }` is already broadcast at the end of both branches. Both client hooks (`useDocumentsTabNotifications` and `useOrderNotificationCounts`) already listen for it and refetch. The badge + chip surface will update without UI work.

Resolve each open question in the first implementation step and document the chosen path in the commit message.
