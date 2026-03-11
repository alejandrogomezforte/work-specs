# Locations CRUD — Data Flow & System Architecture

> **Repo:** li-call-processor
> **Date:** 2026-01-30
> **Route:** `/admin/locations`
> **API:** `/api/admin/locations`

---

## 1. Architecture Overview

The Locations feature follows a **4-layer architecture**:

```
┌─────────────────────────────────────────────────────────┐
│  1. PAGE COMPONENTS (React)                             │
│     app/admin/locations/page.tsx          → List         │
│     app/admin/locations/new/page.tsx      → Create       │
│     app/admin/locations/[id]/page.tsx     → Edit/Detail  │
│         Uses: React Hook Form + DynamicForm component    │
└────────────────────────┬────────────────────────────────┘
                         │ fetch()
┌────────────────────────▼────────────────────────────────┐
│  2. CLIENT SERVICE (Browser-side)                       │
│     services/locations/locationService.ts               │
│         fetchLocations(), getLocationById(),             │
│         createLocation(), updateLocation(),              │
│         deleteLocation()                                 │
└────────────────────────┬────────────────────────────────┘
                         │ HTTP
┌────────────────────────▼────────────────────────────────┐
│  3. API ROUTES (Next.js App Router)                     │
│     app/api/admin/locations/route.ts        → GET, POST  │
│     app/api/admin/locations/[id]/route.ts   → GET, PUT,  │
│                                               DELETE     │
│         Auth checks + validation + PHI masking           │
└────────────────────────┬────────────────────────────────┘
                         │ function call
┌────────────────────────▼────────────────────────────────┐
│  4. DB SERVICE (Server-side)                            │
│     services/locations/locationDbService.ts             │
│         Uses: Native MongoDB driver (not Mongoose)       │
│         Collection: 'patientLocations'                   │
└────────────────────────┬────────────────────────────────┘
                         │
                    ┌────▼────┐
                    │ MongoDB │
                    └─────────┘
```

---

## 2. Layer 1 — Page Components

Three routes, all under `app/admin/locations/`:

| Route | File | Purpose |
|-------|------|---------|
| `/admin/locations` | `page.tsx` (272 lines) | List with search, sort, pagination |
| `/admin/locations/new` | `new/page.tsx` (302 lines) | Create form |
| `/admin/locations/[id]` | `[id]/page.tsx` (502 lines) | Edit form + detail view |

### State Management

- `useState` for data, loading, and error states.
- `useSearchParams` + `useRouter` for URL-persisted filters (page, search, sortBy, sortOrder).

### Forms

All forms use **React Hook Form** (`useForm<LocationFormData>`) combined with `DynamicForm` from `components/UICompositions/`.

The form config defines a **12-column grid layout** where each field specifies:
- `row` — which row it appears in
- `columns` — grid column span (1–12)
- `rules` — validation rules
- `render` — a function returning `StyledInput`, `TagsInput`, or a checkbox

### Edit Page Form Fields (8 rows)

| Row | Fields | Columns |
|-----|--------|---------|
| 1 | WeInfuse ID (required), Location Name | 6 + 6 |
| 2 | Facility Name (required), Servicing Address | 6 + 6 |
| 3 | City (required), Callback Number, Fax Number | 4 + 4 + 4 |
| 4 | Group NPI, Tax ID/TIN | 6 + 6 |
| 5 | Reminder FNF (checkbox) | 12 |
| 6 | Pharmacy Eligible (checkbox) | 12 |
| 7 | Enable Intake Processing (checkbox) | 12 |
| 8 | Location City Names (TagsInput) | 12 |

### Create Page Form Fields (7 rows)

Same structure as edit, but **without** `location_name` and `isIntakeProcessingEnabled`.

### PHI Masking

The list page uses the `usePHIMasking()` hook to mask `we_infuse_id` in the table via `maskMemberId()`.

---

## 3. Layer 2 — Client Service

**File:** `services/locations/locationService.ts` (152 lines)

