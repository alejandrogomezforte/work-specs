# MLID-2306 — Data Testing: Order Details Auth Review

Working notebook for end-to-end verification of the Auth Review sub-tab on Order
Details, backed by the existing `intakes` (prior-auth) collection and scoped to a
specific order by decision date and drug.

- **Branch:** `feature/MLID-2306-order-auth-review`
- **Plan:** `docs/agomez/plans/MLID-2306-order-auth-review.md` (5 stages)
- **Feature flag:** `ORDER_DETAILS_AUTH_REVIEW` (default `false`)
- **Also requires (layout gate):** `ORDER_DETAILS_ACTIONS` + `CLINICAL_REVIEWS` ON,
  same as the other order-detail review tabs.
- **API routes (Stage 2):**
  - `GET /api/orders/new/[orderId]/auth-review` (list)
  - `GET /api/orders/new/[orderId]/auth-review/filter-options`

---

## How the order scope works (important for picking test data)

A prior-auth document appears in an order's Auth Review list only when **both**
hold:

1. **Date:** effective decision date `>=` the order's `createdAt`. The effective
   date is OCR `additionalInfo.ocrData.authorization.dateOfDecision.value`
   (a `YYYY-MM-DD` string), falling back to the intake's own `createdAt` when
   absent or unparseable. In plain terms: the PA decision happened on or after
   the order was created.
2. **Drug:** `additionalInfo.priorAuthData.liDrugId.value` equals the order's
   `drug` id, **or** the document has no `liDrugId` (un-reviewed documents pass
   through the null/missing branch — this is the common case before MLID-2363).

Consequence: an order created **earlier** captures **more** documents. To see a
patient's full document history, the order must be created on or before the
earliest document's effective date.

> **Bug fixed during Stage 2 testing (2026-06-03):** the order-scoped drug match
> used `$in: ['$…liDrugId.value', [null, '']]`, but MongoDB's `$in` does **not**
> match a *missing* field path against `null`. Un-reviewed documents have
> `liDrugId` entirely absent (not null), so they were silently excluded — every
> test order showed zero rows. Fixed by coalescing first:
> `$in: [{ $ifNull: ['$…liDrugId.value', null] }, [null, '']]`.
>
> **Data reality:** there are **no** orders in this dataset with a *positive*
> drug match (`liDrugId == order.drug`) that also satisfy the date condition —
> drug-matched documents always predate their orders. Every working test order
> below renders rows via the null/missing-`liDrugId` branch.

---

## Pre-flight

- [ ] `npm run dev:web` running
- [ ] `ORDER_DETAILS_AUTH_REVIEW` toggled **ON** in `/admin/feature-flags`
- [ ] `ORDER_DETAILS_ACTIONS` and `CLINICAL_REVIEWS` toggled **ON**
- [ ] MongoDB MCP connected to local dev DB (`local-infusion-db`)

---

## Verified test orders (render ≥1 Auth Review row)

> **Stale counts (2026-06-04):** the row counts in the tables below were measured
> under the *old* show-everything filter (before the drug-scope rule landed in
> commit `ca7ba91b0`). With the drug filter, counts are now **lower** — documents
> whose drug name does not match the order's drug (and that have a brand/generic
> name) are correctly excluded; only hard `liDrugId` matches, fuzzy name matches,
> and truly info-less documents remain. Re-measure before relying on a specific
> number. Example: order 05DC (Panzyga) dropped from 13 to ~6.

These existing, non-deleted orders were found by simulating the order-scoped
query in MongoDB. Open at `/orders-tracker/new/<_id>`.

### Best for filters / pagination — patient WeInfuse `8081` (`6989d559e2d8c6414306d17b`)

These were created in mid-February, so 12–13 documents fall into scope, spanning
all five letter types (Approval, Denial, Receipt, Request, Other).

