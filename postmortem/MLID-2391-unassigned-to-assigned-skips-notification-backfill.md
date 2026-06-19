# Postmortem: Reassignment does not backfill notifications when previous `assignedTo` is empty

**Status:** Open — fix scheduled as a hotfix on a separate branch
**Date discovered:** 2026-05-26
**Environment:** Local (also affects develop / production — pre-existing behavior, not specific to MLID-2413)
**Related ticket:** MLID-2391 — `[D0-T1] Reassignment cascades notifications to new assignee (neworders)`
**Discovered during:** UI verification of the MLID-2413 fix branch (`fix/MLID-2413-document-unlink-notification-cleanup`) against fresh order `6a161fabd11fe1b093dc47da` (displayId `05WL`).

---

## Summary

When a new order is created in the **unassigned** state (`assignedTo: ""`) and then assigned to a user later through the normal `PUT /api/orders/new` reassignment flow, **no notifications are written for the documents that were auto-attached to the order at creation time**. The order-history audit entries for those auto-attachments DO exist, so the UI's history tab shows the documents — but the per-user badge / "New" chip surface stays at zero.

This is a gap in the MLID-2391 reassignment cascade design. The cascade was specified as a **retarget** of *existing* notifications from a previous assignee to a new one, not as a **backfill** when the order transitions from "no assignee" to "an assignee" for the first time.

---

## How it was discovered

Reproducing the MLID-2413 unlink fix in the local UI required a fresh order with no pre-existing linked Order-category document. The test session created order `6a161fabd11fe1b093dc47da` (displayId `05WL`) via the standard manual-creation flow, which produced `assignedTo: ""`. Two non-Order patient documents (`Referral Copy` and `Other`) auto-attached at creation time (confirmed by 3 `auditLogs` entries: one `create`, two `update` rows with `field: documents`). The expectation — based on the behavior observed on the earlier 05WF order — was that assigning the order to a user would produce 2 notifications (one per auto-attached document) for that user's badge surface.

After running Order Tracker → Order Details → assign-to-self:

| Surface | State |
|---|---|
| `neworders.assignedTo` | populated correctly (`695c25a530f7a75a8b1ed8c8`) |
| `neworders.lastAssigned` | updated to the assignment timestamp |
| `auditLogs` | gained a 4th entry: `assignedTo: "" → "Alejandro Gomez FO"` |
| `notifications` for `(order × document *)` | **0** — no rows written |
| `notificationreads` | **0** |

The audit trail confirms the assignment was recorded; the notification surface did not respond.

---

## Root cause

Two independent code paths conspire to produce the gap. Neither path is wrong in isolation — together they fail to cover the "initial assignment" case.

### Path 1 — auto-attach at order creation skips notification writes when unassigned

`propagateDocumentNotificationToOrder` in `apps/web/services/notifications/documentNotification.ts` runs four steps per `(order, document)` pair. Step 2 (`writeOrderDocumentNotification`) is gated on `order.assignedTo` being truthy — it writes a notification only when there is a user to target. Step 3 (`writeOrderDocumentAuditEntry`) has no such gate; it writes regardless.

When the manual `POST /api/orders/new` flow creates an order with `assignedTo: ""` and the route then runs `notifyOrderOfRecentDocuments` for the patient's recent documents:

- Each non-Order document fires the propagation orchestrator.
- Step 2 sees `assignedTo` empty → returns without inserting into `notifications`.
- Step 3 still writes the audit row.
- Step 4 broadcasts `documentAdded` (clients refetch, but the by-order endpoint returns 0 unread for an unassigned order, so the badge stays empty).

Net effect at order-creation time: audit entries exist, notifications do not.

### Path 2 — `handleOrderReassignment` early-returns when `previousAssignedTo` is empty

`handleOrderReassignment` in `apps/web/services/notifications/handleOrderReassignment.ts` is the helper called from `PUT /api/orders/new` after `assignedTo` changes. Its guard clause at the top:

```typescript
if (!previousAssignedTo || !newAssignedTo) {
  return;
}
```

When the assignment transitions from `""` → `"user-1"`, `previousAssignedTo` is `""` (falsy) and the helper exits before doing anything. Even without that early return, the helper's internal call is `reassignUserNotificationsForOrder`, which runs `Notification.updateMany(...)` to **flip `targetUserId` from previous to new** — a retarget operation against existing rows. With zero pre-existing notifications for the `(order, *)` tuple, the retarget has nothing to flip.

### Combined effect

- Auto-attach time: `assignedTo` empty → no notifications written.
- Assignment time: `previousAssignedTo` empty → reassignment helper returns immediately, and the underlying `updateMany` would not have created anything anyway.
- Result: the order has documents the assignee should see badge-counted, but `notifications` contains zero matching rows.

---

## Scope of impact

Affected: every order whose creation produces `assignedTo: ""` and which is later assigned to a user through the reassignment flow. From the audit log evidence on 05WL, this is at minimum the standard "Add new order" UI flow that does not auto-assign. (Orders created via paths that set `assignedTo` at creation — for example the "Auto Assigned" route observed on order 05WF — are unaffected because step 2 of propagation fires with `assignedTo` populated.)

