# MLID-1418: Differentiate Real Locations from Non-Location Entries

## Overview

Add a `location_type` field to the `PatientLocations` collection to distinguish between clinics, pharmacies, and other entries. This allows filtering location dropdowns throughout LISA to show only actual clinic locations.

---

## Progress

| Stage | Description | Status | Completed Date |
|-------|-------------|--------|----------------|
| 1 | Create field in MongoDB schema | **COMPLETED** | 2026-02-04 |
| 2 | Create dropdown in Edit Location popup | **COMPLETED** | 2026-02-04 |
| 3 | Investigation - Location dropdowns | **COMPLETED** | 2026-02-05 |
| 4 | Investigation - API endpoints | **COMPLETED** | 2026-02-05 |
| 5 | Execute API & Frontend filter work | **COMPLETED** | 2026-02-05 |
| 6 | Backfill via webhook hook | **COMPLETED** | 2026-02-06 |

### Progress Notes

#### Stage 1 - Completed 2026-02-04
- Added `location_type` field to `apps/web/models/Location.ts`
  - Type: String enum with values `'clinic' | 'pharmacy' | 'other'`
  - Default: `'clinic'`
  - Exported `LOCATION_TYPES` constant and `LocationType` type for reuse
- Updated `apps/web/types/locationDTO.ts` to include optional `location_type` field
- Added 4 unit tests in `apps/web/models/__tests__/Location.test.ts`:
  - Defaults to 'clinic' when not provided
  - Allows 'pharmacy' value
  - Allows 'other' value
  - Rejects invalid values
- All tests passing, TypeScript types check passed

#### Stage 2 - Completed 2026-02-04
- Moved `LOCATION_TYPES` and `LocationType` to `apps/web/types/locationDTO.ts` (to avoid mongoose import issues in tests)
- Updated `apps/web/models/Location.ts` to re-export types from locationDTO
- Added `location_type` field to `AddEditLocationModal`:
  - Updated `FormData` interface
  - Added dropdown with Clinic/Pharmacy/Other options
  - Updated `getInitialFormData` to read existing value
  - Added CSS styles for the select element
- Updated `UpdateLocationPayload` in both services:
  - `apps/web/services/locations/locationDbService.ts`
  - `apps/web/services/locations/locationService.ts`
- Added 5 new tests in `AddEditLocationModal.test.tsx`:
  - Renders location type dropdown in add mode
  - Defaults to "clinic" in add mode
  - Shows existing location_type value in edit mode
  - Includes location_type in create payload
  - Includes location_type in update payload
- All 34 tests passing, TypeScript types check passed

#### Stage 3 - Completed 2026-02-05
- Completed codebase investigation for all location dropdowns
- Found 9 locations where dropdowns display `location_name`:
  - `/appointments` - uses `LocationSelect` ~~(shows `city` only)~~ **FIXED: now shows `location_name || city`**
  - `/intakes` - filter uses `MuiStandaloneMultiSelect`
  - `/intakes/[id]/review` - PatientCreateMode uses `MuiLocationSelect`
  - `/orders-tracker/[category]` - filter dropdown uses native `<select>` (shared across all 3 tabs)
  - `/orders-tracker` - AddOrderDialog uses `MuiLocationSelect`
  - `/orders-tracker` - AddPartialOrderDialog uses `MuiLocationSelect`
  - `/users-locations` (Users tab) - UsersFilters uses custom MultiSelect
  - `/users-locations` (Users tab) - edit-user-dialog uses `MuiLocationSelect`
  - `/users-locations` (Locations tab) - shows all locations in table (no filter needed)
- Identified 5 component patterns that handle location display:
  - `MuiLocationSelect` (4 usages) - primary component, fetches from `/api/locations`
  - `LocationSelect` (1 usage) - **FIXED: now shows `location_name || city`** (was `city` only)
  - `MuiStandaloneMultiSelect` (1 usage) - intakes page filter
  - `UsersFilters` (1 usage) - custom implementation
  - Native `<select>` in orders-tracker (1 usage) - builds options from order data, NOT from API
- **Fixed LocationSelect.tsx** to display `location_name` with fallback to `city`:
  - Modified line 32: `label: location.location_name || location.city`
  - Added 3 unit tests in `__tests__/LocationSelect.test.tsx`
  - All tests passing, TypeScript types check passed