| displayId | order `_id` | created | order drug `_id` | rows in scope |
|-----------|-------------|---------|------------------|---------------|
| 05DF | `698db733a52e0630a255ea85` | 2026-02-12 | `698cd21def01c5ef8e60eefb` | 13 |
| 05DC | `698db48da52e0630a255ea82` | 2026-02-12 | `6938992b9879dc15c6a1de6e` | 13 |
| 05D0 | `698cd5d41bc5c7f23c0ed20c` | 2026-02-11 | `698c536eef01c5ef8e6059b9` | 13 |
| 05Cz | `698cd5bd1bc5c7f23c0ed20b` | 2026-02-11 | `698cd21def01c5ef8e60eefb` | 13 |
| 05DH | `698ee13c11c39d4f5185a92e` | 2026-02-13 | `698c536eef01c5ef8e6059b9` | 12 |

### Original test patient WeInfuse `7719`

The May orders previously listed showed nothing (the only in-scope doc was being
dropped by the `$in` bug). Use these earlier February orders instead — each
renders 5 documents (Approval + Denial).

> Note: WeInfuse `7719` maps to **two** distinct patient `_id` records, and the
> Auth Review list scopes by WeInfuse id (`patient.patientId == "7719"`), so both
> see the same document pool. The "Patient `_id`" column below shows which record
> each order belongs to.

| displayId | order `_id` | created | order drug `_id` | Patient `_id` | rows in scope |
|-----------|-------------|---------|------------------|---------------|---------------|
| 05Db | `6998908dd31cfd87f5d3b515` | 2026-02-20 | `6938990b9879dc15c6a1de6d` | `6971e73af002e9d03851a4bd` | 5 |
| 05Da | `69988e00d31cfd87f5d3b514` | 2026-02-20 | `6938990b9879dc15c6a1de6d` | `6971e73af002e9d03851a4bd` | 5 |
| 05Df | `699c1cafa541f0bdd44205be` | 2026-02-23 | `6938990b9879dc15c6a1de6d` | `6971e8ea1522a6fbea4865c1` | 5 |
| 05DX | `699748b8a006745f96e50fd9` | 2026-02-19 | `69389a1f9879dc15c6a1de6f` | `6971e8ea1522a6fbea4865c1` | 5 |

> Soft-deleted orders are excluded everywhere (`deleted: { $ne: true }`), and
> `getNewOrderById` does the same, so a deleted order returns 404 from the Auth
> Review routes. The earlier May orders for patient `7719` (e.g. 05YO, 05NG,
> 05Md, 05JH, 05JE) each have only a single late-May document in scope and are no
> longer the recommended test targets.

---

## Stage 1 — Feature flag foundation

Not data-testable on its own (enum entry + default value only). Covered by unit
tests; no manual data verification needed.

---

## Stage 2 — Auth Review list view (tab + DataGrid + filters + preview)

End-of-stage testable outcome: enable the flag, open an order, confirm the Auth
Review tab appears, the list loads scoped to that order, filters/chips/clear-all
work, and the preview modal opens.

**Test order used:** 05DF (`698db733a52e0630a255ea85`, patient `8081`) — 13 rows,
all five letter types. Good for filters/chips/pagination. _(adjust as needed)_

| Check | Result | Notes |
|---|---|---|
| Auth Review tab appears between Clinical Reviews and Financial Review | ☐ | |
| List loads, scoped to the order (date + drug) | ☐ | |
| "All drugs" / "All types" / "All statuses" / "All payors" filters work | ☐ | |
| Active filter chips appear and remove individually | ☐ | |
| "Clear all filters" resets the list | ☐ | |
| Sorting works | ☐ | |
| Pagination works | ☐ | |
| Preview icon opens the document viewer modal | ☐ | |
| Received cell links to `.../auth-review/[letterId]` (404 until Stage 3) | ☐ | Detail route is Stage 3 |

---

## Stage 3 — Detail screen: header + Step 1 + single-step finish

_(to document when Stage 3 is implemented)_

---

## Stage 4 — Detail screen: Step 2 (Verify Data)

_(to document when Stage 4 is implemented)_

---

## Stage 5 — Detail screen: Step 3 (WeInfuse Summary Note) + patient view refactor

_(to document when Stage 5 is implemented)_
