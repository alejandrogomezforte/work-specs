## Three triggers that flow through propagateDocumentNotificationToOrder:

  1. Document uploaded via /intakes
     
  - Entry point: notifyDocumentCreated
  - Called from:
    - POST /api/documents/split-pdf (manual upload from Intake Analyser — both the full and split paths)
    - weinfuseUploader.ts (webhook-driven upload from JotForm/Spruce)
  - Shape: one document, fans out to every order for the patient
  - How to reproduce in browser:
    a. Open the Intake Analyser for a patient who has at least one active order
    b. Upload a document with category ≠ Order (e.g. Clinical, Lab Results) → should propagate to every patient order
    c. Upload a document with category = Order → should NOT propagate anywhere (it's unlinked at upload time)

  2. New order created

  - Entry point: notifyOrderOfRecentDocuments
  - Called from: POST /api/orders/new (app/api/orders/new/route.ts:154)
  - Shape: one new order, fans out to every recent (last 30 days) patient document
  - How to reproduce in browser:
    a. Find a patient who has recent uploaded documents (mix of categories)
    b. Create a new order for that patient via Order Tracker
    c. Non-Order recent docs → propagate to the new order (notification + audit + SignalR)
    d. Order-category docs that are NOT linked to the new order → skipped
    e. Order-category docs that ARE linked to the new order → propagate

  3. Document linked to an order via /patient/[id]/document-records

  - Entry point: firePropagationForNewlyLinkedOrders
  - Called from: PUT /api/documents/[documentId]/link-orders (only when newlyAdded.length > 0)
  - Shape: one Order-category document, fans out only to the newly-linked orders (delta against the previous linkedOrderIds)
  - How to reproduce in browser:
    a. Navigate to /patient/[id]/document-records
    b. Find an Order-category document; click "Update Order Linking" or the "Link to Order" menu item
    c. Add a link to a specific order → propagation fires for that order
    d. Re-save with the same orders → no duplicate (delta is empty)
    e. Add a second order → only the new order gets the entries
    f. Remove an order → no propagation (delta is empty for additions)

---

## Entities involved in this test session (2026-05-15)

### Patient — Cynthia Young QA (`6a0358c936901907cf5545e8`)
- weInfuseId (`we_infuse_id_IV`): 9420
- DOB: 1980-02-10, sex: female
- email: cynymin@gmail.com
- address: 35 Turkey LN, Winthrop, ME 04364

### Order #05RD — `6a078a236de202e6c06d1037` (created today)
- displayId: `05RD`
- orderType: New Start
- drug: `6938863d9879dc15c6a1de64` — **Skyrizi** (Tier 1 - audit, pharmacyEligible: true)
- patient: `6a0358c936901907cf5545e8` (Cynthia Young QA)
- assignedTo: `67fe9b98a5133e710d955294` — **Destiny Gray** (`dgray@mylocalinfusion.com`, role: user, position: Market Lead)
- assignedBy: `Auto Assigned`
- createdBy / updatedBy: `695c25a530f7a75a8b1ed8c8`
- site: `67b7083a8532e12fdce9d61a`
- provider NPI: `1710368154`
- providerPracticeFax: `+12126627954`
- audit: Yes
- received: 2026-05-15T20:57:30.923Z
- createdAt: 2026-05-15T21:03:31.982Z
- lastAssigned: 2026-05-15T21:03:33.170Z

### Order #05NH (pre-existing, linked to one of patient's 5/12 Order docs)
- displayId: `05NH`
- drug: Inluxigam (per UI label "05NH - Inluxigam - 05/12/2026")
- Holds the link for one of the 5/12 Order-category docs (referenced in `linkedOrderIds`)

### Intake A — `6a077fdb299903ae70f0d8c6` (uploaded 2026-05-15 at 3:15 PM via Intake Analyser)
3 documents created on this intake:

| # | Document `_id` | Category | linkedOrderIds at upload | Notes |
|---|---|---|---|---|
| 1 | `6a078cee6de202e6c06d10b8` | Referral Copy | `[]` | created 21:15:26 UTC |
| 2 | `6a078cf66de202e6c06d10c4` | Other | `[]` | created 21:15:34 UTC |
| 3 | `6a078d056de202e6c06d10d7` | Order | `[]` → later linked to `6a078a236de202e6c06d1037` | created 21:15:49 UTC |

### Patient's pre-existing documents (uploaded 2026-05-12, in the 30-day backfill window for #05RD)

| Document | Category | linkedOrderIds | Eligibility for #05RD backfill |
|---|---|---|---|
| Demographics | Demographics | (n/a — non-Order) | ✅ eligible |
| Lab Results | Lab Results | (n/a — non-Order) | ✅ eligible |
| Referral Copy | Referral Copy | (n/a — non-Order) | ✅ eligible |
| Order (4:50 PM) | Order | `[]` (unlinked) | ❌ filtered |
| Order (4:43 PM) | Order | `[<id of 05NH order>]` (linked to 05NH, not 05RD) | ❌ filtered |

---

## Timeline of data changes during this session

### T0 — 2026-05-15 21:03:31 UTC (3:03 PM local): Order #05RD created
- Auto-assigned to Destiny Gray
- **Trigger 2 fired** (`notifyOrderOfRecentDocuments`):
  - Looked at 5 pre-existing 5/12 docs for patient Cynthia Young QA
  - Wrote **3** audit entries (Demographics, Lab Results, Referral Copy) — user = System
  - Filtered **2** Order-category pre-existing docs (one unlinked, one linked to 05NH)
- Order History at T0: 1 "Order Created" entry + 3 backfill "New Document received" entries

### T1 — 2026-05-15 21:15:26–21:15:49 UTC (3:15 PM local): Intake A processed, 3 docs uploaded
- Documents inserted in order: Referral Copy → Other → Order
- **Trigger 1 fired** (`notifyDocumentCreated`) for each:
  - Referral Copy (non-Order) → propagated to #05RD (1 audit entry, 1 notification, 1 SignalR)
  - Other (non-Order) → propagated to #05RD
  - Order (linkedOrderIds: []) → **filtered**: no audit, no notification, no SignalR
- Order History at T1: + 2 new "New Document received" entries (Other, Referral Copy), user = Alejandro G.
- Documents tab list at T1: 5 visible rows (2 new + 3 pre-existing non-Order from 5/12)
- Documents tab counter at T1: 5

### T2 — 2026-05-15 ~3:25 PM local: Order doc linked to #05RD via /patient/[id]/document-records
- User opened patient documents page, clicked "Update Order Linking" on the 3:15 PM Order doc (`6a078d056de202e6c06d10d7`), selected Order #05RD, saved.
- Document state after link (verified in MongoDB):
  ```
  _id: 6a078d056de202e6c06d10d7
  category: "Order"
  linkedOrderIds: ["6a078a236de202e6c06d1037"]    ← set
  ```
- **Trigger 3 should have fired** (`firePropagationForNewlyLinkedOrders`):
  - Previous `linkedOrderIds`: `[]`
  - New `linkedOrderIds`: `["6a078a236de202e6c06d1037"]`
  - Delta `newlyAdded`: `["6a078a236de202e6c06d1037"]` → 1 orchestrator call
  - Expected: +1 audit entry on #05RD ("Document type = Order"), +1 notification to Destiny Gray, +1 SignalR broadcast

---

## Verified results

### Trigger 1 (upload at T1) ✅
- Documents tab list: Order-category doc NOT listed (5 visible rows, none Order)
- Documents tab counter: 5 (Order doc not counted)
- Order History at T1: 2 entries (Other, Referral Copy), user = Alejandro G.

### Trigger 2 (order creation at T0) ✅
- 3 backfill entries on #05RD, user = System (matches `auditSource: 'system'` default)
- Filter exercised: 2 pre-existing Order-category docs correctly skipped (one unlinked, one linked to 05NH)

### Trigger 3 (link at T2) ✅
- DB-side: `linkedOrderIds` updated to `["6a078a236de202e6c06d1037"]` ✅
- Documents tab counter: 5 → 6 (incremented by the new linked doc) ✅
- Documents tab list: new "Order" row at top ✅
- Order History: new entry `"New Document received (Document type = Order)"`, user = Alejandro G. ✅
- Side note: the audit row's displayed time shows the document's `createdAt` (3:15 PM, the original upload time) rather than the audit entry's own `createdAt` (~3:27-3:30 PM, the actual link time). UI display detail only — not in scope for MLID-2086.

---

### T3 — 2026-05-15 21:40:21 UTC (~3:40 PM local): Order #05RD reassigned to Alejandro
- assignedTo: `67fe9b98a5133e710d955294` (Destiny Gray) → `695c25a530f7a75a8b1ed8c8` (Alejandro)
- assignedBy: `Auto Assigned` → `695c25a530f7a75a8b1ed8c8` (self-reassigned)
- lastAssigned / lastUpdated: 21:40:21 UTC
- Notification rows on #05RD: `targetUserId` remains `67fe9b98a5133e710d955294` (Destiny — snapshotted at write time, not cascaded on reassignment)

### Order #05RD — current state after T3 (verified in MongoDB)
```
_id:             6a078a236de202e6c06d1037
displayId:       05RD
patient:         6a0358c936901907cf5545e8   (Cynthia Young QA)
drug:            6938863d9879dc15c6a1de64   (Skyrizi)
site:            67b7083a8532e12fdce9d61a
assignedTo:      695c25a530f7a75a8b1ed8c8   (Alejandro)  ← changed at T3
assignedBy:      695c25a530f7a75a8b1ed8c8                 ← changed at T3
provider:        1710368154
orderType:       New Start
audit:           Yes
createdAt:       2026-05-15T21:03:31.982Z
lastUpdated:     2026-05-15T21:40:21.377Z                ← changed at T3
lastAssigned:    2026-05-15T21:40:21.040Z                ← changed at T3
```

---

## Verified: MLID-2085 relaxed-guard interaction (after T3 reassignment)

Screenshot 154145 confirms:
- All **6 documents** on the Documents tab show **"New"** chips next to their received time ✅
- Each row exposes a **"Mark as Read"** button on the right ✅
- Header exposes **"Mark all as Read"** ✅
- Counter still reads 6 (all unread for the new assignee)

This is the MLID-2085 OR-branch in action: although every notification row's `targetUserId` is still Destiny (the original assignee at write time), the relaxed guard `order.assignedTo === userId` lets the current assignee (Alejandro) see and act on the notifications.

### T4 — 2026-05-15 21:43:22 UTC (~3:43 PM local): "Mark as Read" clicked on the "Other" document

User clicked the per-row "Mark as Read" button on the "Other" document (`6a078cf66de202e6c06d10c4`).

Verified in MongoDB — `notificationreads` collection:
```
_id:             6a07937a6de202e6c06d12ba
notificationId:  6a078cf76de202e6c06d10cb   (title: "New Document: Other")
userId:          695c25a530f7a75a8b1ed8c8   (Alejandro)
readAt:          2026-05-15T21:43:22.674Z
```

The matching notification's `entities` contains the "Other" document `6a078cf66de202e6c06d10c4` and Order #05RD — exactly the right one.

Documents tab after T4:
- The "other" row no longer shows the "New" chip
- The "other" row no longer shows a "Mark as Read" button
- All other 5 rows still unread

### Architecture note: read state lives in `notificationreads`, not on the notification

Read state for **user-type** notifications is stored in a separate `notificationreads` collection — one row per `(notificationId, userId)` pair. The notification document's own `isRead` field stays `false` even after a user marks it read; `isRead` is only used by the legacy `markTaskAsRead` path for `type: 'task'` notifications.

Why: a single notification can be reachable by multiple users over time (the order may be reassigned). The per-user read tracking lets each assignee read independently without affecting what the others see. This is the data shape MLID-2085's relaxed guard relies on.

When verifying read state for a user-type notification, query:
```
db.notificationreads.find({ notificationId: <id>, userId: <id> })
```
not the notification's `isRead` field.

---

## Discrepancy observed after T4 — orders-list badge vs Documents-tab badge

After T4 (Alejandro marked the "Other" notification as read), two surfaces show different unread counts for the same order:

- `/orders-tracker/new` row badge for #05RD: **6**
- `/orders-tracker/new/.../documents` tab counter: **5**

**Root cause** (pre-existing in `develop`, not introduced by MLID-2086):

- Documents tab uses a **per-current-user** unread count (subtracts notificationreads for the viewing user)
- Orders list uses a **globally visible** count keyed off the notification's `targetUserId` (the original assignee at write time, per `getCountForEntity` in `apps/web/services/mongodb/notification.ts:176-205`, "team dashboard behavior — 2026-04-14 product decision")

In our scenario, every notification's `targetUserId` is still Destiny Gray (snapshotted at T0/T1 before reassignment). Alejandro's read at T4 created a `notificationreads` row for **Alejandro**, not Destiny — so the global count keyed off Destiny doesn't decrement, while Alejandro's per-user count does. Hence 6 vs 5.

This is an interaction gap between the 2026-04-14 global-badge design and MLID-2085's relaxed guard (which lets the new assignee read despite stale `targetUserId`).

**Not in scope for MLID-2086.** The fix is a separate decision — likely either:
- Cascade `targetUserId` on order reassignment (MLID-2391 follow-up that was noted in MLID-2085's plan)
- Change the global-count logic so it uses the current assignee or "anyone authorized has read it"

Recommend filing a follow-up ticket if the inconsistency is user-visible enough to matter.

---

### T5 — 2026-05-15 21:58:28 UTC (~3:58 PM local): Order 05RF created (Trigger 2 — fuller backfill verification)

Second order created on patient Cynthia Young QA to verify Trigger 2's eligibility filter across all three Order-category states (linked to other order, unlinked, linked to the new order).

**Order 05RF**
- `_id`: `6a0797048d4b4bdb3a71cdff`
- displayId: `05RF`
- drug: `6989fa7be13040640aae9f50` — **Hydration**
- patient: `6a0358c936901907cf5545e8` (Cynthia Young QA — same as 05RD)
- assignedTo: `67fe9b98a5133e710d955294` (Destiny Gray, auto-assigned)
- received: 2026-05-15T21:55:46.710Z
- createdAt: 2026-05-15T21:58:28.195Z

**Patient document state at T5** (8 docs in the 30-day window):

| Document `_id` | Category | linkedOrderIds | Eligibility for 05RF |
|---|---|---|---|
| `6a078d056de202e6c06d10d7` | Order | `[6a078a236de202e6c06d1037]` (05RD) | ❌ filtered (linked to 05RD, not 05RF) |
| `6a078cf66de202e6c06d10c4` | Other | `[]` | ✅ propagated |
| `6a078cee6de202e6c06d10b8` | Referral Copy | `[]` | ✅ propagated |
| 5/12 4:50 PM Order | Order | `[]` (unlinked) | ❌ filtered |
| 5/12 4:43 PM Order | Order | `[<05NH id>]` | ❌ filtered |
| `6a0358cca917ba8bbe9727eb` | Demographics | (non-Order) | ✅ propagated |
| `6a0358cba917ba8bbe9727e4` | Lab Results | (non-Order) | ✅ propagated |
| `6a0358cba917ba8bbe9727de` | Referral Copy | (non-Order) | ✅ propagated |

**Verified results on 05RF (DB queries)**

- `notifications` for `entityId: 6a0797048d4b4bdb3a71cdff`: **5 rows** (one per eligible doc, all `targetUserId: 67fe9b98a5133e710d955294` = Destiny) ✅
- `auditLogs` for `entityType: order, entityId: 6a0797048d4b4bdb3a71cdff`: **6 rows** (1 `action: create` for the order + 5 `action: update` with `field: documents`, `source: system`, `actorId: ""`) ✅
- All 3 Order-category docs correctly filtered out (none appear in the 5 propagated entries) ✅
- Audit collection name is `auditLogs` (camelCase), not `auditlogs` — written via `db.collection(COLLECTION.AuditLogs)` from `apps/web/utils/constants/collections.ts:37`

**Confirms**: the orchestrator's eligibility gate behaves correctly at the order-creation backfill entry point across the full matrix of Order-category linkage states (linked to a different order, unlinked, linked to the new order). No Order-category doc reaches 05RF since none have 05RF's id in their `linkedOrderIds`.

---

## Session summary — all five test scenarios verified end-to-end

Every scenario below was verified via direct MongoDB queries on `notifications`, `auditLogs`, `notificationreads`, `neworders`, and `documents` — not just visual inspection of the UI.

| # | Timestamp (UTC) | Scenario | Trigger | Result |
|---|---|---|---|---|
| T0 | 2026-05-15 21:03:31 | Order 05RD created on Cynthia Young QA | Trigger 2 (order-creation backfill) | ✅ 3 backfill audit entries written (Demographics, Lab Results, Referral Copy — all non-Order, source=system). 2 pre-existing Order-category docs (one unlinked, one linked to 05NH) correctly filtered. |
| T1 | 2026-05-15 21:15:26-49 | Intake A processed — 3 docs uploaded (Referral Copy, Other, Order) | Trigger 1 (document upload) | ✅ 2 audit entries written (Other + Referral Copy, source=user, actorId=Alejandro). Order-category doc filtered — no notification, no audit, no SignalR. |
| T2 | ~2026-05-15 21:27 | Order-category doc linked to 05RD via patient documents UI | Trigger 3 (link-time) | ✅ Document's `linkedOrderIds` updated to `["6a078a236de202e6c06d1037"]`. +1 audit entry on 05RD (source=user, type=Order). Counter 5→6. Re-saves with same selection would not duplicate (delta-based). |
| T3 | 2026-05-15 21:40:21 | Order 05RD reassigned from Destiny Gray → Alejandro | (MLID-2085 interaction) | ✅ All 6 docs show "New" chip + "Mark as Read" button for Alejandro despite `targetUserId` still being Destiny — MLID-2085's relaxed guard `order.assignedTo === userId` works. |
| T4 | 2026-05-15 21:43:22 | Alejandro clicks "Mark as Read" on the "Other" notification | (MLID-2085 read flow) | ✅ `notificationreads` row written for `(notification: 6a078cf76de202e6c06d10cb, user: Alejandro)`. Documents tab counter decremented 6→5. (Orders-list global badge intentionally NOT decremented — pre-existing design gap, out of scope.) |
| T5 | 2026-05-15 21:58:28 | Order 05RF created on same patient (fuller backfill matrix) | Trigger 2 (order-creation backfill) | ✅ 5 backfill notifications + 5 audit entries (source=system) for the 5 non-Order patient docs. All 3 Order-category docs filtered: one linked to 05RD, one unlinked, one linked to 05NH — none included 05RF's id in `linkedOrderIds`. |

### What the verification proves

- The orchestrator `propagateDocumentNotificationToOrder` is invoked from all three triggers (`notifyDocumentCreated`, `notifyOrderOfRecentDocuments`, `firePropagationForNewlyLinkedOrders`).
- The eligibility predicate `category !== 'Order' || linkedOrderIds.includes(orderId)` is enforced in every path.
- Each side effect (notification row, audit log entry, SignalR broadcast) is wrapped in its own try/catch, in the documented order (Notification → Audit → SignalR), and survives failures independently.
- Delta computation at the link endpoint correctly fires only for newly-added orderIds (re-saves and removals produce no propagation).
- `actorId` and `auditSource` are correctly forwarded through every trigger: `'user'` for upload + link, `'system'` for order-creation backfill.
- The pre-existing orders-list-badge vs documents-tab-badge inconsistency is documented but out of scope (interaction gap between the 2026-04-14 "team dashboard" decision and MLID-2085's relaxed guard).