- **Decision**: All dropdowns will filter by `location_type = 'clinic'` except:
  - `/appointments` - route deprecated soon, no work needed
  - `/users-locations` (Locations tab) - needs to show all types for admin management

#### Stage 4 - Completed 2026-02-05
- Mapped all components from Stage 3 to their API endpoints
- Found 2 main API endpoints used for location data:
  - `GET /api/locations` - used by MuiLocationSelect, LocationSelect, /intakes filter
  - `GET /api/admin/locations` - used by UsersLocationsPanel (both tabs)
- Found 2 different `fetchLocations` functions:
  - `utils/api/location/index.ts` → `/api/locations`
  - `services/locations/locationService.ts` → `/api/admin/locations`
- Identified non-dropdown usages that must not break:
  - SenderInformationCard uses `/api/locations?fax=xxx` for fax lookup
  - Locations tab table needs ALL locations (no filtering)
- **Decision**: Add optional `?type=clinic` query parameter to both endpoints

---

## Development Stages

### Stage 1: Create Field in MongoDB Schema
Add the `location_type` field to the schema and types. No migration yet - all work will be deployed together.

**Status:** COMPLETED

**Files to modify:**
- `apps/web/models/Location.ts` - Add field to Mongoose schema
- `apps/web/types/locationDTO.ts` - Add field to TypeScript interface

**Schema change:**
```typescript
location_type: {
  type: String,
  enum: ['clinic', 'pharmacy', 'other'],
  default: 'clinic'
}
```

---

### Stage 2: Create Dropdown in Edit Location Popup
Add a dropdown to the AddEditLocationModal to display and capture the `location_type` value.

**Status:** COMPLETED

**Files to modify:**
- `apps/web/components/Modules/users-locations/components/AddEditLocationModal/AddEditLocationModal.tsx`
- `apps/web/components/Modules/users-locations/components/AddEditLocationModal/__tests__/AddEditLocationModal.test.tsx`
- `apps/web/services/locations/locationDbService.ts` - Add `location_type` to `UpdateLocationPayload`
- `apps/web/services/locations/locationService.ts` - Add `location_type` to `UpdateLocationPayload`

**Changes:**
1. Add `location_type` to `FormData` interface
2. Add dropdown/select with options: Clinic, Pharmacy, Other
3. Include `location_type` in form submission payload
4. Update `getInitialFormData` to read `location_type` from location

---

### Stage 3: Investigation - Location Dropdowns in the App
Identify all places in the app where location dropdowns are displayed.

**Status:** COMPLETED

#### Location Dropdown Inventory

| App Route | Route File | Component Name | Lines | Filter by Type? | Work on it? |
|-----------|------------|----------------|-------|-----------------|-------------|
| `/appointments` | `apps/web/app/appointments/page.tsx` | `LocationSelect` | 225-230 | **YES** | **NO** - route deprecated soon |
| `/intakes` (filter) | `apps/web/app/intakes/page.tsx` | `MuiStandaloneMultiSelect` | 785-792 | **YES** | **YES** |
| `/intakes/[id]/review` (patient create) | `apps/web/components/Modules/document-intake/steps/patient-lookup-modes/PatientCreateMode.tsx` | `MuiLocationSelect` | 248-254 | **YES** | **YES** |
| `/orders-tracker/[category]` (filter) | `apps/web/app/orders-tracker/[category]/page.tsx` | Native `<select>` | 396-408 | **YES** | **YES** |
| `/orders-tracker` (Add Order dialog) | `apps/web/app/orders-tracker/AddOrderDialog/AddOrderDialog.tsx` | `MuiLocationSelect` | 216-224 | **YES** | **YES** |
| `/orders-tracker` (Add Lead dialog) | `apps/web/app/orders-tracker/AddOrderDialog/AddPartialOrderDialog.tsx` | `MuiLocationSelect` | 178-186 | **YES** | **YES** |
| `/users-locations` (Users tab - filter) | `apps/web/app/users-locations/page.tsx` | `UsersFilters` (MultiSelect) | 405-417 | **YES** | **YES** |
| `/users-locations` (Users tab - edit dialog) | `apps/web/components/Modules/users-management/users-list/edit-user-dialog/edit-user-dialog.tsx` | `MuiLocationSelect` | 185-194 | **YES** | **YES** |
| `/users-locations` (Locations tab) | `apps/web/app/users-locations/page.tsx` | Locations table | - | **NO** - show all types | **NO** - needs all types |

