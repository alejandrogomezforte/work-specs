# MLID-1862 — Pharmacy UI Column

## Task Reference

- **Jira**: [MLID-1862](https://localinfusion.atlassian.net/browse/MLID-1862)
- **Story Points**: 2
- **Branch**: `feature/MLID-1862-pharmacy-ui-column`
- **Base Branch**: `epic/MLID-1492-order-eligibility`
- **Status**: To Do

---

## Summary

Add the `pharmacyEligible` column to the New Orders DataGrid, positioned immediately after the 340B column, gated by the same `showEligibilityColumns` flag.

---

## Codebase Analysis

- `NewOrders.tsx` line 228-240: 340B column is in a spread-ternary `...(showEligibilityColumns ? [...] : [])`
- The pharmacy column goes inside the same array, after the 340B column def
- Column spec: `field: 'pharmacyEligible'`, `headerName: 'Pharmacy Eligible'`, `width: 170`, `sortable: false`, `filterable: false`
- `valueGetter`: `row.pharmacyEligible ?? ''`
- Type already defined: `pharmacyEligible?: K extends 'new' ? string : never` (from D1-T1)

---

## Implementation Steps

### Step 1 — RED: Write failing tests

- **File**: `NewOrders.test.tsx`
- **Tests** (inside a new `Pharmacy Eligible column` describe block):
  1. Should not include column by default
  2. Should not include column when `showEligibilityColumns: false`
  3. Should include column when `showEligibilityColumns: true`
  4. Should not be sortable or filterable
  5. Should return `pharmacyEligible` value from valueGetter
  6. Should return empty string when `pharmacyEligible` is undefined
  7. Should be positioned immediately after 340B column

### Step 2 — GREEN: Add the pharmacy column

- **File**: `NewOrders.tsx`
- **What**: Add pharmacy column definition inside the existing spread-ternary array, after the 340B column

### Step 3 — REFACTOR: Verify quality

- All tests pass, types check, lint/format on changed files

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/app/orders-tracker/Orders/NewOrders.tsx` | Modify | Add pharmacy column in eligibility spread-ternary |
| `apps/web/app/orders-tracker/Orders/NewOrders.test.tsx` | Modify | Add 7 pharmacy column tests |
