# Hotfix: Remove deprecated `providerNPIs` field from HospitalSystem model

## Context

The `HospitalSystem` Mongoose model has a deprecated `providerNPIs: string[]` field that was the original design for storing provider NPIs as a flat array. It was superseded by the `providers: ProviderEntry[]` field (array of `{ npi, name }` subdocuments), which is what all production code uses. The deprecated field was already marked `@deprecated` but never removed.

If we deploy D2 to production with this field still in the schema, Mongoose will create an empty `providerNPIs: []` array on every new hospital system document. This is unnecessary data pollution we want to prevent.

**This is a D2 hotfix, not a Jira ticket.**

## Branch Strategy

Per `docs/agomez/plans/epic-git-branch-strategy.md`:
- Create branch: `fix/MLID-1021-remove-providerNPIs-field` from `epic/MLID-1021-hospital-systems-340B`
- Merge back into epic branch after completion

## Changes

### 1. Model — `apps/web/models/HospitalSystem.ts`

- **Remove** `providerNPIs: string[]` from `HospitalSystemDocument` interface (line 19)
- **Remove** the `@deprecated` JSDoc comment (line 18)
- **Remove** `providerNPIs` schema definition (lines 54-62)

### 2. Model Tests — `apps/web/models/HospitalSystem.test.ts`

- **Remove** `providerNPIs` from the "should create a HospitalSystem with all fields" test (lines 20, 27)
- **Remove** the entire "should default providerNPIs to an empty array" test (lines 33-39)
- **Remove** all `providerNPIs` validation tests:
  - "should pass validation with a valid 10-digit NPI" (line 71)
  - "should pass validation with multiple valid NPIs" (line 81)
  - "should fail validation for NPI shorter than 10 digits" (lines 91, 96)
  - "should fail validation for NPI longer than 10 digits" (lines 102, 107)
  - "should fail validation for NPI with non-numeric characters" (lines 113, 118)
  - "should fail validation if any NPI in the array is invalid" (lines 124, 129)
  - "should pass validation with an empty providerNPIs array" (lines 132-135)
- **Remove** `providerNPIs` from the "should have correct document interface structure" test (lines 225, 232)

### 3. API Route Tests — `apps/web/app/api/hospitals/[id]/route.test.ts`

- **Remove** `providerNPIs` from mock data objects (lines 132, 147, 253)

### 4. API Route Tests — `apps/web/app/api/hospitals/route.test.ts`

- **Remove** `providerNPIs` from mock data objects (lines 110, 440)

### 5. Component Tests — `apps/web/app/hospital-system-agreements/components/AddEditHospitalModal.test.tsx`

- **Remove** `providerNPIs` from mock data objects (lines 504, 533)

### NOT touched

- `apps/web/services/jobs/definitions/rxPreferredClaims.ts` — uses a local variable named `providerNPIs` (a `Record<string, string>` for mapping claim IDs to NPI values). Completely unrelated to the HospitalSystem model.

## Verification

1. Run model tests: `cd apps/web && npx jest models/HospitalSystem.test.ts`
2. Run API route tests: `cd apps/web && npx jest app/api/hospitals`
3. Run component tests: `cd apps/web && npx jest AddEditHospitalModal.test`
4. Run full type check: `cd apps/web && npx tsc --noEmit`
5. Run lint: `cd apps/web && npx eslint models/HospitalSystem.ts`

## Plan Progress Update

After merging, add a row to the **QA Hotfixes** table in `docs/agomez/plans/MLID-1021-plan-progress.md`:

| Task ID | Summary | Status | Branch | Merged to Epic | Notes |
|---------|---------|:------:|--------|:--------------:|-------|
| - | Remove deprecated `providerNPIs` field from model | Done | `fix/MLID-1021-remove-providerNPIs-field` | Yes | Prevents Mongoose from creating empty `providerNPIs: []` on new docs |