#### Display Format by Component

| Component | File | Line | Display Logic |
|-----------|------|------|---------------|
| `MuiLocationSelect` | `components/UI/MUI/MuiLocationSelect.tsx` | 61 | `location.location_name \|\| location.city \|\| ''` |
| `LocationSelect` | `components/Modules/appointment/LocationSelect.tsx` | 32 | `location.location_name \|\| location.city` |
| `MuiStandaloneMultiSelect` (intakes) | `app/intakes/page.tsx` | 719 | `loc.location_name \|\| \`${loc.city}, ${loc.state}\`` |
| `UsersFilters` | `components/Modules/users-locations/components/UsersFilters/UsersFilters.tsx` | 57 | `loc.location_name \|\| \`${loc.city}, ${loc.state}\`` |
| Native `<select>` (orders-tracker) | `app/orders-tracker/[category]/page.tsx` | 312-332 | `order.site.location_name \|\| \`${order.site.city}, ${order.site.state}\`` |

#### Key Findings

1. **Primary component**: `MuiLocationSelect` is used in 4 locations
2. **Legacy component**: `LocationSelect` - **FIXED** to show `location_name || city` (was `city` only)
3. **Custom implementations**:
   - `UsersFilters` builds its own options from location data
   - Orders tracker filter uses native `<select>` with options built from order data (not from API)
4. **API-based dropdowns**: `fetchLocations()` → `/api/locations` (all except orders-tracker filter)
5. **Orders tracker filter is different**: It builds location options from loaded orders, not from `/api/locations` API

#### Filtering Decisions

| Location | Decision | Reason |
|----------|----------|--------|
| `/users-locations` Users tab (filter) | **Filter by clinic** | Users work at clinic locations |
| `/users-locations` Locations tab | **No filter** | Admins need to manage all location types |

#### Summary
- All dropdowns (except `/users-locations` Locations tab) will filter by `location_type = 'clinic'`
- `/appointments` route is deprecated soon - will not be modified
- 8 dropdowns to implement filtering across 5 component patterns

---

### Stage 4: Investigation - API Endpoints
Determine which API endpoints need updates or if new endpoints are required.

**Status:** COMPLETED

#### Component to API Mapping

| App Route | Component Name | API Endpoint | API Filename | Function & Lines |
|-----------|----------------|--------------|--------------|------------------|
| `/intakes` (filter) | `MuiStandaloneMultiSelect` | `GET /api/locations` | `apps/web/app/api/locations/route.ts` | `GET()` lines 10-55 |
| `/intakes/[id]/review` | `MuiLocationSelect` | `GET /api/locations` | `apps/web/app/api/locations/route.ts` | `GET()` lines 10-55 |
| `/orders-tracker/[category]` (filter) | Native `<select>` | `GET /api/orders/{category}` | `apps/web/app/api/orders/[new\|lead\|maintenance]/route.ts` | Orders include `site` via `$lookup` |
| `/orders-tracker` (Add Order) | `MuiLocationSelect` | `GET /api/locations` | `apps/web/app/api/locations/route.ts` | `GET()` lines 10-55 |
| `/orders-tracker` (Add Lead) | `MuiLocationSelect` | `GET /api/locations` | `apps/web/app/api/locations/route.ts` | `GET()` lines 10-55 |
| `/users-locations` (Users tab - filter) | `UsersFilters` | `GET /api/admin/locations` | `apps/web/app/api/admin/locations/route.ts` | `GET()` lines 25-62 |
| `/users-locations` (Users tab - edit) | `MuiLocationSelect` | `GET /api/locations` | `apps/web/app/api/locations/route.ts` | `GET()` lines 10-55 |
| `/users-locations` (Locations tab) | Locations table | `GET /api/admin/locations` | `apps/web/app/api/admin/locations/route.ts` | `GET()` lines 25-62 |

#### API Endpoints Summary