Thin `fetch()` wrappers — no caching, no SWR, no React Query. Each function builds the URL, calls the API, and returns typed data.

| Function | HTTP | Endpoint |
|----------|------|----------|
| `fetchLocations(params)` | GET | `/api/admin/locations?page=&pageSize=&search=&sortBy=&sortOrder=` |
| `getLocationById(id)` | GET | `/api/admin/locations/{id}` |
| `createLocation(data)` | POST | `/api/admin/locations` |
| `updateLocation(id, data)` | PUT | `/api/admin/locations/{id}` |
| `deleteLocation(id)` | DELETE | `/api/admin/locations/{id}` |

### Interfaces

```typescript
LocationsListParams {
  page?: number
  pageSize?: number
  search?: string
  sortBy?: string
  sortOrder?: 'asc' | 'desc'
}

LocationsListResponse {
  data: ILocation[]
  pagination: { page, pageSize, totalCount, totalPages }
}

UpdateLocationPayload {
  location_name?, city?, fax?, callback_number?,
  servicing_address?, facility_name?, group_npi?,
  tin_tax_id?, reminderFNF?, cityMapping?,
  isIntakeProcessingEnabled?, isPharmacyEligible?
}
```

---

## 4. Layer 3 — API Routes

### List/Create: `app/api/admin/locations/route.ts` (104 lines)

| Method | Auth | Behavior |
|--------|------|----------|
| **GET** | Any logged-in user (`getServerSession`) | Parses query params, calls `fetchLocationsFromDB()`, masks response, returns paginated list |
| **POST** | `requireCorporate()` | Validates required fields (`we_infuse_id`, `city`, `facility_name`), defaults `isPharmacyEligible: true`, calls `createLocationInDB()`, masks response, returns 201 |

### Detail/Edit/Delete: `app/api/admin/locations/[id]/route.ts` (143 lines)

| Method | Auth | Behavior |
|--------|------|----------|
| **GET** | `hasAdminPermissions()` (admin or auditor) | Fetches by ObjectId, masks, returns single location |
| **PUT** | `hasAdminPermissions()` | Passes update payload to `updateLocationInDB()`, masks response |
| **DELETE** | `hasAdminPermissions()` | Calls `deleteLocationFromDB()` — **hard delete**, not soft |

### Response Handling

All routes apply `maskAPIResponseData(data, 'generic')` before returning.

### HTTP Status Codes

| Code | Meaning |
|------|---------|
| 200 | Success with data |
| 201 | Created with data (POST) |
| 400 | Missing required fields (POST) |
| 401 | Not logged in |
| 403 | Role/permission check failed |
| 404 | Location not found |
| 500 | Internal server error |

---

## 5. Layer 4 — Database Service

**File:** `services/locations/locationDbService.ts` (282 lines)

This service uses the **native MongoDB driver** (not Mongoose), connecting via `connectToDatabase()` and operating on `db.collection(COLLECTION.PatientLocations)`.

| Function | MongoDB Operation | Notes |
|----------|-------------------|-------|
| `fetchLocationsFromDB(params)` | `.find()` + `.sort()` + `.skip()` + `.limit()` + `.toArray()` + `.countDocuments()` | Search across 5 fields, case-insensitive regex |
| `getLocationByIdFromDB(id)` | `.findOne({ _id: new ObjectId(id) })` | Returns single doc |
| `createLocationInDB(data)` | `.insertOne()` then `.findOne()` | Sets `createdAt` and `updatedAt` manually |
| `updateLocationInDB(id, data)` | `.findOneAndUpdate()` with `$set`, `returnDocument: 'after'` | Sets `updatedAt` manually |
| `deleteLocationFromDB(id)` | `.deleteOne()` | Hard delete, returns boolean |
| `normalizePharmacyEligible()` | In-memory normalization | Defaults `isPharmacyEligible` to `true` if missing — applied after every read |

### Search Query (list endpoint)

