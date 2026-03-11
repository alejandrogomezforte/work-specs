# MLID-1859 — 340B UI Column + Feature Flag Wiring

## Task Reference

- **Jira**: [MLID-1859](https://localinfusion.atlassian.net/browse/MLID-1859)
- **Story Points**: 2
- **Branch**: `feature/MLID-1859-340b-ui-column`
- **Base Branch**: `epic/MLID-1492-order-eligibility`
- **Status**: To Do
- **Epic Plan**: [MLID-1492-plan-progress.md](./MLID-1492-plan-progress.md)

---

## Summary

Wire the `ORDER_ELIGIBILITY` feature flag from `page.tsx` through to `getNewOrdersColumns()` and add a `340B Eligible` column to the New Orders DataGrid. The column displays the `hospital340BEligible` value computed by the pipeline stages (D1-T2). It is only visible when the feature flag is enabled.

---

## Codebase Analysis

Key findings from investigation:

### `page.tsx` (orders-tracker/[category]/page.tsx)
- **Lines 6-7**: Already imports `useFeatureFlag` and `FeatureFlag` — no new imports needed.
- **Line 43**: Already calls `useFeatureFlag(FeatureFlag.ORDER_DETAILS)` for the order details feature. Same pattern for `ORDER_ELIGIBILITY`.
- **Lines 101-105**: `getNewOrdersColumns()` called with `{ user, onDelete, isOrderDetailsEnabled }`. Need to add `showEligibilityColumns`.
- **Line 112**: `useMemo` dependency array `[session, isOrderDetailsEnabled, onEdit, onDelete]`. Need to add the new flag variable.

### `NewOrders.tsx` (orders-tracker/Orders/NewOrders.tsx)
- **Lines 28-36**: `getNewOrdersColumns` accepts `{ user, onDelete, isOrderDetailsEnabled }`. Need to add `showEligibilityColumns?: boolean`.
- **Lines 244-262**: Existing spread-ternary pattern for the delete action column — exact pattern to reuse for the eligibility column.
- Column should be inserted **before** the delete action (end of the static columns array), using the same spread-ternary approach.

### `NewOrders.test.tsx` (orders-tracker/Orders/NewOrders.test.tsx)
- Tests `getNewOrdersColumns` directly (not rendering a component).
- Uses `columns.find(col => col.headerName === '...')` pattern for column lookups.
- Uses `valueGetter?.()` calls with `(undefined as never, row, {} as never, {} as never)` signature.
- Has `createMockOrder()` helper that can be extended with `hospital340BEligible`.

### `page.test.tsx` (orders-tracker/[category]/page.test.tsx)
- **Line 18-20**: Mocks `useFeatureFlag` to return `true` for all flags.
- **Lines 144-146**: Mocks `getNewOrdersColumns` to return `[]` — ignores all params.
- Since `getNewOrdersColumns` is fully mocked, the page test doesn't validate param pass-through. Adding a dedicated test to verify `showEligibilityColumns` is passed would require changing the mock to a spy. This is out of scope for this task — the column behavior is fully tested in `NewOrders.test.tsx`.

### Precedent: Spread-ternary pattern (NewOrders.tsx:244-262)
```typescript
...(condition
  ? [{
      field: 'fieldName',
      type: 'actions' as GridColType,
      headerName: '',
      sortable: false,
      filterable: false,
      ...
    } as GridColDef]
  : []),
```

---

## Implementation Steps

### Step 1 — Add `showEligibilityColumns` param to `getNewOrdersColumns`

- **File**: `apps/web/app/orders-tracker/Orders/NewOrders.tsx`
- **What**: Add `showEligibilityColumns?: boolean` to the function's param type (line 32). Destructure with default `false`.

### Step 2 — Add the 340B Eligible column using spread-ternary

- **File**: `apps/web/app/orders-tracker/Orders/NewOrders.tsx`
- **What**: Before the delete action spread-ternary (line 244), add:
  ```typescript
  ...(showEligibilityColumns
    ? [{
        field: 'hospital340BEligible',
        headerName: '340B Eligible',
        width: 200,
        sortable: false,
        filterable: false,
        valueGetter: (_: unknown, row: FullOrder<'new'>) => row.hospital340BEligible ?? '',
      } as GridColDef]
    : []),
  ```

### Step 3 — Wire the feature flag in `page.tsx`

- **File**: `apps/web/app/orders-tracker/[category]/page.tsx`
- **What**:
  1. Add `const isOrderEligibilityEnabled = useFeatureFlag(FeatureFlag.ORDER_ELIGIBILITY);` (after line 43)
  2. Pass `showEligibilityColumns: isOrderEligibilityEnabled` to `getNewOrdersColumns()` (line 103)
  3. Add `isOrderEligibilityEnabled` to the `useMemo` dependency array (line 112)

### Step 4 — Write tests in `NewOrders.test.tsx`

- **File**: `apps/web/app/orders-tracker/Orders/NewOrders.test.tsx`
- **What**: Add a new `describe('340B Eligible column')` block with the following tests:

  1. **Column absent by default** — call `getNewOrdersColumns({ onDelete })` without `showEligibilityColumns`, verify no column with `headerName: '340B Eligible'` exists.
  2. **Column absent when `showEligibilityColumns: false`** — explicit false, same assertion.
  3. **Column present when `showEligibilityColumns: true`** — verify column exists with correct field, headerName, width.
  4. **Column is not sortable and not filterable** — verify `sortable: false`, `filterable: false`.
  5. **valueGetter returns `hospital340BEligible` value** — create mock order with `hospital340BEligible: 'Hospital ABC'`, verify valueGetter returns `'Hospital ABC'`.
  6. **valueGetter returns empty string when `hospital340BEligible` is undefined** — verify fallback to `''`.
  7. **Column positioned before delete action** — verify column index is before the actions column.

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/app/orders-tracker/Orders/NewOrders.tsx` | Modify | Add `showEligibilityColumns` param + 340B column |
| `apps/web/app/orders-tracker/[category]/page.tsx` | Modify | Wire `ORDER_ELIGIBILITY` feature flag to column function |
| `apps/web/app/orders-tracker/Orders/NewOrders.test.tsx` | Modify | Add tests for 340B column presence/absence and valueGetter |

---

## Testing Strategy

- **Unit tests**: `NewOrders.test.tsx` — test `getNewOrdersColumns` column definitions directly (7 test cases described in Step 4)
- **No page test changes needed**: `page.test.tsx` mocks `getNewOrdersColumns` entirely, so column wiring is not testable there without refactoring the mock (out of scope)
- **Manual verification**: Enable `ORDER_ELIGIBILITY` flag in admin, navigate to Orders Tracker > New Orders, verify 340B Eligible column appears with values from the pipeline

---

## Security Considerations

- **Input validation**: N/A — display-only column, no user input
- **Authorization**: Feature flag gates visibility; no new API surface
- **PHI handling**: `hospital340BEligible` contains hospital system names and insurance status text, not PHI

---

## Open Questions

None — all dependencies (D1-T1, D1-T2, D1-T4) are complete.