| Endpoint | File | Used By Dropdowns | Used By Non-Dropdowns | Needs Filtering? |
|----------|------|-------------------|----------------------|------------------|
| `GET /api/locations` | `app/api/locations/route.ts` | MuiLocationSelect (4x), /intakes filter | SenderInformationCard (fax lookup) | **YES** - add `?type=` param |
| `GET /api/admin/locations` | `app/api/admin/locations/route.ts` | UsersFilters (Users tab) | Locations tab table | **BOTH** - param optional |
| `GET /api/orders/{category}` | `app/api/orders/*/route.ts` | orders-tracker filter (via order.site) | Orders table | N/A - filter in frontend |

#### Client-side Fetch Functions

| Function | File | Calls API |
|----------|------|-----------|
| `fetchLocations()` | `utils/api/location/index.ts` | `GET /api/locations` |
| `fetchLocations()` | `services/locations/locationService.ts` | `GET /api/admin/locations` |

**Note:** Two different `fetchLocations` functions exist - one for public API, one for admin API.

#### Non-Dropdown Usages (must not break)

1. **SenderInformationCard** - Uses `/api/locations?fax=xxx` to lookup location by fax number
   - This is a single-location lookup, not a dropdown
   - Should still work with `?type=` param (just ignore if not provided)

2. **Locations tab table** - Uses `/api/admin/locations` to show ALL locations for management
   - Must NOT filter by type - admins need to see all locations
   - Solution: Make `?type=` param optional, only filter when provided

3. **Orders table** - Uses `/api/orders/{category}` which populates `site` from PatientLocations
   - The orders-tracker filter builds options from loaded orders
   - Filter should happen in frontend when building dropdown options

#### Questions Answered

- [x] Should `/api/locations` accept a `?type=clinic` filter parameter? **YES**
- [x] Or should we create a new endpoint like `/api/locations/clinics`? **NO** - use query param instead
- [x] Do we need to update `/api/admin/locations` to support filtering? **YES** - optional `?type=` param
- [x] Which endpoints need to include `location_type` in response? **All GET endpoints already return full documents**
- [x] Which endpoints need to accept `location_type` in request body? **Already done in Stage 2**

#### Decision

Add optional `?type=clinic` query parameter to:
1. `GET /api/locations` - Filter locations by type
2. `GET /api/admin/locations` - Filter locations by type (optional)

Frontend components will pass `?type=clinic` to get only clinic locations for dropdowns.

---

### Stage 5: Execute API & Frontend Filter Work
Implement the API changes and update frontend dropdowns to filter by location type.

**Status:** COMPLETED

#### Stage 5 - Completed 2026-02-05
- Updated `GET /api/locations` (`apps/web/app/api/locations/route.ts`):
  - Added `type` query parameter parsing
  - Added MongoDB filter for `location_type` when `type` param is provided
  - Added 7 unit tests for type filtering
- Updated `GET /api/admin/locations` (`apps/web/app/api/admin/locations/route.ts`):
  - Added `type` query parameter parsing
  - Passes `type` to `fetchLocationsFromDB()` service function
- Updated `fetchLocationsFromDB()` (`apps/web/services/locations/locationDbService.ts`):
  - Added `type` to `LocationsListParams` interface
  - Added `location_type` filter when `type` param is provided
  - Added 2 unit tests for type filtering
- Updated `fetchLocations` (public API) (`apps/web/utils/api/location/index.ts`):
  - Added `type?: LocationType` to params interface
  - Added 6 unit tests for type parameter
- Updated `fetchLocations` (admin API) (`apps/web/services/locations/locationService.ts`):
  - Added `type?: LocationType` to `LocationsListParams` interface
  - Added 2 unit tests for type parameter
- Updated `MuiLocationSelect` (`apps/web/components/UI/MUI/MuiLocationSelect.tsx`):
  - Now passes `{ type: 'clinic' }` to `fetchLocations()`
  - Added unit test to verify type filtering
- Updated `/intakes` page (`apps/web/app/intakes/page.tsx`):
  - Now passes `{ type: 'clinic' }` to `fetchLocations()`
- Updated `UsersLocationsPanel` (`apps/web/components/Modules/users-locations/UsersLocationsPanel.tsx`):
  - Now passes `{ type: 'clinic' }` to `fetchLocations()`
- Updated `/orders-tracker/[category]` page (`apps/web/app/orders-tracker/[category]/page.tsx`):
  - Added frontend filter for `location_type === 'clinic'` (or undefined for backward compatibility)
