# Epic MLID-1021: Hospital System Agreements (340B Provider Screens)

## Overview

This document outlines the user stories for implementing the Hospital System Agreements feature, which allows operations users to manage 340B provider lists through a bulk CSV process.

**Epic:** MLID-1021 - Referral Tracker - 340B Table for Lookup (Hospital System Agreements)

**Purpose:** Enable operations users to upload and manage provider NPI lists received from hospital systems for 340B eligibility tracking.

---

## Access Control

**This feature is restricted to:**

- Users with `Admin` role, OR
- Users with `Corporate` position in their positions array

**Implementation Notes:**

- Service layer should include a helper function `canAccessHospitalAgreements(user)` that returns true if user has Admin role OR Corporate position
- All API endpoints must validate user access before processing requests (return 403 Forbidden if unauthorized)
- UI pages should check access on load and redirect unauthorized users or show access denied message
- Navigation menu item should only be visible to authorized users

---

## Screenshots Analysis

Based on the Figma designs, the feature consists of:

1. **Hospital System Agreements List Page** - Table with Hospital name, Number of Providers, Last Updated, Updated By columns. Includes search bar, "Add new" button, and pagination.

2. **Add New Hospital Modal** - Simple modal with Hospital name input field and "Add Hospital" button.

3. **Hospital Detail Page - Providers Tab** - Shows hospital header with name, provider count, last updated info. Has tabs for "Providers" and "Bulk Update". Providers tab shows a table with NPI and Provider Name columns.

4. **Edit Hospital Modal** - Similar to Add modal but for editing existing hospital name with "Update Hospital" button.

5. **Bulk Update Tab** - Warning banner about replacing entire provider list, CSV upload area (drag & drop), textarea for pasting NPI list from Excel, Cancel and "Confirm & Replace" buttons.

6. **Bulk Update Tab (With Valid Data)** - Shows validation badge "Found X Valid Entries" and enabled "Confirm & Replace" button.

---

## User Stories

### Backend/Database Layer

#### Story 1: Create MongoDB schemas for Hospital System Agreements

**Summary:** Create MongoDB models for Hospital Systems and Provider NPIs

**Description:**
As a developer, I need to create the MongoDB schemas to store hospital system agreements and their associated provider NPIs.

**Acceptance Criteria:**
- Create `HospitalSystem` model with fields:
  - `_id` (ObjectId, primary key)
  - `name` (string, required)
  - `providerNPIs` (array of strings)
  - `createdAt` (date)
  - `updatedAt` (date)
  - `updatedBy` (reference to user)
  - `createdBy` (reference to user)
  - `deletedAt` (date, nullable - for soft delete)
- Create appropriate indexes:
  - Text index on `name` field for search functionality
- NPI should be validated as 10-digit numeric string
- Model includes virtual for provider count
- Note: An NPI can belong to multiple hospital systems (many-to-many relationship)

**Unit Tests:**
- Test model validation (required fields, NPI format validation)
- Test virtual provider count calculation
- Test index creation

**Story Points:** 2

---

#### Story 2: Create API endpoints for Hospital System CRUD operations

**Summary:** Implement REST API for hospital system management

**Description:**
As a developer, I need API endpoints to manage hospital systems (create, read, update, delete).

**Acceptance Criteria:**
- `GET /api/hospitals` - List hospitals with pagination, search, and sorting
  - Supports query params: `search`, `page`, `limit`, `sortBy`, `sortOrder`
  - Returns: hospital name, provider count, lastUpdated, updatedBy
- `GET /api/hospitals/:id` - Get single hospital with its providers
- `POST /api/hospitals` - Create new hospital
  - Validates name is not empty
  - Returns created hospital
- `PUT /api/hospitals/:id` - Update hospital name
  - Validates name is not empty
- `DELETE /api/hospitals/:id` - Soft delete hospital (sets `deletedAt` timestamp, does not remove from database)
  - Note: This endpoint will not be exposed in the UI but is available for future use or admin purposes
- All endpoints require authentication
- All endpoints must validate user access (Admin role OR Corporate position) - return 403 if unauthorized
- All endpoints track `updatedAt` and `updatedBy`

