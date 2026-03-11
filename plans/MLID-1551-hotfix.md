# MLID-1551 Hotfix — Show User Name Instead of Email

## QA Feedback

**Reporter:** Olha Turovych (QA)
**Comment:** "To display not user's email but user's name who updated hospital."

**Scope:** Both the **Hospital List page** and the **Hospital Detail page** currently display `updatedBy` as a raw email string. The acceptance criteria says: *"Last Updated: 12/10/2025 by Jenifer D."* — i.e., a display name, not an email.

---

## Root Cause

The `updatedBy` field in the `hospitalSystems` collection stores `session.user.email` (a string like `"alejandro.gomez@fortegrp.com"`). The API returns this value as-is, and both pages render it directly.

### Affected Files

| Layer | File | Line(s) | Issue |
|-------|------|---------|-------|
| Model | `apps/web/models/HospitalSystem.ts` | 21-22 | `createdBy`/`updatedBy` stored as plain email string |
| Service | `apps/web/services/mongodb/hospitalSystem.ts` | 105, 145-146, 167 | Stores `userEmail` directly, returns `updatedBy` as-is |
| API (POST) | `apps/web/app/api/hospitals/route.ts` | 111 | Passes `session.user.email` |
| API (PUT) | `apps/web/app/api/hospitals/[id]/route.ts` | 114 | Passes `session.user.email` |
| API (DELETE) | `apps/web/app/api/hospitals/[id]/route.ts` | 171 | Passes `session.user.email` |
| Client types | `apps/web/services/hospitalSystems/hospitalSystemService.ts` | 12, 37 | `updatedBy?: string` |
| List page | `apps/web/app/hospital-system-agreements/page.tsx` | 236 | `{hs.updatedBy \|\| 'N/A'}` |
| Detail page | `apps/web/app/hospital-system-agreements/[id]/page.tsx` | 160 | `by {hospital.updatedBy \|\| 'N/A'}` |

---

## Approach

**Strategy: Resolve email → user name at the service layer (server-side)**

Keep storing `email` in `updatedBy` (it's a stable, unique identifier). Resolve the display name via a lookup against the Users collection before returning data to the client. Return a new `updatedByName` field alongside `updatedBy`.

This is the minimal-change approach — no schema migration, no model changes, no client-side lookup overhead.

---

## Implementation Plan

### Step 1: Add `resolveUserName` helper to service layer

**File:** `apps/web/services/mongodb/hospitalSystem.ts`

- Import the User model (or user service) to look up a user by email.
- Create a small helper: `resolveUserName(email: string | undefined): Promise<string | undefined>` that returns the user's `name` field from the `users` collection matched by `email`.
- For the list endpoint (multiple records), batch-resolve: collect unique emails, do a single `User.find({ email: { $in: [...] } })` query, build a map, and attach names.

### Step 2: Update `getHospitalSystems` (list endpoint)

**File:** `apps/web/services/mongodb/hospitalSystem.ts` — `getHospitalSystems()`

- After fetching hospitals, collect all unique `updatedBy` emails.
- Batch-query the Users collection for those emails.
- Map the resolved `name` onto each list item as `updatedByName`.
- Update `HospitalSystemListItem` type to include `updatedByName?: string`.

### Step 3: Update `getHospitalSystemById` (detail endpoint)

**File:** `apps/web/services/mongodb/hospitalSystem.ts` — `getHospitalSystemById()`

- After fetching the hospital, resolve `updatedBy` email → user name.
- Return the result with the additional `updatedByName` field.
- The return type is currently `HospitalSystemDocument | null`. We need a mapped response type that includes `updatedByName`.

### Step 4: Update client-side types

**File:** `apps/web/services/hospitalSystems/hospitalSystemService.ts`

- Add `updatedByName?: string` to `HospitalSystemDetail` and `HospitalSystemListItem` interfaces.

### Step 5: Update List page UI

**File:** `apps/web/app/hospital-system-agreements/page.tsx`

- Line 236: Change `{hs.updatedBy || 'N/A'}` → `{hs.updatedByName || hs.updatedBy || 'N/A'}`
- This provides graceful fallback: name → email → N/A.

### Step 6: Update Detail page UI

**File:** `apps/web/app/hospital-system-agreements/[id]/page.tsx`

- Line 160: Change `by {hospital.updatedBy || 'N/A'}` → `by {hospital.updatedByName || hospital.updatedBy || 'N/A'}`

### Step 7: Update existing tests + add new tests

- Update existing tests for `getHospitalSystems` and `getHospitalSystemById` to assert the new `updatedByName` field.
- Add tests for the `resolveUserName` helper (found/not-found/undefined cases).
- Update list and detail page component tests to verify name is rendered.

---

## Test Plan (TDD)

### RED phase — write failing tests first:

1. **Service layer test** (`hospitalSystem.test.ts`):
   - `getHospitalSystems` returns `updatedByName` resolved from Users collection
   - `getHospitalSystems` returns `updatedByName` as `undefined` when user not found
   - `getHospitalSystemById` returns `updatedByName` resolved from Users collection

2. **List page test** (`page.test.tsx`):
   - Renders user name in "Updated By" column when `updatedByName` is present
   - Falls back to email when `updatedByName` is absent

3. **Detail page test** (`page.test.tsx`):
   - Renders user name in "Last Updated by" when `updatedByName` is present
   - Falls back to email when `updatedByName` is absent

### GREEN phase — implement minimal code to pass tests

### REFACTOR phase — clean up, ensure lint/types pass

---

## Out of Scope

- Changing the schema to store `updatedBy` as ObjectId (would require migration of existing data)
- Changing `createdBy` display (not shown in any current UI)
- Other features that store email in `updatedBy` (biosimilars, treatments, etc.)

---

## Files Modified (Summary)

| File | Change |
|------|--------|
| `apps/web/services/mongodb/hospitalSystem.ts` | Add user name resolution logic to `getHospitalSystems` and `getHospitalSystemById` |
| `apps/web/services/hospitalSystems/hospitalSystemService.ts` | Add `updatedByName` to client types |
| `apps/web/app/hospital-system-agreements/page.tsx` | Display `updatedByName` with email fallback |
| `apps/web/app/hospital-system-agreements/[id]/page.tsx` | Display `updatedByName` with email fallback |
| Test files for the above | Updated + new tests |
