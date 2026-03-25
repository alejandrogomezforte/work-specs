# MLID-1955 — Locations missing isPharmacyEligible field show incorrect eligibility

## Task Reference

- **Jira**: [MLID-1955](https://localinfusion.atlassian.net/browse/MLID-1955)
- **Story Points**: 2
- **Branch**: `fix/MLID-1955-location-pharmacy-eligible-default`
- **Base Branch**: `develop`
- **Status**: Done
- **Epic Plan**: [MLID-1492-plan-progress.md](./MLID-1492-plan-progress.md)

---

## Summary

The `patientLocations` collection has a data inconsistency: the Mongoose schema defaults `isPharmacyEligible` to `true`, but the Looker webhook (raw MongoDB driver) never writes this field. This creates a split-brain problem where the Locations UI shows "Enabled" (Mongoose applies schema default on read / service layer normalizes with `?? true`) while the Orders Tracker pipeline shows "No" (aggregation uses `$ifNull: [..., false]`). The fix aligns all code paths to treat missing `isPharmacyEligible` as `false`, explicitly sets the field in the webhook upsert, and includes a migration script for existing data.

---

## Codebase Analysis

Key findings from investigation:

- **Mongoose model** (`apps/web/models/Location.ts:33`): `isPharmacyEligible: { type: Boolean, default: true }` — this default is wrong; locations should be opt-in, not opt-out
- **Webhook handler** (`apps/web/services/webhooks/handlers/util/webhookUpsertLocations.ts:128-137`): Uses raw MongoDB bulk ops with aggregation pipeline `$set`. Already has a `$ifNull` pattern for `location_type` backfill that can be replicated for `isPharmacyEligible`
- **Orders Tracker pipeline** (`apps/web/services/mongodb/ordersTracker.ts`): Already uses `$ifNull: ['$site.isPharmacyEligible', false]` — this is correct and needs no change
- **Location DB service** (`apps/web/services/locations/locationDbService.ts:49`): `normalizePharmacyEligible` function uses `?? true` — this is the primary data layer for the active `/users-locations` page
- **Users & Locations panel** (`apps/web/components/Modules/users-locations/UsersLocationsPanel.tsx:354`): Already uses `?? false` — correct, no change needed
- **Add/Edit Location modal** (`apps/web/components/Modules/users-locations/components/AddEditLocationModal/AddEditLocationModal.tsx:65`): Form default uses `?? true` — should be `?? false`
- **Deprecated admin locations page** (`apps/web/app/admin/locations/page.tsx:240,245`): Uses `?? true` — should be `?? false`
- **Deprecated admin detail page** (`apps/web/app/admin/locations/[id]/page.tsx:69,107`): Form default and reset both use `true` — should be `false`
- **Locations POST API** (`apps/web/app/api/admin/locations/route.ts:96`): Uses `body.isPharmacyEligible ?? true` — should be `?? false`
- **Test fixtures** (`apps/web/tests/locations/fixtures.ts:21`): `isPharmacyEligible: true` — should be `false` to match new default
- **Migration precedent**: `apps/web/scripts/migrate-reporting-preferences.ts` uses `connectToDatabase()` + raw `db.collection().updateMany()`

---

## Implementation Steps

### Step 1 — RED: Update Location model test to expect `false` default

- **Files**: `apps/web/models/__tests__/Location.test.ts`
- **What**: Change the test at line 20 from `expect(location.isPharmacyEligible).toBe(true)` to `expect(location.isPharmacyEligible).toBe(false)`. This test will fail because the model still defaults to `true`.

### Step 2 — GREEN: Change Mongoose schema default to `false`

- **Files**: `apps/web/models/Location.ts`
- **What**: Change line 33 from `default: true` to `default: false`. The test from Step 1 now passes.

### Step 3 — RED: Add webhook upsert test for `isPharmacyEligible` backfill

- **Files**: `apps/web/services/webhooks/handlers/util/webhookUpsertLocations.test.ts`
- **What**: Add a new `describe('isPharmacyEligible backfill')` block (following the existing `location_type backfill` pattern) with two tests:
  1. `should set isPharmacyEligible to false using $ifNull for new and existing documents` — verifies the `$set` payload includes `isPharmacyEligible: { $ifNull: ['$isPharmacyEligible', false] }`
  2. `should include isPharmacyEligible $ifNull in all upserted documents` — verifies multi-document payloads all include the field

  These tests will fail because the webhook handler does not include `isPharmacyEligible` in `$set`.

### Step 4 — GREEN: Add `isPharmacyEligible` backfill to webhook upsert

- **Files**: `apps/web/services/webhooks/handlers/util/webhookUpsertLocations.ts`
- **What**: In the `$set` block (line 130-135), add `isPharmacyEligible: { $ifNull: ['$isPharmacyEligible', false] }` alongside the existing `location_type` pattern. This preserves existing values for locations already configured via admin UI, and defaults to `false` for new/unconfigured locations.

### Step 5 — Fix location DB service `normalizePharmacyEligible`

- **Files**: `apps/web/services/locations/locationDbService.ts`
- **What**: Change `normalizePharmacyEligible` at line 49 from `?? true` to `?? false`. This is the primary data normalization layer used by the active `/users-locations` page. Update corresponding tests in `apps/web/services/locations/__tests__/locationDbService.test.ts` — flip assertions at lines 105 and 129 from `.toBe(true)` to `.toBe(false)`.

### Step 6 — Fix Add/Edit Location modal form default

- **Files**: `apps/web/components/Modules/users-locations/components/AddEditLocationModal/AddEditLocationModal.tsx`
- **What**: Change line 65 from `loc?.isPharmacyEligible ?? true` to `loc?.isPharmacyEligible ?? false`. This ensures the edit modal shows the correct unchecked state for locations missing the field.

### Step 7 — RED: Update deprecated admin locations page test to verify `undefined` renders "No"

- **Files**: `apps/web/app/admin/locations/__tests__/page.test.tsx`
- **What**: Add test cases for `isPharmacyEligible: false` rendering "No" and `isPharmacyEligible: undefined` rendering "No". The `undefined` test will fail because the page currently uses `?? true`.

### Step 8 — GREEN: Fix deprecated admin locations list page fallback

- **Files**: `apps/web/app/admin/locations/page.tsx`
- **What**: Change both occurrences of `location.isPharmacyEligible ?? true` (lines 240, 245) to `location.isPharmacyEligible ?? false`. The test from Step 7 now passes.

### Step 9 — Fix deprecated admin location detail page form defaults

- **Files**: `apps/web/app/admin/locations/[id]/page.tsx`
- **What**: Change the form `defaultValues` at line 69 from `isPharmacyEligible: true` to `isPharmacyEligible: false`. Change the `form.reset()` call at line 107 from `?? true` to `?? false`.

### Step 10 — Fix locations POST API fallback

- **Files**: `apps/web/app/api/admin/locations/route.ts`
- **What**: Change line 96 from `body.isPharmacyEligible ?? true` to `body.isPharmacyEligible ?? false`. Update corresponding test assertions in `apps/web/app/api/admin/locations/__tests__/route.test.ts` and `apps/web/app/api/admin/locations/route.spec.ts`.

### Step 11 — Update test fixtures

- **Files**: `apps/web/tests/locations/fixtures.ts`
- **What**: Change line 21 from `isPharmacyEligible: true` to `isPharmacyEligible: false` to match the new default behavior.

### Step 12 — Create migration script

- **Files**: `apps/web/scripts/migrate-locations-pharmacy-eligible.ts`
- **What**: Create a migration script following the pattern in `migrate-reporting-preferences.ts`. The script:
  1. Connects to the database via `connectToDatabase()`
  2. Runs `db.collection('patientLocations').updateMany({ isPharmacyEligible: { $exists: false } }, { $set: { isPharmacyEligible: false } })` to backfill all locations missing the field
  3. Logs the count of modified documents
  4. Exits with code 0 on success, 1 on failure

  Usage: `npx ts-node scripts/migrate-locations-pharmacy-eligible.ts`

### Step 13 — Final verification

- **Files**: None (verification only)
- **What**: Run full test suite, type checking, and formatting on changed files.

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/models/Location.ts` | Modify | Change `isPharmacyEligible` default from `true` to `false` |
| `apps/web/models/__tests__/Location.test.ts` | Modify | Update default assertion from `true` to `false` |
| `apps/web/services/webhooks/handlers/util/webhookUpsertLocations.ts` | Modify | Add `isPharmacyEligible: { $ifNull: ['$isPharmacyEligible', false] }` to `$set` |
| `apps/web/services/webhooks/handlers/util/webhookUpsertLocations.test.ts` | Modify | Add `isPharmacyEligible backfill` test block |
| `apps/web/services/locations/locationDbService.ts` | Modify | Change `normalizePharmacyEligible` from `?? true` to `?? false` |
| `apps/web/services/locations/__tests__/locationDbService.test.ts` | Modify | Update normalization assertions from `true` to `false` |
| `apps/web/components/Modules/users-locations/components/AddEditLocationModal/AddEditLocationModal.tsx` | Modify | Change form default from `?? true` to `?? false` |
| `apps/web/app/admin/locations/page.tsx` | Modify | Change `?? true` to `?? false` (2 occurrences) — deprecated page |
| `apps/web/app/admin/locations/__tests__/page.test.tsx` | Modify | Add tests for `false` and `undefined` rendering "No" — deprecated page |
| `apps/web/app/admin/locations/[id]/page.tsx` | Modify | Change form default and reset fallback — deprecated page |
| `apps/web/app/api/admin/locations/route.ts` | Modify | Change `?? true` to `?? false` |
| `apps/web/app/api/admin/locations/__tests__/route.test.ts` | Modify | Update default assertion from `true` to `false` |
| `apps/web/app/api/admin/locations/route.spec.ts` | Modify | Update default assertion from `true` to `false` |
| `apps/web/tests/locations/fixtures.ts` | Modify | Change fixture default from `true` to `false` |
| `apps/web/scripts/migrate-locations-pharmacy-eligible.ts` | Create | One-time migration to backfill `isPharmacyEligible: false` on documents missing the field |

---

## Testing Strategy

- **Unit tests — Location model** (`models/__tests__/Location.test.ts`):
  - `isPharmacyEligible` defaults to `false` when not provided
  - `isPharmacyEligible` can be explicitly set to `true`
  - `isPharmacyEligible` can be explicitly set to `false`

- **Unit tests — Webhook upsert** (`webhookUpsertLocations.test.ts`):
  - `$set` payload includes `isPharmacyEligible: { $ifNull: ['$isPharmacyEligible', false] }`
  - Multi-document payloads all include the `isPharmacyEligible` backfill
  - Existing `location_type` backfill tests still pass (regression)

- **Unit tests — Location DB service** (`locationDbService.test.ts`):
  - `normalizePharmacyEligible` converts undefined to `false`
  - Explicit `true`/`false` values pass through unchanged

- **Component tests — Deprecated locations page** (`locations/__tests__/page.test.tsx`):
  - Location with `isPharmacyEligible: true` renders "Yes"
  - Location with `isPharmacyEligible: false` renders "No"
  - Location with `isPharmacyEligible: undefined` renders "No"

- **Manual verification**:
  - Check `/users-locations?tab=locations`: locations never configured should show "Disabled"
  - Edit Location modal: checkbox unchecked for locations with undefined `isPharmacyEligible`
  - Check Orders Tracker: pharmacy eligibility should agree with Locations page
  - Create a new location without setting pharmacy eligible: should default to "Disabled"
  - Run migration script against dev/staging and verify document count (3 documents as of 2026-03-23)

---

## Security Considerations

- **Input validation**: The `?? false` fallback at the API boundary (`route.ts`) ensures missing boolean input defaults safely to disabled rather than enabled
- **Authorization**: No changes to authorization — existing admin-only access controls remain in place
- **PHI handling**: Not applicable — `isPharmacyEligible` is a location configuration field, not patient data

---

## Open Questions

- **Migration timing**: The migration script should be run after deployment to each environment. Confirm with the team whether this should be run before or after the code changes are deployed (recommendation: after, since the code changes make all paths treat missing as `false`, and the migration makes it explicit in the database).
- **Existing admin-configured locations**: Locations where an admin explicitly set `isPharmacyEligible: true` via the admin UI will be unaffected by this change. The webhook `$ifNull` pattern preserves existing values. Confirm this is the expected behavior.

---

## Lessons Learned

The initial codebase investigation missed the active `/users-locations` page and its service layer (`locationDbService.ts`). The investigation focused on the deprecated `/admin/locations` page. During manual testing, the user caught that the active page still showed "Enabled" for unconfigured locations. A broader grep for `isPharmacyEligible ?? true` across the entire codebase revealed the missed files. **Takeaway**: always search the full codebase for all occurrences of the pattern being fixed, not just the files mentioned in the bug report.