**Unit Tests:**
- Test GET /api/hospitals returns paginated list
- Test GET /api/hospitals with search query filters results
- Test GET /api/hospitals/:id returns hospital details
- Test GET /api/hospitals/:id returns 404 for non-existent hospital
- Test POST /api/hospitals creates hospital successfully
- Test POST /api/hospitals returns 400 for empty name
- Test PUT /api/hospitals/:id updates hospital name
- Test DELETE /api/hospitals/:id soft deletes hospital (sets deletedAt)
- Test GET endpoints exclude soft-deleted hospitals
- Test authentication is required for all endpoints
- Test returns 403 for users without Admin role or Corporate position

**Story Points:** 3

---

#### Story 3: Create API endpoint for Provider NPI bulk update

**Summary:** Implement bulk NPI update endpoint with validation

**Description:**
As a developer, I need an API endpoint to replace all provider NPIs for a hospital system.

**Acceptance Criteria:**
- `PUT /api/hospitals/:id/providers/bulk` - Replace entire provider list
- Accepts array of NPI strings in request body
- Validates each NPI:
  - Must be exactly 10 digits
  - Must contain only numeric characters
  - Returns validation errors with specific NPI failures
- Replaces entire existing list (not append)
- Updates `updatedAt` and `updatedBy` on hospital
- Returns updated hospital with new provider count
- Validates user access (Admin role OR Corporate position) - return 403 if unauthorized

**Unit Tests:**
- Test bulk update replaces entire provider list
- Test validation rejects NPIs with less than 10 digits
- Test validation rejects NPIs with more than 10 digits
- Test validation rejects NPIs with non-numeric characters
- Test returns validation errors with specific NPI failures
- Test updates `updatedAt` and `updatedBy` fields
- Test returns 404 for non-existent hospital
- Test returns 403 for users without Admin role or Corporate position

**Story Points:** 3

---

#### Story 4: Create API endpoint for Provider NPI list retrieval

**Summary:** Implement paginated provider list endpoint

**Description:**
As a developer, I need an API endpoint to retrieve the list of provider NPIs for a hospital with pagination and search.

**Acceptance Criteria:**
- `GET /api/hospitals/:id/providers` - Get providers for a hospital
- Supports pagination: `page`, `limit` (default 25)
- Supports search by NPI
- Returns: NPI, provider count total, pagination metadata
- Note: Provider name display is out of scope for this story (per epic: "at this time we will not be showing/accepting provider name")
- Validates user access (Admin role OR Corporate position) - return 403 if unauthorized

**Unit Tests:**
- Test returns paginated list of NPIs
- Test pagination with page and limit parameters
- Test search filters NPIs correctly
- Test returns correct total count
- Test returns 404 for non-existent hospital
- Test returns 403 for users without Admin role or Corporate position

**Story Points:** 2

---

#### Story 5: Create service layer for Hospital System management

**Summary:** Implement business logic layer for hospital operations

**Description:**
As a developer, I need a service layer to encapsulate business logic for hospital system management.

**Acceptance Criteria:**
- Create `services/mongodb/hospitalSystem.ts`
- Implement functions:
  - `canAccessHospitalAgreements(user)` - returns true if user has Admin role OR Corporate position
  - `getHospitalSystems(options)` - with search, pagination (excludes soft-deleted)
  - `getHospitalSystemById(id)` - excludes soft-deleted
  - `createHospitalSystem(data, userId)`
  - `updateHospitalSystem(id, data, userId)`
  - `deleteHospitalSystem(id)` - soft delete (sets `deletedAt` timestamp)
  - `bulkUpdateProviders(hospitalId, npis, userId)`
  - `getProviders(hospitalId, options)` - with search, pagination
- Include proper error handling and validation
- Access control helper function should check: `user.role === 'Admin' || user.positions?.includes('Corporate')`

