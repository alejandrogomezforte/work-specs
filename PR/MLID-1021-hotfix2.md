# [MLID-1021] Hotfix: Remove deprecated `providerNPIs` field from HospitalSystem model

## Summary

- Remove the deprecated `providerNPIs: string[]` field from the `HospitalSystem` Mongoose schema and TypeScript interface
- This field was the original design for storing provider NPIs as a flat array, superseded by `providers: ProviderEntry[]` (array of `{ npi, name }` subdocuments) which all production code uses
- Without this removal, Mongoose would auto-create an empty `providerNPIs: []` on every new hospital system document, causing unnecessary data pollution

**Branch:** `fix/MLID-1021-remove-providerNPIs-field` → `epic/MLID-1021-hospital-systems-340B`

---

## Changes

| File | Change |
|------|--------|
| `models/HospitalSystem.ts` | Removed `providerNPIs` from interface (+ `@deprecated` JSDoc) and schema definition |
| `models/HospitalSystem.test.ts` | Removed 9 `providerNPIs` tests (creation, default, 7 validation cases, interface structure) |
| `app/api/hospitals/[id]/route.test.ts` | Removed `providerNPIs` from 2 mock objects + 1 assertion |
| `app/api/hospitals/route.test.ts` | Removed `providerNPIs` from 2 mock objects |
| `app/hospital-system-agreements/components/AddEditHospitalModal.test.tsx` | Removed `providerNPIs` from 2 mock objects |

*All paths relative to `apps/web/`.*

### NOT touched

- `services/jobs/definitions/rxPreferredClaims.ts` — uses a local variable named `providerNPIs` (`Record<string, string>` for mapping claim IDs to NPI values). Completely unrelated to the HospitalSystem model.

---

## Verification

| Check | Result |
|-------|--------|
| Model tests (`models/HospitalSystem.test.ts`) | 12 passed |
| API route tests (`app/api/hospitals/`) | 52 passed (3 suites) |
| Component tests (`AddEditHospitalModal.test.tsx`) | 27 passed |
| TypeScript (`tsc --noEmit`) | Clean (only pre-existing `@slack/web-api` errors) |
| ESLint (`models/HospitalSystem.ts`) | Clean |

---

## Test Plan

- [ ] Create a new hospital system → verify no `providerNPIs` field appears in the MongoDB document
- [ ] Edit an existing hospital system → verify no regression in name update flow
- [ ] View hospital detail page → verify providers tab still loads correctly from `providers` array