```typescript
{
  $or: [
    { location_name: { $regex: search, $options: 'i' } },
    { facility_name: { $regex: search, $options: 'i' } },
    { city: { $regex: search, $options: 'i' } },
    { group_npi: { $regex: search, $options: 'i' } },
    { tin_tax_id: { $regex: search, $options: 'i' } },
  ]
}
```

### Additional Services

**Location Matching Service:** `services/locations/locationMatchingService.ts` (61 lines)

- `normalizeFaxNumber(fax)` — Removes non-digits, strips leading 1 for US numbers.
- `findLocationByFax(fax)` — Uses **Mongoose model** (not native driver) to find a location by normalized fax number.

**City Mapping Lookup:** `findLocationByCityMapping(preferredLocation)` in `locationDbService.ts`

- Two-step regex search on the `cityMapping` array.
- Includes regex injection protection.
- Used by other parts of the system (not the admin CRUD).

---

## 6. Data Model

**Mongoose schema:** `models/Location.ts` (54 lines)
**Collection:** `patientLocations`
**DTO:** `types/locationDTO.ts` (18 lines)

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `we_infuse_id` | Number | Yes | Unique index |
| `location_name` | String | No | |
| `city` | String | Yes | Indexed |
| `facility_name` | String | Yes | Indexed |
| `fax` | String | No | |
| `callback_number` | String | No | |
| `servicing_address` | String | No | |
| `group_npi` | String | No | |
| `tin_tax_id` | String | No | |
| `cityMapping` | String[] | No | Default `[]` — aliases for city matching |
| `reminderFNF` | Boolean | No | |
| `isIntakeProcessingEnabled` | Boolean | No | Default `true` |
| `isPharmacyEligible` | Boolean | No | Default `true` |
| `createdAt` / `updatedAt` | Date | Auto | Mongoose timestamps |

### Indexes

- Single-field: `city`, `facility_name`, `we_infuse_id` (unique), `location_id`, `location_name`
- Text index: `city` + `facility_name`

### TypeScript Interfaces

```typescript
// types/locationDTO.ts
interface LocationDTO {
  _id: string;
  we_infuse_id: number;
  location_name?: string;
  city: string;
  zip: string;
  state: string;
  fax: string;
  callback_number: string;
  servicing_address: string;
  facility_name: string;
  group_npi: string;
  tin_tax_id: string;
  reminderFNF?: boolean;
  cityMapping?: string[];
  isIntakeProcessingEnabled?: boolean;
  isPharmacyEligible?: boolean;
}

// models/Location.ts
interface ILocation extends Omit<LocationDTO, '_id'>, Document {
  createdAt: Date;
  updatedAt: Date;
}

// services/locations/locationDbService.ts
interface UpdateLocationPayload {
  location_name?: string;
  city?: string;
  fax?: string;
  callback_number?: string;
  servicing_address?: string;
  facility_name?: string;
  group_npi?: string;
  tin_tax_id?: string;
  reminderFNF?: boolean;
  cityMapping?: string[];
  isIntakeProcessingEnabled?: boolean;
  isPharmacyEligible?: boolean;
}
```

---

## 7. Authentication & Authorization

| Endpoint | Required Role | Check Mechanism |
|----------|---------------|-----------------|
| GET `/api/admin/locations` (list) | Any logged-in user | `getServerSession()` |
| POST `/api/admin/locations` | Corporate position | `requireCorporate()` → `isCorporateEmployee()` |
| GET `/api/admin/locations/[id]` | Admin or Auditor | `hasAdminPermissions(role)` |
| PUT `/api/admin/locations/[id]` | Admin or Auditor | `hasAdminPermissions(role)` |
| DELETE `/api/admin/locations/[id]` | Admin or Auditor | `hasAdminPermissions(role)` |

---

## 8. UI Components Used

**From `@/components/UI`:**
- `Button` — Primary, secondary, danger variants
- `Card` — Container for content sections
- `LoadingSpinner` — Loading state
- `Pagination` — Page navigation with page size selector
- `Select` — Dropdown for sort options
- `StyledInput` — Text input fields
- `TagsInput` — Multi-value tag input (for cityMapping)
- `Alert` — Error/success messages
- `BackButton` — Navigation back
- `FormItem` — Form field wrapper with error handling