**Unit Tests:**
- Test `canAccessHospitalAgreements` returns true for Admin role
- Test `canAccessHospitalAgreements` returns true for Corporate position
- Test `canAccessHospitalAgreements` returns false for non-Admin without Corporate position
- Test `getHospitalSystems` returns paginated results
- Test `getHospitalSystems` filters by search term
- Test `getHospitalSystemById` returns hospital or throws not found
- Test `createHospitalSystem` creates and returns new hospital
- Test `createHospitalSystem` throws error for empty name
- Test `updateHospitalSystem` updates hospital name
- Test `deleteHospitalSystem` sets deletedAt timestamp (soft delete)
- Test `getHospitalSystems` excludes soft-deleted hospitals
- Test `getHospitalSystemById` returns null for soft-deleted hospital
- Test `bulkUpdateProviders` replaces provider list
- Test `bulkUpdateProviders` validates NPI format
- Test `getProviders` returns paginated NPI list

**Story Points:** 3

---

### Frontend - Hospital List Page

> **Note:** All frontend stories must use the existing UI components from `components/UI/` and `components/UICompositions/`. Do not introduce new UI libraries or create custom controls when existing ones are available.

> **Access Control:** All frontend pages must check user access on load (Admin role OR Corporate position). Unauthorized users should be redirected or shown an access denied message. Navigation menu item should only be visible to authorized users.

#### Story 6: Create Hospital System Agreements list page

**Summary:** Build the main list page for hospital system agreements

**Description:**
As an operations user, I need to view a list of all hospital system agreements so I can manage 340B provider lists.

**Acceptance Criteria:**
- Create page at `/hospital-system-agreements`
- Use existing UI components (Table, Pagination, Button, Card, etc.)
- Check user access on page load (Admin role OR Corporate position)
- Redirect unauthorized users or show access denied message
- Display table with columns: Hospital name, Number of Providers, Last Updated, Updated By
- Hospital name is clickable and navigates to detail page
- Table shows "Showing X - Y of Z results" pagination info
- Pagination controls: first, previous, page number, next, last
- Default 25 results per page with dropdown to change
- Empty state message when no hospitals exist
- Loading state while fetching data
- Add navigation menu item to access this page (only visible to authorized users)

**Unit Tests:**
- Test access check on page load
- Test unauthorized users see access denied or are redirected
- Test table renders with correct columns
- Test hospital name links to detail page
- Test pagination displays correct info
- Test empty state renders when no data
- Test loading state renders while fetching
- Test navigation menu item hidden for unauthorized users

**Story Points:** 3

---

#### Story 7: Implement search functionality on hospital list

**Summary:** Add real-time search filter for hospital names

**Description:**
As an operations user, I need to search for hospitals by name so I can quickly find specific agreements.

**Acceptance Criteria:**
- Use existing Input component from `components/UI/MUI`
- Search input field with placeholder "Search"
- Filters hospitals in real-time as user types (debounced)
- Search matches hospital name (case-insensitive)
- Clear search results when input is cleared
- Search preserves pagination (resets to page 1 on new search)

**Unit Tests:**
- Test search input renders with placeholder
- Test search triggers API call with debounce
- Test search resets pagination to page 1
- Test clearing search shows all results

**Story Points:** 2

---

#### Story 8: Create Add New Hospital modal

**Summary:** Implement modal dialog for creating new hospital entries

**Description:**
As an operations user, I need to add new hospital system agreements so I can start tracking their 340B providers.

> **Note:** This modal is only accessible from the hospital list page, which already enforces access control.

**Acceptance Criteria:**
- Use existing Modal, Input, and Button components from `components/UI/`
- "Add new" button on list page opens modal
- Modal titled "Add new Hospital"
- Hospital name input field with label and placeholder
- Validation: name is required
- Display validation errors inline
- "Add Hospital" button submits form
- Close (X) button dismisses modal without saving
- On success: modal closes, list refreshes, new hospital appears
- Loading state on submit button while processing

**Unit Tests:**
- Test modal opens when "Add new" button clicked
- Test modal displays correct title and input field
- Test validation shows error for empty name
- Test submit button shows loading state
- Test modal closes on successful submission
- Test modal closes when X button clicked

**Story Points:** 2

---

### Frontend - Hospital Detail Page

> **Access Control:** The hospital detail page must check user access on load (Admin role OR Corporate position). Unauthorized users should be redirected or shown an access denied message.