Not affected: future user-driven actions on the now-assigned order. After assignment, linking an Order-category document via the link-orders route fires `handleOrderLink` with the correct `assignedTo`, and a notification IS written for the link. The gap is strictly for the documents that auto-attached **before** the assignee was set.

Severity: the assignee sees the documents listed in the order's Documents tab (the tab listing comes from the order document itself, not from notifications), but the unread-count badge and per-row "New" chip do not appear. From an end-user perspective the order looks "already-read" without ever having been read.

---

## Why MLID-2391 did not catch this

MLID-2391 was scoped as a retarget cascade for the previous-to-new assignee transition. The acceptance criteria and tests cover:

- Previous assignee populated → new assignee populated (standard reassignment).
- `previousAssignedTo === newAssignedTo` early return (no-op).
- `previousAssignedTo` empty → early return (this case was treated as "out of scope" because the helper exists to *move* notifications, and there are no notifications to move when nothing was previously assigned).
- Un-assignment (`assignedTo` cleared) was explicitly listed as out of scope in the MLID-2391 plan.

What was not on the criteria list: the case where notifications **should** exist (because non-Order documents auto-attached at creation) but **do not** exist (because the assignedTo gate skipped them). The orchestrator design assumed notifications are always written at the time the document arrives — which is true when `assignedTo` is populated at that moment. The interaction with the manual-creation flow's "create unassigned, assign later" pattern was never exercised in the MLID-2391 tests.

---

## Proposed fix (for the future hotfix branch)

The hotfix needs to add a **backfill** code path triggered specifically when `previousAssignedTo` was empty and `newAssignedTo` is non-empty. Two design choices to settle when picking this up:

### Choice 1 — What to backfill

- **Option A — Backfill from the order's audit log.** Read the `field: 'documents'` audit entries for this order (the ones written by step 3 at auto-attach time) and for each entry whose `to._id` corresponds to a document still attached to the order, fire the propagation orchestrator to write the missing notification. Authoritative because the audit log is the record of what auto-attached.
- **Option B — Reuse `notifyOrderOfRecentDocuments`.** Call it again at assignment time. This re-walks the last 30 days of the patient's documents and fires propagation for each. Slightly wider than strictly necessary (could fire for documents the user did not actually auto-attach to this order), but the eligibility gate inside the orchestrator (`isDocumentEligibleForOrder`) should filter correctly.
- **Option C — New helper.** Introduce `backfillOrderNotificationsOnFirstAssignment(order)` that reads `(documents where this order is in linkedOrderIds) OR (documents where the patient matches and category is not Order)` and fires propagation per document. Most precise scope.

Recommendation: start with Option B if it is shown to be safe in practice (the orchestrator's eligibility gate is doing the right thing today for the post-creation backfill on assigned orders). Fall back to Option A or C if Option B over-fires.

### Choice 2 — Where to wire the trigger

The natural site is `handleOrderReassignment` itself: detect the empty-to-non-empty case and branch to the backfill path instead of the retarget path. Keep both behaviors inside the same orchestrator so callers (`PUT /api/orders/new` today, the maintenance order reassignment route in D4-T5 / MLID-2454 later) don't need to grow new branches.

Sketch:

```typescript
export async function handleOrderReassignment(
  order: ReassignableOrder,
  previousAssignedTo: string,
  newAssignedTo: string
): Promise<void> {
  // ... existing feature-flag check, equality guard ...

  if (!newAssignedTo) {
    return; // un-assignment still out of scope
  }

  if (!previousAssignedTo) {
    // NEW: backfill path — no notifications exist to retarget, fire propagation
    // for the documents that auto-attached prior to assignment.
    await backfillOrderNotificationsOnFirstAssignment(order);
    await broadcastReassigned(order._id);
    return;
  }

  // existing retarget path (unchanged)
}
```

### Choice 3 — Tests

- New `handleOrderReassignment` test cases: empty → populated triggers backfill, never retarget; populated → populated triggers retarget, never backfill; populated → populated identical user is a no-op (existing).
- New integration test on `PUT /api/orders/new`: create order unassigned with auto-attached non-Order docs, assign to a user, confirm notifications appear in `notifications` with the correct `targetUserId`.
- Regression: MLID-2391's existing retarget tests must still pass unchanged.

---

## Action items (for the future hotfix branch)

- [ ] Create branch `fix/MLID-2391-initial-assignment-notification-backfill` (or whichever Jira key is selected — this may warrant its own ticket separate from MLID-2391 since MLID-2391 is already merged to develop).
- [ ] Pick a design between Choice 1 Option A / B / C above (recommend B unless a concrete over-fire is identified).
- [ ] Implement the empty-to-non-empty branch inside `handleOrderReassignment`.
- [ ] Add the test coverage outlined in Choice 3.
- [ ] Verify on staging with a fresh manually-created order: create unassigned → confirm zero notifications → assign → confirm notifications appear for the auto-attached non-Order docs → confirm badge / "New" chip surface for the assignee.
- [ ] Update the MLID-2417 epic plan progress's "Shared helpers" table entry for `handleOrderReassignment` to reflect the dual retarget-or-backfill responsibility.