- All 365 location-related tests passing, TypeScript types check passed

---

#### 5.1 API Endpoint Changes

##### 5.1.1 Update `GET /api/locations`
**File:** `apps/web/app/api/locations/route.ts`

**Changes:**
- [x] Add `type` query parameter parsing (line ~12)
- [x] Add MongoDB filter for `location_type` when `type` param is provided
- [x] Keep backward compatible (no `type` param = return all locations)

**Example:**
```typescript
const typeParam = searchParams.get('type'); // 'clinic' | 'pharmacy' | 'other' | null

// Add filter if type is specified
const filter = typeParam ? { location_type: typeParam } : {};
const result = await collection.find(filter).toArray();
```

##### 5.1.2 Update `GET /api/admin/locations`
**File:** `apps/web/app/api/admin/locations/route.ts`

**Changes:**
- [x] Add `type` query parameter to `LocationsListParams` interface
- [x] Pass `type` to `fetchLocationsFromDB()` service function
- [x] Update `fetchLocationsFromDB()` in `locationDbService.ts` to filter by type

**Files to modify:**
- `apps/web/app/api/admin/locations/route.ts` - Add param parsing
- `apps/web/services/locations/locationDbService.ts` - Add filter logic

---

#### 5.2 Client-side Fetch Function Changes

##### 5.2.1 Update `fetchLocations` (public API)
**File:** `apps/web/utils/api/location/index.ts`

**Changes:**
- [x] Add `type?: LocationType` to params interface
- [x] Append `?type=` to URL when provided

```typescript
export const fetchLocations = async (params?: {
  fax?: string;
  type?: LocationType;  // NEW
}): Promise<...>
```

##### 5.2.2 Update `fetchLocations` (admin API)
**File:** `apps/web/services/locations/locationService.ts`

**Changes:**
- [x] Add `type?: LocationType` to `LocationsListParams` interface
- [x] Include `type` in URLSearchParams when provided

---

#### 5.3 Component Changes

##### 5.3.1 `MuiLocationSelect` - Pass `type=clinic`
**File:** `apps/web/components/UI/MUI/MuiLocationSelect.tsx`

**Changes:**
- [x] Update `fetchLocations()` call (line ~54) to pass `{ type: 'clinic' }`

**Used by:** 4 locations (all need clinic filtering)
- `/intakes/[id]/review` (PatientCreateMode)
- `/orders-tracker` (AddOrderDialog)
- `/orders-tracker` (AddPartialOrderDialog)
- `/users-locations` (edit-user-dialog)

##### 5.3.2 `/intakes` page filter - Pass `type=clinic`
**File:** `apps/web/app/intakes/page.tsx`

**Changes:**
- [x] Update `fetchLocations()` call (line ~245) to pass `{ type: 'clinic' }`

##### 5.3.3 `UsersFilters` (Users tab) - Pass `type=clinic`
**File:** `apps/web/components/Modules/users-locations/UsersLocationsPanel.tsx`

**Changes:**
- [x] Update `fetchLocations()` call (line ~140) to pass `{ type: 'clinic' }`
- [x] This fetches locations for the Users tab filter dropdown only

**Note:** The Locations tab table must NOT filter - it needs all location types.

##### 5.3.4 `/orders-tracker/[category]` filter - Frontend filtering
**File:** `apps/web/app/orders-tracker/[category]/page.tsx`

**Changes:**
- [x] Filter `locationFilters` (lines 312-332) to only include sites with `location_type === 'clinic'`
- [x] This dropdown builds options from order data, not from API

**Note:** Orders already have `site` populated via `$lookup`. The `site` object includes `location_type`.

---

#### 5.4 Implementation Order

1. **API Layer** (backend first)
   - 5.1.1 - Update `/api/locations`
   - 5.1.2 - Update `/api/admin/locations` + `locationDbService.ts`

2. **Client Fetch Functions** (middle layer)
   - 5.2.1 - Update `utils/api/location/index.ts`
   - 5.2.2 - Update `services/locations/locationService.ts`

3. **Components** (frontend last)
   - 5.3.1 - `MuiLocationSelect`
   - 5.3.2 - `/intakes` page
   - 5.3.3 - `UsersLocationsPanel`
   - 5.3.4 - `/orders-tracker/[category]` page