#### Story 9: Create Hospital Detail page layout

**Summary:** Build the hospital detail page with header and tab navigation

**Description:**
As an operations user, I need to view and manage a specific hospital's details and providers.

**Acceptance Criteria:**
- Use existing UI components (Card, Tabs, Button, Breadcrumb, etc.)
- Page at `/hospital-system-agreements/:id`
- Check user access on page load (Admin role OR Corporate position)
- Redirect unauthorized users or show access denied message
- Breadcrumb navigation: "< Hospital System Agreements" link back to list
- Header displays:
  - Hospital name (large text)
  - Provider count (e.g., "45 Providers")
  - Last updated info (e.g., "Last Updated: 12/10/2025 by Jenifer D.")
- "Edit Hospital" button in header
- Tab navigation with "Providers" and "Bulk Update" tabs
- "Providers" tab is default/active on page load
- 404 handling if hospital not found

**Unit Tests:**
- Test access check on page load
- Test unauthorized users see access denied or are redirected
- Test breadcrumb navigation renders and links correctly
- Test header displays hospital name, provider count, and last updated info
- Test "Edit Hospital" button renders
- Test tab navigation renders with correct tabs
- Test "Providers" tab is active by default
- Test 404 page renders for non-existent hospital

**Story Points:** 3

---

#### Story 10: Create Edit Hospital modal

**Summary:** Implement modal dialog for editing hospital name

**Description:**
As an operations user, I need to edit a hospital's name to correct errors or update information.

> **Note:** This modal is only accessible from the hospital detail page, which already enforces access control.

**Acceptance Criteria:**
- Use existing Modal, Input, and Button components from `components/UI/`
- "Edit Hospital" button opens modal
- Modal titled "Edit Hospital"
- Hospital name input pre-populated with current name
- Validation: name is required
- "Update Hospital" button saves changes
- Close (X) button dismisses without saving
- On success: modal closes, header updates with new name
- Loading state while processing

**Unit Tests:**
- Test modal opens when "Edit Hospital" button clicked
- Test modal displays correct title
- Test input is pre-populated with current hospital name
- Test validation shows error for empty name
- Test submit button shows loading state
- Test modal closes and header updates on successful submission
- Test modal closes when X button clicked without saving

**Story Points:** 2

---

#### Story 11: Implement Providers tab with NPI table

**Summary:** Build the providers list view with search and pagination

**Description:**
As an operations user, I need to view the list of provider NPIs associated with a hospital.

> **Note:** This tab is part of the hospital detail page, which already enforces access control.

**Acceptance Criteria:**
- Use existing Table, Input, and Pagination components from `components/UI/`
- Table displays NPI column (Provider Name column visible but may be empty per epic note)
- Search input filters NPIs in real-time
- Pagination: "Showing X - Y of Z results"
- Adjustable results per page (default: 25)
- Empty state when no providers exist
- Loading state while fetching

**Unit Tests:**
- Test table renders NPI column
- Test search input filters NPIs
- Test pagination displays correct info
- Test results per page dropdown works
- Test empty state renders when no providers
- Test loading state renders while fetching

**Story Points:** 3

---

### Frontend - Bulk Update Feature

> **Access Control:** The bulk update feature is part of the hospital detail page, which requires Admin role OR Corporate position. Access is inherited from the parent page.

#### Story 12: Create Bulk Update tab layout

**Summary:** Build the bulk update tab with warning and input areas

**Description:**
As an operations user, I need a bulk update interface to replace the entire provider list efficiently.

> **Note:** This tab is part of the hospital detail page, which already enforces access control.

**Acceptance Criteria:**
- Use existing UI components (Alert/Banner, Button, Card, etc.)
- Warning banner with yellow background:
  - Warning icon
  - "Bulk Update Mode" title
  - "Pasting a new list below will replace the entire existing provider list." message
- Two input methods displayed:
  1. CSV file upload section
  2. Paste NPI list textarea
- "Cancel" button returns to Providers tab
- "Confirm & Replace" button (disabled until valid input)
- Clear visual separation between upload and paste sections

