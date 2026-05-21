# MLID-2213 — Data Testing: "(No Value)" filter option for enum columns

Working notebook for UI + server verification that every `singleSelect` (enum) column on the orders tracker exposes a `(No Value)` option in the filter dropdown — and only there — and that the server translates that selection into MongoDB predicates matching rows where the field is missing, `null`, or `''`.

- **Branch:** `feature/MLID-2213-enum-no-value-filter`
- **Plan:** `docs/agomez/plans/MLID-2213-enum-no-value-filter.md`
- **Sentinel constant:** `NO_VALUE_SENTINEL = '__LI_NO_VALUE__'` (declared in `apps/web/utils/api/orders/gridToMongoQuery.ts`)
- **Helper:** `withNoValueOption()` in `apps/web/app/orders-tracker/utils/enumValueOptions.ts`
- **Commits on branch:**
  - `28bead10` — server sentinel translation
  - `9b5262a2` — `withNoValueOption` helper
  - `babad272` — wired helper into 10 enum columns
  - `2fd0c6e3` — chip label sentinel mapping

---

## Affected enum columns (10 total)

The `(No Value)` option is added to every `type: 'singleSelect'` column. Some are gated by the `ORDERS_TRACKER_IG_COLUMNS` feature flag — when the flag is OFF locally, those columns won't render and there's nothing to test for them.

### New tab — `apps/web/app/orders-tracker/Orders/NewOrders.tsx`

| # | Field | Header | Enum | Always visible? |
|---|-------|--------|------|-----------------|
| 1 | `orderType` | **Order Type** | `NewOrderType` | ✅ |
| 2 | `coverageStatus` | **Coverage Status** | `CoverageStatus` | only when `ORDERS_TRACKER_IG_COLUMNS` is ON |
| 3 | `status` | **Order Status** | `OrderStatus` | only when `ORDERS_TRACKER_IG_COLUMNS` is ON |
| 4 | `liPharmacyStatus` | **LI Pharm Status** | `LIPharmacyStatus` | only when `ORDERS_TRACKER_IG_COLUMNS` is ON |
| 5 | `audit` | **Audit** | `Audit` | only when `ORDERS_TRACKER_IG_COLUMNS` is ON |

### Maintenance tab — `apps/web/app/orders-tracker/Orders/MaintenanceOrders.tsx`

| # | Field | Header | Enum | Always visible? |
|---|-------|--------|------|-----------------|
| 6 | `changeType` | **Change Type** | `MaintenanceOrderChangeType` | ✅ |
| 7 | `coverageStatus` | **Coverage Status** | `CoverageStatus` | only when `ORDERS_TRACKER_IG_COLUMNS` is ON |
| 8 | `status` | **Order Status** | `OrderStatus` | only when `ORDERS_TRACKER_IG_COLUMNS` is ON |

### Lead tab — `apps/web/app/orders-tracker/Orders/LeadOrder.tsx`

| # | Field | Header | Enum | Always visible? |
|---|-------|--------|------|-----------------|
| 9 | `orderType` | **Order Type** | `NewOrderType` | ✅ |
| 10 | `status` | **Status** | `LeadOrderStatus` | ✅ |

> **Minimum coverage if the IG columns flag is OFF:** #1 (New tab Order Type), #6 (Maintenance Change Type), #9 + #10 (Lead Order Type and Status). The other six columns share the same helper — once the contract is verified on one column it holds for all.

---

## UI checklist

Run for **each** column listed above (or the minimum subset noted above). Tick as you go.

### Filter dropdown contract
- [ ] Open the column header menu → "Filter" → operator dropdown shows `Is`, `Is not`, `Is any of` (no new operators).
- [ ] Value dropdown lists every enum value followed by **`(No Value)`** as the LAST item.
- [ ] `(No Value)` is rendered with literal parentheses (not `__LI_NO_VALUE__`).

### Inline edit dropdown contract (regression)
- [ ] Click a cell to inline-edit the column → dropdown lists ONLY the enum values; **`(No Value)` does NOT appear**.
- [ ] Selecting any enum value from inline edit saves the row (no regression).

### Filter behavior — `is`
- [ ] Apply `<column> is (No Value)` → grid shows only rows where the field is missing, `null`, or `''`.
- [ ] Active-filter chip text reads `<column> is "(No Value)"` (parens preserved, NOT `__LI_NO_VALUE__`).
- [ ] Remove the filter → grid restores to unfiltered view.