---

#### 5.5 Test Checklist

**API Tests:**
- [x] `/api/locations` returns all locations when no `type` param
- [x] `/api/locations?type=clinic` returns only clinics
- [x] `/api/locations?type=pharmacy` returns only pharmacies
- [x] `/api/locations?fax=xxx` still works (fax lookup)
- [x] `/api/admin/locations` returns all locations when no `type` param
- [x] `/api/admin/locations?type=clinic` returns only clinics

**Component Tests:**
- [x] `MuiLocationSelect` only shows clinic locations
- [x] `/intakes` filter only shows clinic locations (via `fetchLocations({ type: 'clinic' })`)
- [x] `UsersFilters` only shows clinic locations (via `fetchLocations({ type: 'clinic' })`)
- [x] `/orders-tracker` filter only shows clinic locations (via frontend filter)
- [x] Locations tab table still shows ALL locations (uses separate `useLocations` hook)

**Non-breaking Tests:**
- [x] `SenderInformationCard` fax lookup still works (type param is optional)
- [x] Locations tab management still works (separate data source)

---

### Stage 6: Backfill via Webhook Hook
Modify the existing webhook hook that updates the `PatientLocations` collection to backfill `location_type` for existing records.

**Status:** COMPLETED

#### Stage 6 - Completed 2026-02-06
- Modified `webhookUpsertLocations.ts` (`apps/web/services/webhooks/handlers/util/webhookUpsertLocations.ts`):
  - Changed from simple `$set` update to aggregation pipeline update
  - Added `location_type: { $ifNull: ['$location_type', 'clinic'] }` to the `$set` operation
  - This ensures:
    - **New documents**: Get `location_type: 'clinic'` set
    - **Existing documents with undefined/null `location_type`**: Get `location_type: 'clinic'` set
    - **Existing documents with `location_type` already set**: Preserve their value ('clinic', 'pharmacy', or 'other')
- Updated 7 existing tests to use aggregation pipeline format `[{ $set: ... }]`
- Added 2 new tests for `location_type` backfill logic:
  - Verifies `$ifNull` is used for setting `location_type`
  - Verifies all upserted documents include the `$ifNull` logic
- All 9 tests passing, TypeScript types check passed

**Approach:** Instead of a standalone migration script, modify the Looker locations webhook handler to set `location_type` during normal sync operations.

**Files to modify:**
- `apps/web/services/webhooks/handlers/looker/` (location webhook handler)

**Hook logic:**
When updating a location document:
1. If `location_type` is `null` or `undefined` for that specific document → set `location_type: 'clinic'`
2. Otherwise, preserve the existing `location_type` value

**Benefits of this approach:**
- No separate migration script to run
- Existing locations get backfilled as they are synced via normal webhook flow
- Simpler deployment - just deploy the code change
- Self-healing - any locations that somehow lose the field will be corrected automatically

**Testing:**
- [x] Test webhook handler with location that has no `location_type` → uses `$ifNull` to set 'clinic'
- [x] Test webhook handler preserves existing `location_type` values → uses `$ifNull` which returns existing value if not null
- [x] All 9 webhook tests passing

---

## Current State Analysis

### Schema (`apps/web/models/Location.ts`)
- No `location_type` field exists
- Has `isPharmacyEligible` boolean (different purpose - for pharmacy billing eligibility)

### Services
| Service | Location | Purpose |
|---------|----------|---------|
| `locationDbService.ts` | `services/locations/` | Server-side DB operations |
| `locationService.ts` | `services/locations/` | Client-side API calls to `/api/admin/locations` |
| `utils/api/location/index.ts` | `utils/api/location/` | Client-side API calls to `/api/locations` |

---

## Testing Checklist

### Backend Tests
- [x] Schema accepts valid `location_type` values
- [x] Schema rejects invalid `location_type` values
- [x] POST creates location with `location_type`
- [x] PUT updates `location_type`
- [x] GET returns `location_type` in response
- [x] Filtered endpoint returns only requested type
- [x] Webhook hook sets `location_type: 'clinic'` when null/undefined (via `$ifNull`)
- [x] Webhook hook preserves existing `location_type` values (via `$ifNull`)