**Unit Tests:**
- Test warning banner renders with correct message
- Test CSV upload section renders
- Test paste textarea section renders
- Test "Cancel" button returns to Providers tab
- Test "Confirm & Replace" button is disabled initially

**Story Points:** 2

---

#### Story 13: Implement CSV file upload for bulk NPI update

**Summary:** Add drag-and-drop CSV upload functionality

**Description:**
As an operations user, I need to upload a CSV file containing NPIs to update the provider list. The uploaded list will completely replace the existing provider list for the hospital (not append).

> **Note:** This feature is part of the hospital detail page, which already enforces access control.

**Acceptance Criteria:**
- Use existing FileUpload/Dropzone component if available, or create following existing patterns
- Bulk update replaces the entire existing provider list (not append)
- Upload area with:
  - Cloud upload icon
  - "Click to Upload or Drag and Drop" text
  - "Supported formats: CSV" label
  - "Maximum file size: 5MB" label
- Drag and drop functionality
- Click to open file picker
- File validation:
  - Must be CSV format
  - Maximum 5MB file size
  - Must contain NPI column
- Parse CSV and extract NPIs
- Display validation results (valid count / errors)
- Error messages for invalid files or parsing failures

**Unit Tests:**
- Test upload area renders with correct labels
- Test drag and drop triggers file handling
- Test click opens file picker
- Test rejects non-CSV files
- Test rejects files larger than 5MB
- Test parses valid CSV and extracts NPIs
- Test parsed NPIs replace existing list (not append)
- Test displays validation results
- Test displays error for parsing failures

**Story Points:** 3

---

#### Story 14: Implement paste NPI list functionality

**Summary:** Add textarea for pasting NPI list from Excel

**Description:**
As an operations user, I need to paste a list of NPIs directly from Excel to quickly update the provider list. The pasted list will completely replace the existing provider list for the hospital (not append).

> **Note:** This feature is part of the hospital detail page, which already enforces access control.

**Acceptance Criteria:**
- Use existing Textarea/Input component from `components/UI/MUI`
- Bulk update replaces the entire existing provider list (not append)
- Textarea labeled "Paste NPI List (from Excel)"
- Accepts one NPI per line
- Handles various line endings (Windows, Unix, Mac)
- Real-time validation as user types/pastes
- Display validation badge (e.g., "Found 6 Valid Entries")
- Highlight or report invalid NPIs
- Enable "Confirm & Replace" button when valid entries exist

**Unit Tests:**
- Test textarea renders with correct label
- Test handles Windows line endings (CRLF)
- Test handles Unix line endings (LF)
- Test handles Mac line endings (CR)
- Test validates NPIs in real-time
- Test pasted NPIs replace existing list (not append)
- Test displays validation badge with valid count
- Test reports invalid NPIs
- Test enables "Confirm & Replace" when valid entries exist

**Story Points:** 3

---

#### Story 15: Implement NPI validation logic

**Summary:** Create client-side NPI validation utility

**Description:**
As a developer, I need a shared validation utility for NPI numbers.

**Acceptance Criteria:**
- Validate NPI format:
  - Exactly 10 characters
  - Only numeric digits (0-9)
- Remove duplicates from input list
- Handle whitespace trimming
- Return validation result with:
  - Array of valid NPIs
  - Array of invalid entries with error reasons
  - Total counts
- Reusable utility for both CSV and paste input

**Unit Tests:**
- Test validates NPI with exactly 10 digits
- Test rejects NPI with less than 10 digits
- Test rejects NPI with more than 10 digits
- Test rejects NPI with non-numeric characters
- Test removes duplicate NPIs
- Test trims whitespace from NPIs
- Test returns correct valid and invalid arrays
- Test returns correct counts

**Story Points:** 2

---

#### Story 16: Implement Confirm & Replace flow

**Summary:** Complete the bulk update submission with confirmation

**Description:**
As an operations user, I need to confirm and submit the bulk update to replace all providers.

> **Note:** This feature is part of the hospital detail page, which already enforces access control. The API endpoint also validates user access.

