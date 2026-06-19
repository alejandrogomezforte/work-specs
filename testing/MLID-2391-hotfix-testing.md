# MLID-2391 hotfix — Data Testing: initial-assignment notification backfill

Working notebook for verifying the MLID-2391 hotfix (`fix/MLID-2391-initial-assignment-notification-backfill`). The fix makes `handleOrderReassignment` backfill notifications when an order transitions from `assignedTo: ""` to a populated assignee via `PUT /api/orders/new`.

- **Plan**: `docs/agomez/plans/MLID-2391-hotfix-initial-assignment-notification-backfill.md`
- **Postmortem (background)**: `docs/agomez/postmortem/MLID-2391-unassigned-to-assigned-skips-notification-backfill.md`
- **Branch**: `fix/MLID-2391-initial-assignment-notification-backfill`
- **Feature flag**: `ORDER_DOCUMENT_NOTIFICATIONS` must be ON locally to exercise the notification surface

## Bug fixed mid-session (2026-05-27)

First attempt against order `05Wu` failed (assignment persisted but 0 notifications). Root cause: the orchestrator passed `patientId: order.patient` to `notifyOrderOfRecentDocuments`, but after `updateOrder`'s aggregation pipeline runs, `patient` is a populated **object** (`{ _id, fullName, ... }`), not a string. The notification helper's downstream `getDocumentsByPatientId({ patientId: <object> })` matched zero rows → silent no-op.

Fix: added an `extractPatientId(patient)` helper that accepts either a string or a populated object with `_id`, mirroring the existing successful caller pattern at `app/api/orders/new/route.ts:158` (`patientId: String(createdOrder.patient._id)`). The `ReassignableOrder` interface's `patient` field now allows both shapes. 2 new orchestrator tests cover the populated-object and missing-patient cases.

---

## Fixture: order `05Wv` (`6a171b8b16035e07bc75199f`)

Right shape for the hotfix test:

- Order created at an IG-less location → `assignedTo: ""` at creation ✅
- 2 non-Order documents auto-attached at creation, both within the 30-day backfill window ✅
- 0 notifications written at creation time (the gap our fix closes) ✅

---

## Patient

| Field | Value |
|---|---|
| `_id` | `6a15dda68c4205d331775425` |
| Name | Alex Test (Gomez) |
| `we_infuse_id_IV` | `9562` |
| `we_infuse_location_id` | `252` |

Same patient used in MLID-2413 testing — the 3 patient documents created on 2026-05-26 17:51 UTC are all within the 30-day lookback window relative to this order's creation.

## Patient documents (3 attached to the patient)

| Document `_id` | `category` | `createdAt` | Auto-attached to this order? |
|---|---|---|---|
| `6a15dda76b5a32726f4b0be1` | Referral Copy | `2026-05-26T17:51:35.227Z` | **Yes** (non-Order) |
| `6a15dda86b5a32726f4b0bf1` | Other | `2026-05-26T17:51:36.126Z` | **Yes** (non-Order) |
| `6a15dda96b5a32726f4b0bf8` | Order | `2026-05-26T17:51:37.084Z` | No (Order-category only attaches via explicit link) |

---

## Snapshot 1 — pre-assignment baseline (current state)

> Captured 2026-05-27T16:27 UTC via MongoDB MCP. Order just created; nobody assigned yet.

### Order state (from `neworders`)

