# MLID-1861 — Pharmacy Pipeline Stages

## Task Reference

- **Jira**: [MLID-1861](https://localinfusion.atlassian.net/browse/MLID-1861)
- **Story Points**: 3
- **Branch**: `feature/MLID-1861-pharmacy-pipeline-stages`
- **Base Branch**: `epic/MLID-1492-order-eligibility`
- **Status**: To Do

---

## Summary

Extend the existing 340B eligibility pipeline with pharmacy eligibility stages. This adds a `$lookup` to `insurance_plans` and a `$addFields` for `pharmacyEligible` computation, reusing the shared `_primaryInsurance` lookup already added by D1-T2.

---

## Codebase Analysis

### Current state (`ordersTracker.ts`)

- `get340BEligibilityStages()` (lines 335-437) returns 5 stages:
  1. `$addFields: _providerNpi`
  2. `$lookup: hospitalSystems`
  3. `$lookup: patient_insurances` → `_primaryInsurance` (shared)
  4. `$addFields: hospital340BEligible`
  5. `$unset: [_providerNpi, _340b_hospitals, _primaryInsurance]`

- **Problem**: Stage 5 removes `_primaryInsurance`, but pharmacy stages need it. The `$unset` must move to the end, after pharmacy computation.

- Pipeline appended at line 804-806 in `$facet.data`:
  ```typescript
  ...(category === 'new' && includeEligibility
    ? get340BEligibilityStages()
    : []),
  ```

### Existing tests (`ordersTracker.test.ts`, lines 1660-1809)

- 3 tests verify pipeline shape via `mockAggregate.mock.calls[0][0]`
- Pattern: find `$facet` stage → inspect `dataStages` for lookups/addFields
- Tests check for `hospitalSystems` lookup, `patient_insurances` lookup, `hospital340BEligible` addFields

### Types (`orders.ts`, line 131)

- `pharmacyEligible?: K extends 'new' ? string : never;` — already defined in D1-T1

---

## Implementation Steps

### Step 1 — RED: Write failing tests for pharmacy pipeline stages

- **File**: `apps/web/services/mongodb/__tests__/ordersTracker.test.ts`
- **What**: Add 3 new test cases inside the existing `getOrders with includeEligibility` describe block:

  1. **`should include pharmacy eligibility stages when category is new and includeEligibility is true`**
     - Assert `$lookup` to `insurance_plans` exists in dataStages
     - Assert `$addFields` with `pharmacyEligible` exists in dataStages

  2. **`should include $unset that cleans up _insurancePlan`**
     - Assert the `$unset` stage includes `_insurancePlan` in its array

  3. **`should NOT include insurance_plans lookup when includeEligibility is false`**
     - Assert `insurance_plans` lookup is absent (mirrors existing 340B negative test)

### Step 2 — GREEN: Extend pipeline function with pharmacy stages

- **File**: `apps/web/services/mongodb/ordersTracker.ts`
- **What**:
  1. Rename `get340BEligibilityStages()` → `getEligibilityPipelineStages()`
  2. Remove current stage 5 (`$unset`) from its position
  3. After stage 4 (hospital340BEligible), add:
     - **Stage 5**: `$lookup` to `insurance_plans` — match `planName` from `_primaryInsurance.insurance_plan_name`
     - **Stage 6**: `$addFields: { pharmacyEligible }` — `$switch` with 3 branches (waiting, yes, no)
  4. Add **Stage 7**: `$unset: ['_providerNpi', '_340b_hospitals', '_primaryInsurance', '_insurancePlan']`
  5. Update the call site (line 804-806) to use the new function name

### Step 3 — REFACTOR: Clean up and verify

- Ensure all existing 340B tests still pass (function rename shouldn't break assertions)
- Run full test suite, types check, lint/format on changed files

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/services/mongodb/ordersTracker.ts` | Modify | Rename function, add pharmacy stages 5-7, move $unset to end |
| `apps/web/services/mongodb/__tests__/ordersTracker.test.ts` | Modify | Add 3 pharmacy pipeline tests |

---

## Testing Strategy

- **Unit tests**: Verify pipeline shape — pharmacy lookups/addFields present when enabled, absent when disabled
- **Existing tests**: 3 existing 340B tests must continue to pass unchanged
- **Manual verification**: Enable `ORDER_ELIGIBILITY` flag, load new orders page, confirm `pharmacyEligible` values appear

---

## Pharmacy `$switch` Logic

```
Branch 1: _primaryInsurance empty → "Waiting on Insurance Input"
Branch 2: drug.pharmacyEligible ≠ false
           AND site.isPharmacyEligible = true
           AND (Medicare/Medicare Advantage auto-pass
                OR Commercial/Medicaid with plan.isPharmacyEligible = true)
         → "Yes"
Default: → "No"
```

---

## Security Considerations

- **No user input**: Pipeline stages use only internal collection data
- **No PHI exposure**: Temp fields cleaned up via `$unset`
- **Feature flag gated**: Stages only run when `ORDER_ELIGIBILITY` is enabled