### Filter behavior — `is not`
- [ ] Apply `<column> is not (No Value)` → grid excludes rows from the previous step.
- [ ] Chip reads `<column> is not "(No Value)"`.

### Filter behavior — `is any of`
- [ ] Apply `<column> is any of [SomeRealValue, (No Value)]` → grid shows the union (real-value rows + null/missing/empty rows).
- [ ] Chip reads `<column> is any of "SomeRealValue, (No Value)"`.
- [ ] With ≥3 values selected including the sentinel, the chip truncates with `+N more` and the sentinel counts toward visible/remaining like any other value.

### Cross-tab regression
- [ ] Existing string filters (e.g. `Patient contains "smith"`) still work and chips render unchanged.
- [ ] Existing date filters (e.g. `Due Date after 4/2/2026`) still work and chips render `4/2/2026` not the ISO string.

---

## Server-side verification (optional, Mongo Compass)

If the UI looks wrong, drop into the network tab and grab the request to `/api/orders-tracker/...`. The `filterModel` query param should encode the sentinel literally:

```
filterModel.items=[{"field":"orderType","operator":"is","value":"__LI_NO_VALUE__"}]
```

The server pipeline appends:
- `is` + sentinel → `{ $or: [ { _filterKey0: { $exists: false } }, { _filterKey0: null }, { _filterKey0: '' } ] }`
- `not` + sentinel → `{ $and: [ { _filterKey0: { $exists: true } }, { _filterKey0: { $ne: null } }, { _filterKey0: { $ne: '' } } ] }`
- `isAnyOf` only sentinel → same as `is`
- `isAnyOf` mixed → `{ $or: [ { _filterKey0: { $in: [...real] } }, <emptyPredicate> ] }`

### Predict expected matches per column

Use these Compass queries against `neworders` / `maintenanceorders` / `leadorders` to count how many rows the `(No Value)` filter SHOULD return for each column. Compare against the row count the UI shows after applying the filter.

```js
// neworders — Order Type (No Value)
{ $or: [ { orderType: { $exists: false } }, { orderType: null }, { orderType: "" } ] }

// neworders — Order Status (No Value)
{ $or: [ { status: { $exists: false } }, { status: null }, { status: "" } ] }

// neworders — Coverage Status (No Value)
{ $or: [ { coverageStatus: { $exists: false } }, { coverageStatus: null }, { coverageStatus: "" } ] }

// neworders — LI Pharm Status (No Value)
{ $or: [ { liPharmacyStatus: { $exists: false } }, { liPharmacyStatus: null }, { liPharmacyStatus: "" } ] }

// neworders — Audit (No Value)
// Note: rows where audit === "N/A" are NOT included by design.
{ $or: [ { audit: { $exists: false } }, { audit: null }, { audit: "" } ] }

// maintenanceorders — Change Type (No Value)
{ $or: [ { changeType: { $exists: false } }, { changeType: null }, { changeType: "" } ] }

// maintenanceorders — Order Status (No Value)
{ $or: [ { status: { $exists: false } }, { status: null }, { status: "" } ] }

// maintenanceorders — Coverage Status (No Value)
{ $or: [ { coverageStatus: { $exists: false } }, { coverageStatus: null }, { coverageStatus: "" } ] }

// leadorders — Order Type (No Value)
{ $or: [ { orderType: { $exists: false } }, { orderType: null }, { orderType: "" } ] }

// leadorders — Status (No Value)
{ $or: [ { status: { $exists: false } }, { status: null }, { status: "" } ] }
```

> **Note on the `audit` column:** rows with `audit === Audit.NA` (`'N/A'`) render BLANK in the cell because of `valueFormatter`, but the underlying value is the string `'N/A'`. By design, `(No Value)` does NOT match those rows — only true `null` / missing / `''`. If the PO wants `'N/A'` to count as "no value" too, that's a follow-up plan-level decision (Open Question in the plan).

---

## Findings

Fill in as you test.

| Column | Tab | `is` count (Mongo) | `is` count (UI) | Match? | Notes |
|--------|-----|--------------------|------------------|--------|-------|
| Order Type | New | | | | |
| Order Status | New | | | | |
| Coverage Status | New | | | | |
| LI Pharm Status | New | | | | |
| Audit | New | | | | |
| Change Type | Maintenance | | | | |
| Order Status | Maintenance | | | | |
| Coverage Status | Maintenance | | | | |
| Order Type | Lead | | | | |
| Status | Lead | | | | |

### Issues found

_None yet._