| Field | Value |
|---|---|
| `_id` | `6a171b8b16035e07bc75199f` |
| `displayId` | `05Wv` |
| `orderType` | New Start |
| `drug` | `6938863d9879dc15c6a1de64` |
| `site` | `67b7083a8532e12fdce9d624` (IG-less — that's why `assignedTo` ends up empty) |
| `provider` | `1710368154` (Dr. JOHN JOHN) |
| `providerPracticeFax` | `+12126627954` |
| `patient` | `6a15dda68c4205d331775425` |
| `received` | `2026-05-27T16:22:16.556Z` |
| `createdAt` / `lastUpdated` / `lastAssigned` | `2026-05-27T16:27:55.864Z` (all identical — no edit since creation) |
| `createdBy` / `updatedBy` / `assignedBy` | `695c25a530f7a75a8b1ed8c8` |
| `assignedTo` | `""` (**empty — exactly the case the hotfix targets**) |
| `audit` | `"Yes"` |

### Audit log (`auditLogs` for `6a171b8b16035e07bc75199f`): 3 entries

| # | `timestamp` (UTC) | `action` | `source` | `actorId` | `changes[0].field` | `changes[0].from` → `changes[0].to` |
|---|---|---|---|---|---|---|
| 1 | `2026-05-27T16:27:55.953Z` | `create` | `user` | `695c25a530f7a75a8b1ed8c8` | — | (order create event) |
| 2 | `2026-05-27T16:27:55.968Z` | `update` | `system` | `""` (empty) | `documents` | `null` → `{ _id: "6a15dda86b5a32726f4b0bf1", category: "Other", source: "upload" }` |
| 3 | `2026-05-27T16:27:55.997Z` | `update` | `system` | `""` (empty) | `documents` | `null` → `{ _id: "6a15dda76b5a32726f4b0be1", category: "Referral Copy", source: "upload" }` |

The 2 system `update` rows confirm both non-Order docs auto-attached at order creation.

### Notifications: **0**

`db.notifications` for `(order 6a171b8b16035e07bc75199f × document *)`: **0 rows**.

This is the gap. Because `assignedTo` was `""` when the auto-attach propagation ran, `writeOrderDocumentNotification` was gated out (it requires `assignedTo` truthy). The 2 audit rows (steps 2 and 3 above) are the propagation orchestrator's step-3 audit writes, which have no gate. The step-2 notification writes never happened.

### NotificationReads: **0**

No notifications exist, so no reads.

---

## Snapshot 2 — TBD (after assigning the order to yourself — the fix should fire)

> Run after Order Tracker → Order Details → assign to Alejandro Gomez FO. Make sure the dev server is running the fix branch and was restarted after the `extractPatientId` fix so the new code is loaded.

Expected diffs vs Snapshot 1:

- **`neworders`**: `assignedTo` populated with the new user ID, `lastAssigned` and `lastUpdated` advance.
- **AuditLogs**: gains a 4th entry — `action: update`, `source: user`, `actorId: 695c25a530f7a75a8b1ed8c8`, `changes: [{ field: 'assignedTo', from: '', to: 'Alejandro Gomez FO' }]`. (This audit is written by the route's audit pipeline, not by `handleOrderReassignment`.)
- **Notifications**: gains **2 rows** — one for each auto-attached non-Order document. **This is the fix in action.**
  - Notif A: `entities: [{ entityType: 'order', entityId: '6a171b8b16035e07bc75199f' }, { entityType: 'document', entityId: '6a15dda86b5a32726f4b0bf1' }]`, `title: "New Document: Other"`, `targetUserId: 695c25a530f7a75a8b1ed8c8`, `isRead: false`.
  - Notif B: same shape but for `6a15dda76b5a32726f4b0be1` (Referral Copy).
- **NotificationReads**: still 0.
- **UI**: the order's Documents tab badge shows `2`; both rows show the "New" chip + "Mark as Read" button when viewed by the assignee.

_<observations to fill in after the assignment>_

---

## Snapshot 3 — TBD (after a subsequent reassignment to a different user — sanity check the existing retarget path still works)

> Run after re-assigning the order to a different user via Order Tracker.

Expected diffs vs Snapshot 2:

- **`neworders`**: `assignedTo` updates to the second user's ID.
- **AuditLogs**: gains a 5th entry — `assignedTo from "Alejandro Gomez FO" to "<new user>"`.
- **Notifications**: same count (2), but `targetUserId` on both rows flips from the previous assignee to the new one. No new rows, no deletions — pure retarget via `reassignUserNotificationsForOrder`.
- **NotificationReads**: any reads from the previous assignee survive in the collection but stop participating in the unread join (the join keys on the notification's current `targetUserId`).
- **UI**: previous assignee no longer sees the badge / chips; new assignee sees both as unread.

_<observations to fill in after the reassignment>_

---

## Mongo Compass filters

**Order:**
```json
{ "_id": ObjectId("6a171b8b16035e07bc75199f") }
```

**Notifications for this order (any document):**
```json
{ "entities": { "$elemMatch": { "entityType": "order", "entityId": "6a171b8b16035e07bc75199f" } } }
```
Sort: `{ "createdAt": -1 }`

**Audit log for this order:**
```json
{ "entityId": "6a171b8b16035e07bc75199f" }
```
Sort: `{ "timestamp": -1 }`

> `auditLogs.entityId` is stored as a String (not ObjectId) — paste the order id without the `ObjectId(...)` wrapper.

---

## Pre-test checklist

- [ ] Dev server restarted **after the `extractPatientId` fix** (Next.js needs a fresh module load — hot-reload caches imports indirectly used by API routes).
- [ ] `ORDER_DOCUMENT_NOTIFICATIONS` feature flag is ON in `/admin/feature-flags`.
- [ ] Current git branch is `fix/MLID-2391-initial-assignment-notification-backfill` and the working tree includes both the orchestrator and route changes.
- [ ] Snapshot 1 above matches the live DB state (order `assignedTo: ""`, audit 3 entries, notifications 0).

If all four boxes check, the assignment action below will exercise the fix end-to-end.