**Acceptance Criteria:**
- Use existing Button and Toast/Notification components from `components/UI/`
- "Confirm & Replace" button enabled only when valid NPIs exist
- Button click submits bulk update to API
- Loading state during submission
- On success:
  - Display success notification
  - Navigate to Providers tab
  - Show updated provider list
  - Header updates with new count and timestamp
- On error:
  - Display error notification with details
  - Keep user on Bulk Update tab
  - Preserve their input

**Unit Tests:**
- Test "Confirm & Replace" button is disabled without valid NPIs
- Test "Confirm & Replace" button is enabled with valid NPIs
- Test button click triggers API call
- Test loading state displays during submission
- Test success notification displays on successful update
- Test navigates to Providers tab on success
- Test error notification displays on failure
- Test input is preserved on error

**Story Points:** 2

---

### Navigation

#### Story 17: Add Hospital System entry to main navigation menu

**Summary:** Add navigation menu entry for Hospital System feature with access control

**Description:**
As an operations user with appropriate permissions, I need a navigation menu entry to access the Hospital System Agreements feature.

> **Access Control:** This menu entry must only be visible to users with `Admin` role OR users with `Corporate` position in their positions array. Users without these permissions should not see this menu item.

**Acceptance Criteria:**
- Add "Hospital System" entry to the main navigation sidebar menu
- Position the menu item between "Drug Admin Catalog" and "Manage Users"
- Use hospital/medical building icon (🏥) for the menu entry
- Menu item links to `/hospital-system-agreements`
- Menu entry only visible to authorized users:
  - Users with `Admin` role, OR
  - Users with `Corporate` position in their positions array
- Check user permissions when rendering navigation menu
- Hide menu entry completely for unauthorized users (do not show disabled state)

**Unit Tests:**
- Test menu entry renders for users with Admin role
- Test menu entry renders for users with Corporate position
- Test menu entry hidden for users without Admin role or Corporate position
- Test menu entry links to correct URL
- Test menu entry displays correct icon and label

**Story Points:** 2

---

## Summary Table

| # | Story | Points |
|---|-------|--------|
| 1 | MongoDB schemas for Hospital Systems | 2 |
| 2 | API endpoints for Hospital CRUD | 3 |
| 3 | API endpoint for bulk NPI update | 3 |
| 4 | API endpoint for Provider list | 2 |
| 5 | Service layer for Hospital management | 3 |
| 6 | Hospital list page | 3 |
| 7 | Search functionality | 2 |
| 8 | Add New Hospital modal | 2 |
| 9 | Hospital Detail page layout | 3 |
| 10 | Edit Hospital modal | 2 |
| 11 | Providers tab with NPI table | 3 |
| 12 | Bulk Update tab layout | 2 |
| 13 | CSV file upload | 3 |
| 14 | Paste NPI list functionality | 3 |
| 15 | NPI validation utility | 2 |
| 16 | Confirm & Replace flow | 2 |
| 17 | Navigation menu entry | 2 |
| **Total** | | **42** |

---

## Implementation Notes

### Data Model Considerations
- A single Provider (NPI) can be associated with multiple hospital systems (many-to-many relationship)
- Hospital names are not unique - multiple entries with the same name are allowed
- No historical data retention needed - only current state matters
- Provider name is not captured at this time (may be added in future)
- Soft delete pattern: hospitals are not removed from database, `deletedAt` timestamp is set instead
- Delete functionality is available via API but not exposed in the UI

### Technical Stack
- **Frontend:** Next.js App Router, React components
- **Backend:** Next.js API routes
- **Database:** MongoDB with Mongoose ODM
- **Styling:** CSS Modules following existing patterns

### Dependencies
- This epic blocks MLID-1187 (Patient Insurance) and is related to MLID-1492 (340B & Pharmacy Eligible on New Orders)

---

## Open Questions

1. Is there a need for audit logging of bulk updates?

## Resolved Questions

1. ~~Should there be a confirmation dialog before deleting a hospital system?~~ - No delete option in UI
2. ~~Should we implement soft delete or hard delete for hospitals?~~ - Soft delete (sets `deletedAt` timestamp)
3. ~~Should the three-dot action menu on the list page include "Delete" option?~~ - No, delete will not be exposed in UI