**From `@/components/UICompositions`:**
- `DynamicForm` — Grid-based form layout driven by `formConfig` array

---

## 9. Complete Request Flow — Example: Update Location

```
1. User edits form on /admin/locations/[id] page
   └─ React Hook Form validates fields

2. form.handleSubmit(onSubmit) fires
   └─ onSubmit() calls updateLocation(id, data) from locationService.ts

3. locationService.ts sends PUT /api/admin/locations/[id]
   └─ fetch('/api/admin/locations/' + id, { method: 'PUT', body: JSON.stringify(data) })

4. API route handler (app/api/admin/locations/[id]/route.ts)
   ├─ getServerSession() → verify user is logged in
   ├─ hasAdminPermissions(role) → verify admin/auditor role
   ├─ Parse request body
   └─ Call updateLocationInDB(id, updateData)

5. locationDbService.ts → updateLocationInDB()
   ├─ connectToDatabase() → get native MongoDB { db, client }
   ├─ db.collection('patientLocations').findOneAndUpdate(
   │     { _id: new ObjectId(id) },
   │     { $set: { ...updateData, updatedAt: new Date() } },
   │     { returnDocument: 'after' }
   │   )
   └─ Return updated document

6. API route masks response
   └─ maskAPIResponseData(location, 'generic')

7. JSON response sent back to browser
   └─ { location: maskedLocationData }

8. Page component shows success Alert
   └─ Clears after 3 seconds
```

---

## 10. Key Observations

1. **Native driver, not Mongoose** — Despite having a Mongoose model defined in `models/Location.ts`, the DB service uses the native MongoDB driver via `connectToDatabase()`. The Mongoose model is only used by `locationMatchingService.ts` for fax-based lookups. This is part of the hybrid pattern (see `mongodb-architecture.md`).

2. **No client-side caching** — Every navigation or filter change triggers a fresh `fetch()`. No SWR, React Query, or similar library.

3. **Authorization is split** — The list endpoint (GET) only requires authentication, but all single-record operations (GET by id, PUT, DELETE) require admin/auditor role. POST requires the "corporate" position.

4. **Hard delete** — `deleteLocationFromDB()` uses `deleteOne()`, not a soft-delete pattern. The schema has no `deletedAt` field.

5. **Manual timestamps in DB service** — Even though the Mongoose schema has `timestamps: true`, the native driver doesn't benefit from it. `createLocationInDB` and `updateLocationInDB` set `createdAt`/`updatedAt` manually.

6. **Normalization after read** — `normalizePharmacyEligible()` patches older documents that don't have `isPharmacyEligible` set, defaulting to `true` in memory after every fetch.

---

## 11. File Reference

| Layer | File | Lines |
|-------|------|-------|
| List page | `app/admin/locations/page.tsx` | 272 |
| Create page | `app/admin/locations/new/page.tsx` | 302 |
| Edit page | `app/admin/locations/[id]/page.tsx` | 502 |
| List page styles | `app/admin/locations/styles.module.css` | 191 |
| Edit page styles | `app/admin/locations/[id]/styles.module.css` | 113 |
| Create page styles | `app/admin/locations/new/styles.module.css` | 91 |
| Client service | `services/locations/locationService.ts` | 152 |
| DB service | `services/locations/locationDbService.ts` | 282 |
| Matching service | `services/locations/locationMatchingService.ts` | 61 |
| API list/create | `app/api/admin/locations/route.ts` | 104 |
| API detail/edit/delete | `app/api/admin/locations/[id]/route.ts` | 143 |
| Mongoose model | `models/Location.ts` | 54 |
| DTO type | `types/locationDTO.ts` | 18 |
| Collection constant | `utils/constants/collections.ts` | 33 |
| DynamicForm component | `components/UICompositions/DynamicForm/DynamicForm.tsx` | 73 |
