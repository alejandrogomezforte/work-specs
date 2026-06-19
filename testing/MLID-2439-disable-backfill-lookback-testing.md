# MLID-2439 hotfix — Data Testing: disable backfill lookback window

Working notebook for verifying the MLID-2439 hotfix (`fix/MLID-2439-disable-backfill-lookback`). The fix flips `BACKFILL_LOOKBACK_DAYS` from `30` to the sentinel `-1`, which makes `notifyOrderOfRecentDocuments` skip the `since` filter on `getDocumentsByPatientId` and surface **all** of the patient's documents to a new order regardless of age. The date-filter code path is preserved so the lookback can be re-enabled by flipping the constant back to a positive number.

- **Branch**: `fix/MLID-2439-disable-backfill-lookback`
- **Source change**: `apps/web/services/notifications/documentNotification.ts` — constant + comment + `since` guard.
- **PO direction (2026-05-27)**: turn the filter off for now, keep the logic.
- **Feature flag**: `ORDER_DOCUMENT_NOTIFICATIONS` must be ON locally.

## Why this patient is the right fixture

Patient `Lisa Test` (`69943672467fd1beb923814b`) has 9 documents, **all created in February 2026** — far outside the old 30-day lookback. Before this hotfix, creating a new order for this patient would have surfaced **zero** documents (all filtered out by `createdAt >= now - 30 days`). After this hotfix, all 9 patient docs become candidates for the propagator (the orchestrator's eligibility gate then filters Order-category docs that are not linked to the new order).

## Reference order `05VY` (soft-deleted — can't be used directly, kept for context)

> Captured 2026-05-27 via MongoDB MCP. The order was created on 2026-05-25 and soft-deleted today by QA. The UI hides soft-deleted orders, so you cannot interact with this one directly — use it as the *shape* reference; create a fresh order for the same patient to exercise the hotfix.

| Field | Value |
|---|---|
| `_id` | `6a14612b874a7800c4130a75` |
| `displayId` | `05VY` |
| `orderType` | New Start |
| `drug` | `693896d59879dc15c6a1de68` (lookup returned no doc in `drugs` collection — possibly removed or in a different collection) |
| `site` | `6a0414d036901907cf556615` |
| `provider` | `1265534150` |
| `providerPracticeFax` | `+19176755175` |
| `patient` | `69943672467fd1beb923814b` |
| `received` | `2026-05-25T14:46:40.371Z` |
| `createdAt` | `2026-05-25T14:48:11.091Z` |
| `lastAssigned` | `2026-05-25T14:48:21.114Z` (assigned ~10 seconds after creation) |
| `lastUpdated` | `2026-05-27T12:55:27.801Z` (the soft-delete timestamp) |
| `createdBy` / `updatedBy` / `assignedBy` | `681b9106b59e73da06c08486` |
| `assignedTo` | `686eb246db18cb88c3956078` (Olga Turovych — see audit row 2) |
| `deleted` | **`true`** (soft-deleted; hidden in UI) |
| `audit` | `"Yes"` |

### Audit log for `05VY` (3 entries)

| # | timestamp | action | source | actorId | changes |
|---|---|---|---|---|---|
| 1 | `2026-05-25T14:48:11.174Z` | `create` | `user` | `681b9106b59e73da06c08486` | (order create event) |
| 2 | `2026-05-25T14:48:21.201Z` | `update` | `user` | `681b9106b59e73da06c08486` | `assignedTo: "" → "Olga Turovych"` |
| 3 | `2026-05-27T12:55:27.881Z` | `delete` | `user` | `681b9106b59e73da06c08486` | (soft delete) |

### Notifications for `05VY`

`db.notifications` for `(order 6a14612b874a7800c4130a75 × document *)`: **0**.

Consistent with pre-hotfix behavior — the order was created on 2026-05-25, all of the patient's documents are from February 2026, the old 30-day lookback filtered them out, no notifications were ever written. Even though the order got assigned (audit row 2), the existing reassignment cascade only retargets pre-existing notifications, and there were none to retarget.

This is exactly the symptom MLID-2439 fixes.

---

## Patient

| Field | Value |
|---|---|
| `_id` | `69943672467fd1beb923814b` |
| Name | Lisa Test |
| `we_infuse_id_IV` | `8364` |
| `we_infuse_location_id` | `252` |
| DOB | 7 Jan 2026 (synthetic) |
| Sex | unknown |
| Preferred language | en |
| `treatment_authorized_date` | `2026-02-17T09:35:46.860Z` |

No email, phones, or insurance fields populated (test patient).

## Patient documents (9 total — **all created in February 2026**)

| # | `_id` | `category` | `description` | `createdAt` | Age vs 2026-05-27 |
|---|---|---|---|---|---|
| 1 | `699c47953eceb6aecfd2444a` | **Order** | Order | `2026-02-23T12:27:01.590Z` | **93 days old** |
| 2 | `699c47933eceb6aecfd24444` | Referral Copy | test | `2026-02-23T12:26:59.158Z` | 93 days |
| 3 | `69943678fb32e5cbf54033d1` | Other | Other - Pages 1-1 | `2026-02-17T09:35:52.152Z` | 99 days |
| 4 | `69943677fb32e5cbf54033cc` | Demographics | Demographics - Pages 1-1 | `2026-02-17T09:35:51.513Z` | 99 days |
| 5 | `69943676fb32e5cbf54033c3` | Lab Results | Lab Results - Pages 1-1 | `2026-02-17T09:35:50.929Z` | 99 days |
| 6 | `69943676fb32e5cbf54033be` | Prior Authorization | Insurance Authorization - Pages 1-1 | `2026-02-17T09:35:50.275Z` | 99 days |
| 7 | `69943675fb32e5cbf54033b9` | Medical Record | Medical Record - Pages 1-1 | `2026-02-17T09:35:49.426Z` | 99 days |
| 8 | `69943674fb32e5cbf54033b4` | Insurance Card | Insurance Card - Pages 1-1 | `2026-02-17T09:35:48.742Z` | 99 days |
| 9 | `69943673fb32e5cbf54033af` | Referral Copy | Full fax document - Fax | `2026-02-17T09:35:47.958Z` | 99 days |

Every document is from the same patient intake batch (intakeId `699424e378ddbae425e45676` for the 7 originals; `699c45e9784f80cf798db3b1` for the 2 later Order/Referral Copy entries).

### Documents that should propagate vs not

The propagator's eligibility gate (`isDocumentEligibleForOrder`) keeps the existing rule unchanged: **Order-category** documents propagate only when their `linkedOrderIds` includes the target order. For a freshly-created order, that's never true at backfill time. So:

| Eligibility | Count | Categories |
|:--:|:--:|---|
| Propagates | **7** | Demographics, Lab Results, Prior Authorization, Medical Record, Insurance Card, Other, 2× Referral Copy |
| Skipped (Order-category, not yet linked) | 2 | both `category: "Order"` docs (`699c47953eceb6aecfd2444a` + … wait, only one Order doc; row 2 above is Referral Copy) |

Recounting: row 1 is the only Order-category doc. Row 2 is `Referral Copy`. So **7 documents** are non-Order and will all propagate after the fix:

1. Referral Copy (row 2)
2. Other
3. Demographics
4. Lab Results
5. Prior Authorization
6. Medical Record
7. Insurance Card
8. Referral Copy (row 9, Fax variant)

That's **8** non-Order documents. The 1 Order-category doc (row 1) is skipped by the eligibility gate.

---

## Plan to verify the hotfix live

`05VY` is soft-deleted, so use this patient (`Lisa Test`) to create a NEW order at an IG-less location. That order will end up `assignedTo: ""`, and the patient will be `69943672467fd1beb923814b`.

### Snapshot 1 — pre-assignment baseline (after creating the new order)

Expected DB state:
- `neworders.assignedTo === ""` ✅
- `auditLogs` for the new order: **1 entry** (just the `create` row). Per the MLID-2391 hotfix's `assignedTo` gate (already on develop), no auto-attach audit rows are written when the order is unassigned.
- `notifications`: 0 (gate held).

### Snapshot 2 — post-assignment (this is where the MLID-2439 hotfix shows its effect)

Assign the new order to yourself via Order Tracker → Order Details. The route's existing `handleOrderReassignment` empty-to-populated branch (also on develop, from MLID-2391) will fire `notifyOrderOfRecentDocuments` for the backfill. With the lookback DISABLED, all 9 of the patient's documents become candidates.

Expected DB state:
- `neworders.assignedTo` populated, `lastAssigned` advanced.
- `auditLogs` for the new order: gains **9 new rows** total — 1 for the assignedTo change + **8 for the non-Order document attachments** (the eligibility gate skips the 1 Order-category doc).
- `notifications`: gains **8 new rows** (one per non-Order doc), all targeting the new assignee, all `isRead: false`.
- The badge counter on the order's Documents tab shows `8`.

> **Comparison to pre-hotfix behavior**: with the old 30-day lookback active, the backfill would have queried `getDocumentsByPatientId(patientId, { since: now - 30 days })`, found ZERO documents (all of Lisa's docs are >90 days old), and exited early. Zero new audit rows, zero notifications. That's the exact symptom on the soft-deleted `05VY` reference order — see its audit log above.

### Snapshot 3 — TBD (optional sanity check: unlink an Order-category doc)

Out of scope for this hotfix's primary verification, but if you want a parallel check that the rest of the system still works correctly with the lookback disabled, exercise the link/unlink flow on the new order using the Order-category doc `699c47953eceb6aecfd2444a`.

---

## Mongo Compass filters

**New order (replace ID once created):**
```json
{ "_id": ObjectId("<new-order-id>") }
```

**Notifications for the new order:**
```json
{ "entities": { "$elemMatch": { "entityType": "order", "entityId": "<new-order-id>" } } }
```
Sort: `{ "createdAt": -1 }`

**Audit log for the new order:**
```json
{ "entityId": "<new-order-id>" }
```
Sort: `{ "timestamp": -1 }`

**Patient documents (Lisa Test):**
```json
{ "patientId": "69943672467fd1beb923814b" }
```

> `auditLogs.entityId` is stored as a String (not ObjectId) — paste the order id without the `ObjectId(...)` wrapper. `documents.patientId` is also a String.

---

## Pre-test checklist

- [ ] Dev server restarted on the `fix/MLID-2439-disable-backfill-lookback` branch (so the `BACKFILL_LOOKBACK_DAYS: number = -1` change is loaded).
- [ ] `ORDER_DOCUMENT_NOTIFICATIONS` feature flag is ON in `/admin/feature-flags`.
- [ ] Create a fresh order for `Lisa Test` (`69943672467fd1beb923814b`) at an IG-less location — confirm `assignedTo: ""` and 1 audit row (`create` only) at this baseline.
- [ ] Assign the order to yourself; expect the 9 new entries summarized in Snapshot 2 above.