### Frontend Tests
- [ ] AddEditLocationModal displays location type dropdown
- [ ] AddEditLocationModal saves location type correctly
- [ ] Location dropdowns only show clinics (where applicable)
- [ ] All identified dropdown usages work correctly

### E2E Tests (Playwright)
- [ ] Edit location and change type to pharmacy
- [ ] Verify pharmacy no longer appears in filtered dropdowns
- [ ] Create new location with specific type

---

## Open Questions

1. **Default for existing locations**: Should all existing locations default to `'clinic'` or should we require manual review?
   - **Decision**: Default to `'clinic'` via webhook hook backfill. Admins can manually change to `'pharmacy'` or `'other'` via the Edit Location modal.

2. **Should `location_type` be displayed in the Locations table?**: Adding a column could help admins identify and manage location types.
   - **Proposed**: Yes, add as a column in the Locations tab table

3. **User location assignment**: When assigning users to locations, should we show all types or only clinics?
   - **Proposed**: Show all types for user assignment (users might work at pharmacy locations)

---

## Rollback Plan

If issues arise:
1. Webhook hook change can be reverted - existing `location_type` values will be preserved
2. API changes are backward compatible (type param is optional)
3. Frontend changes can be reverted to not pass type filter

---

## Files Modified in This Branch

For historical purposes, here is the complete list of all files modified in the `feature/MLID-1418-locations-or-pharmacies` branch:

### API Routes
- `apps/web/app/api/admin/locations/__tests__/route.test.ts` - Added tests for type filtering
- `apps/web/app/api/admin/locations/route.ts` - Added `type` query parameter support
- `apps/web/app/api/locations/route.test.ts` - Added tests for type filtering
- `apps/web/app/api/locations/route.ts` - Added `type` query parameter support

### Page Components
- `apps/web/app/intakes/page.tsx` - Filter locations by `type: 'clinic'`
- `apps/web/app/orders-tracker/[category]/page.tsx` - Frontend filter for clinic locations only

### UI Components
- `apps/web/components/Modules/appointment/LocationSelect.tsx` - Display `location_name || city`
- `apps/web/components/Modules/appointment/__tests__/LocationSelect.test.tsx` - Added tests
- `apps/web/components/Modules/users-locations/UsersLocationsPanel.tsx` - Filter locations by `type: 'clinic'`
- `apps/web/components/Modules/users-locations/components/AddEditLocationModal/AddEditLocationModal.module.css` - Styles for dropdown
- `apps/web/components/Modules/users-locations/components/AddEditLocationModal/AddEditLocationModal.tsx` - Added `location_type` dropdown
- `apps/web/components/Modules/users-locations/components/AddEditLocationModal/__tests__/AddEditLocationModal.test.tsx` - Added tests
- `apps/web/components/UI/MUI/MuiLocationSelect.test.tsx` - Added test for type filtering
- `apps/web/components/UI/MUI/MuiLocationSelect.tsx` - Filter by `type: 'clinic'`

### Models & Types
- `apps/web/models/Location.ts` - Added `location_type` field to schema
- `apps/web/models/__tests__/Location.test.ts` - Added tests for schema validation
- `apps/web/types/locationDTO.ts` - Added `LocationType` type and `LOCATION_TYPES` constant

### Services
- `apps/web/services/locations/__tests__/locationService.test.ts` - Added tests for type parameter
- `apps/web/services/locations/locationDbService.ts` - Added `type` to query params
- `apps/web/services/locations/locationService.ts` - Added `type` to fetch params

### Webhook Handlers
- `apps/web/services/webhooks/handlers/util/webhookUpsertLocations.test.ts` - Updated and added tests for aggregation pipeline
- `apps/web/services/webhooks/handlers/util/webhookUpsertLocations.spec.ts` - Updated tests for aggregation pipeline format
- `apps/web/services/webhooks/handlers/util/webhookUpsertLocations.ts` - Added `location_type` backfill via `$ifNull`

### Utilities
- `apps/web/utils/api/location/index.test.ts` - New file with tests for type parameter
- `apps/web/utils/api/location/index.ts` - Added `type` to fetch params
- `apps/web/utils/helpers/sanitizers.ts` - Added `location_type` to sanitizeLocationData whitelist

**Total: 26 files modified**
