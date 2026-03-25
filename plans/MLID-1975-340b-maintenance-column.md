# MLID-1975 --- Add 340B Column to Maintenance Table

## Task Reference

- **Jira**: [MLID-1975](https://localinfusion.atlassian.net/browse/MLID-1975)
- **Story Points**: 3
- **Branch**: `feature/MLID-1975-340b-maintenance-column`
- **Base Branch**: `epic/MLID-1492-order-eligibility`
- **Status**: To Do
- **Epic Plan**: [MLID-1492-plan-progress.md](./MLID-1492-plan-progress.md)

---

## Summary

Extend the existing 340B Eligible column (currently only on New Orders) to also appear on the Maintenance Orders table at `/orders-tracker/maintenance`. This reuses the existing `ORDER_ELIGIBILITY` feature flag, the existing `getEligibilityPipelineStages()` pipeline function, and the same UI column pattern. The task is scoped to 340B only --- pharmacy eligibility is NOT included for maintenance orders.

---

## Codebase Analysis

Key findings from investigation:

- **Pipeline guard** (`ordersTracker.ts` lines 927, 946): Currently `category === 'new' && includeEligibility` controls whether eligibility stages are appended. This guard appears in two places (paginated path and non-paginated path). Both need widening to include `'maintenance'`.
- **`getEligibilityPipelineStages()`** returns ALL stages (340B + pharmacy). Going with Option B: compute both, display only 340B. The pharmacy computation adds negligible overhead (one `$lookup` + one `$addFields`), and avoids parameterizing a shared function. When pharmacy is later added to maintenance, it is purely a UI change.
- **Type guard** (`types/orders.ts` line 192): `hospital340BEligible` is `K extends 'new' ? string : never`. Must widen to `K extends 'new' | 'maintenance' ? string : never`. Leave `pharmacyEligible` scoped to `'new'` only.
- **Maintenance API route** (`app/api/orders/maintenance/route.ts` line 55): Does NOT currently import `FeatureFlagService` or pass `includeEligibility`. Must mirror the pattern from `app/api/orders/new/route.ts` lines 56--72.
- **`getMaintenanceOrdersColumns()`** (`MaintenanceOrders.tsx` line 46): Does not have a `showEligibilityColumns` parameter. Must add one and insert the 340B column using the same spread-ternary pattern as `NewOrders.tsx` lines 290--311.
- **`page.tsx`** (line 284): Already has `isOrderEligibilityEnabled` from `useFeatureFlag(FeatureFlag.ORDER_ELIGIBILITY)` at line 101. Just needs to pass `showEligibilityColumns: isOrderEligibilityEnabled` to `getMaintenanceOrdersColumns()`.
- **Existing test** (`ordersTracker.test.ts` line 1847): Test titled "should NOT include eligibility stages when category is maintenance" currently asserts that `hospitalSystems` lookup is absent for maintenance. This test must be updated to assert the OPPOSITE --- eligibility stages SHOULD be present for maintenance.
- **Maintenance route test** (`route.test.ts`): Does not mock `FeatureFlagService`. Must add mock + two tests mirroring new orders route pattern.
- **MaintenanceOrders test** (`MaintenanceOrders.test.tsx`): No eligibility column tests. Must add describe block for 340B column presence/absence.

---

## Implementation Steps

### Step 1 --- RED: Write failing tests for service layer (pipeline includes eligibility for maintenance)

- **Files**: `apps/web/services/mongodb/__tests__/ordersTracker.test.ts`
- **What**:
  1. Update the existing test at line 1847 ("should NOT include eligibility stages when category is maintenance even if includeEligibility is true") --- rename it to "should include eligibility stages when category is maintenance and includeEligibility is true" and flip the assertion from `toBeUndefined()` to `toBeDefined()`.
  2. Add a new test: "should NOT include eligibility stages when category is maintenance and includeEligibility is false" --- same setup but `includeEligibility: false`, assert `hospitalSystems` lookup is absent.
- **Expected result**: The renamed test FAILS because the production code still guards with `category === 'new'`.

### Step 2 --- GREEN: Widen pipeline guard to include maintenance

- **Files**: `apps/web/services/mongodb/ordersTracker.ts` (lines 927, 946)
- **What**: Change both instances of `category === 'new' && includeEligibility` to `(category === 'new' || category === 'maintenance') && includeEligibility`. This allows the full eligibility pipeline (340B + pharmacy stages) to run for maintenance orders when the flag is enabled.
- **Expected result**: The updated test from Step 1 PASSES. Existing new-orders eligibility tests remain green.

### Step 3 --- RED: Write failing tests for maintenance API route (feature flag + includeEligibility pass-through)

- **Files**: `apps/web/app/api/orders/maintenance/route.test.ts`
- **What**:
  1. Add `jest.mock('@/services/featureFlags/featureFlagService')` to the mock section.
  2. Import `FeatureFlagService` and `FeatureFlag`.
  3. Create typed mock: `const mockGetFeatureFlag = FeatureFlagService.getFeatureFlag as jest.MockedFunction<typeof FeatureFlagService.getFeatureFlag>`.
  4. Add test: "should pass includeEligibility: true when ORDER_ELIGIBILITY flag is enabled" --- mock `mockGetFeatureFlag.mockResolvedValue(true)`, call `GET`, assert `mockGetMaintenanceOrders` was called with `expect.objectContaining({ includeEligibility: true })`.
  5. Add test: "should pass includeEligibility: false when ORDER_ELIGIBILITY flag is disabled" --- mock `mockGetFeatureFlag.mockResolvedValue(false)`, same pattern.
- **Expected result**: Both tests FAIL because the production route does not call `FeatureFlagService.getFeatureFlag` or pass `includeEligibility`.

### Step 4 --- GREEN: Add feature flag check to maintenance API route

- **Files**: `apps/web/app/api/orders/maintenance/route.ts`
- **What**:
  1. Add imports: `import { FeatureFlagService } from '@/services/featureFlags/featureFlagService'` and `import { FeatureFlag } from '@/utils/featureFlags/featureFlags'`.
  2. In the `GET` handler, after the auth check and before the `getOrders()` call (between lines 54 and 55), add: `const includeEligibility = await FeatureFlagService.getFeatureFlag(FeatureFlag.ORDER_ELIGIBILITY);`
  3. Pass `includeEligibility` to the `getOrders()` call at line 55.
- **Expected result**: API route tests from Step 3 PASS. Existing GET/POST/PUT/DELETE tests remain green.

### Step 5 --- RED: Write failing tests for 340B column in MaintenanceOrders

- **Files**: `apps/web/app/orders-tracker/Orders/MaintenanceOrders.test.tsx`
- **What**: Add a new `describe('340B Eligible column')` block inside the existing `describe('Column definitions')`:
  1. Test: "should include 340B Eligible column when showEligibilityColumns is true" --- call `getMaintenanceOrdersColumns({ onDeleteClick: () => {}, showEligibilityColumns: true })`, find column with `field === 'hospital340BEligible'`, assert it is defined with `headerName: '340B Eligible'`.
  2. Test: "should NOT include 340B Eligible column when showEligibilityColumns is false" --- same but `showEligibilityColumns: false`, assert column is undefined.
  3. Test: "should NOT include 340B Eligible column when showEligibilityColumns is not provided" --- call without the param, assert column is undefined.
  4. Test: "should return hospital340BEligible value via valueGetter" --- create column with `showEligibilityColumns: true`, find column, call `valueGetter` with a mock order that has `hospital340BEligible: 'Hospital ABC'`, assert result is `'Hospital ABC'`.
  5. Test: "should return empty string via valueGetter when hospital340BEligible is undefined" --- same but order has no `hospital340BEligible`, assert result is `''`.
  6. Test: "340B column should not be sortable or filterable" --- assert `sortable: false` and `filterable: false`.
  7. Test: "should NOT include Pharmacy Eligible column even when showEligibilityColumns is true" --- assert no column with `field === 'pharmacyEligible'` exists (pharmacy is out of scope for maintenance).
- **Expected result**: Tests FAIL because `getMaintenanceOrdersColumns` does not accept `showEligibilityColumns` and does not include the 340B column.

### Step 6 --- GREEN: Widen type and add 340B column to MaintenanceOrders

- **Files**:
  - `apps/web/types/orders.ts` (line 192)
  - `apps/web/app/orders-tracker/Orders/MaintenanceOrders.tsx` (lines 46--67, and insert after line 292)
- **What**:
  1. **Type widening** (`orders.ts` line 192): Change `K extends 'new' ? string : never` to `K extends 'new' | 'maintenance' ? string : never` for `hospital340BEligible` only. Leave `pharmacyEligible` unchanged.
  2. **Function signature** (`MaintenanceOrders.tsx`): Add `showEligibilityColumns?: boolean` to the destructured params object (line 54 area, after `onDrugSwitchIntent`).
  3. **Column insertion** (`MaintenanceOrders.tsx`): After the `providerPracticeFax` column (line 292) and before the `isIgColumsEnabled` spread (line 293), add:
     ```typescript
     ...(showEligibilityColumns
       ? [
           {
             field: 'hospital340BEligible',
             headerName: '340B Eligible',
             width: 200,
             sortable: false,
             filterable: false,
             valueGetter: (_: unknown, row: FullOrder<'maintenance'>) =>
               row.hospital340BEligible ?? '',
           } as GridColDef,
         ]
       : []),
     ```
     Note: Only 340B, no pharmacy column. The spread-ternary pattern matches `NewOrders.tsx`.
- **Expected result**: All tests from Step 5 PASS. TypeScript compiles. Existing maintenance tests remain green.

### Step 7 --- Wire page.tsx to pass showEligibilityColumns

- **Files**: `apps/web/app/orders-tracker/[category]/page.tsx` (line 284)
- **What**: Add `showEligibilityColumns: isOrderEligibilityEnabled` to the `getMaintenanceOrdersColumns()` call at line 284. The variable `isOrderEligibilityEnabled` already exists from line 101.
- **Expected result**: No new tests needed --- this is pure wiring. The feature flag controls both API (from Step 4) and UI (this step). Existing page behavior is unchanged when flag is off.

### Step 8 --- REFACTOR: Verify quality

- **Files**: All modified files
- **What**:
  1. Run `npx eslint --fix` on all changed files.
  2. Run `npx prettier --write` on all changed files.
  3. Run `npm run types:check` from `apps/web/`.
  4. Run `npm run test` from `apps/web/` to confirm all tests pass.
  5. Verify no unrelated changes in `git diff`.

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/services/mongodb/__tests__/ordersTracker.test.ts` | Modify | Update maintenance eligibility test to expect stages present; add test for flag-off |
| `apps/web/services/mongodb/ordersTracker.ts` | Modify | Widen category guard from `'new'` to `'new' || 'maintenance'` (two locations) |
| `apps/web/app/api/orders/maintenance/route.test.ts` | Modify | Add FeatureFlagService mock + two tests for includeEligibility pass-through |
| `apps/web/app/api/orders/maintenance/route.ts` | Modify | Import FeatureFlagService, check ORDER_ELIGIBILITY, pass includeEligibility to getOrders |
| `apps/web/app/orders-tracker/Orders/MaintenanceOrders.test.tsx` | Modify | Add 340B column tests (presence, absence, valueGetter, sortable/filterable, no pharmacy) |
| `apps/web/types/orders.ts` | Modify | Widen hospital340BEligible type to include 'maintenance' |
| `apps/web/app/orders-tracker/Orders/MaintenanceOrders.tsx` | Modify | Add showEligibilityColumns param + 340B column with spread-ternary |
| `apps/web/app/orders-tracker/[category]/page.tsx` | Modify | Pass showEligibilityColumns to getMaintenanceOrdersColumns |

---

## Testing Strategy

### Unit Tests

**Service layer** (`ordersTracker.test.ts`):
- Eligibility stages ARE included when `category='maintenance'` + `includeEligibility=true` (verify `$lookup` to `hospitalSystems` exists in `$facet.data`)
- Eligibility stages are NOT included when `category='maintenance'` + `includeEligibility=false`
- Existing new-orders eligibility tests remain unchanged and green

**API route** (`maintenance/route.test.ts`):
- `GET` passes `includeEligibility: true` when `ORDER_ELIGIBILITY` flag returns `true`
- `GET` passes `includeEligibility: false` when `ORDER_ELIGIBILITY` flag returns `false`
- Existing GET/POST/PUT/DELETE tests remain unchanged and green

**UI columns** (`MaintenanceOrders.test.tsx`):
- 340B column present when `showEligibilityColumns: true`
- 340B column absent when `showEligibilityColumns: false` or not provided
- `valueGetter` returns `hospital340BEligible ?? ''`
- Column is not sortable, not filterable
- Pharmacy column is NOT present (explicit negative test)

### Integration / Manual Verification

1. Enable `ORDER_ELIGIBILITY` flag in admin panel
2. Navigate to `/orders-tracker/maintenance`
3. Verify "340B Eligible" column appears with computed values
4. Verify "Pharmacy Eligible" column does NOT appear
5. Disable the flag and verify the column disappears
6. Navigate to `/orders-tracker/new` and verify both 340B and Pharmacy columns still work correctly (regression check)

---

## Security Considerations

- **Input validation**: No new user inputs. The column is read-only, computed server-side.
- **Authorization**: Existing auth check in the maintenance route (`getServerSession`) is unchanged. The feature flag adds a server-side gate --- if the flag is off, the pipeline stages are not appended, so no eligibility data is returned regardless of what the client requests.
- **Feature flag dual enforcement**: Server (API route checks `ORDER_ELIGIBILITY` before passing `includeEligibility`) + Client (`page.tsx` checks `useFeatureFlag` before passing `showEligibilityColumns`).
- **PHI handling**: No new PHI exposure. The 340B computation uses provider NPI (not PHI) and insurance payer type (categorical, not identifying). No patient-identifying data is added to the response beyond what already exists.

---

## Open Questions

None. The approach (Option B: compute full pipeline, display 340B only) was confirmed in the task description. The feature flag reuse was confirmed by PO.
