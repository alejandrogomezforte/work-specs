# Pagination Components — Current State & Refactoring Needs

## Summary

There are **3 separate Pagination components** in the codebase plus MUI DataGrid's built-in pagination. They have different designs, different features, and no unified API. The Figma spec for newer pages (users-locations, hospital-system-agreements) uses a design that only exists in the drug-admin-panel module.

---

## Components Inventory

### 1. `components/UI/Pagination/Pagination.tsx` (Shared UI)

**Design:** Previous/Next buttons (`<` / `>`), page number buttons, page size dropdown (custom `Select` component)

**Features:**
- Page numbers with ellipsis
- Page size dropdown (10, 25, 50, 100)
- "Showing X - Y of Z results" text
- Hides entirely when `totalPages <= 1`

**Props:** `currentPage`, `totalPages`, `pageSize`, `totalCount`, `onPageChange`, `onPageSizeChange`

**Used by:**
- `app/appointments/page.tsx`
- `app/ai-chat-dashboard/components/TopUsersTable.tsx`
- `app/ai-chat-dashboard/components/TopCategoriesTable.tsx`
- `app/hospital-system-agreements/page.tsx` (current, needs to switch)
- `components/Modules/admin/jobs-management/JobsManagement.tsx`
- `components/Modules/users-management/users-list/users-list.tsx`
- `components/Modules/provider-management/ProviderOfficeManagement.tsx`
- `components/Modules/messages/MessageList.tsx`
- `components/Modules/patient/components/patientDocumentRecords.tsx`

**Missing vs Figma:** No First/Last page buttons, uses custom Select instead of MUI Select

---

### 2. `components/Modules/drug-admin-panel/components/Pagination.tsx`

**Design:** First/Last buttons (`⟨⟨` / `⟩⟩`), Previous/Next (`⟨` / `⟩`), page numbers with smart ellipsis, MUI `Select` for page size

**Features:**
- "Showing X – Y of Z results" text
- First/Last page navigation buttons
- Smart page number generation (max 7 visible, ellipsis for gaps)
- MUI Select dropdown for page size (10, 25, 50, 100)
- Proper disabled states and aria labels

**Props:** `currentPage`, `totalPages`, `totalCount`, `pageSize`, `onPageChange`, `onPageSizeChange`

**Styles:** `components/Modules/drug-admin-panel/styles.module.css` (`.pagination`, `.paginationInfo`, `.paginationControls`, `.paginationButton`, `.paginationButtonActive`, `.pageSize`)

**Used by:**
- `components/Modules/users-locations/UsersLocationsPanel.tsx`
- `components/Modules/drug-admin-panel/components/BiosimilarsTab.tsx`
- `components/Modules/admin/drug-clinical-review/drug-detail-settings/index.tsx`

**Matches Figma:** Yes — this is the design shown in the Figma spec for newer pages

---

### 3. `components/Modules/insurance-plans/components/Pagination.tsx`

**Design:** Local copy, likely similar to drug-admin-panel version

**Used by:**
- `components/Modules/insurance-plans/InsurancePlansPanel.tsx` (only consumer)

**Has tests:** `components/Modules/insurance-plans/components/__tests__/Pagination.spec.tsx`

---

### 4. MUI DataGrid built-in pagination

**Design:** MUI's `MuiTablePagination` — fully integrated into DataGrid

**Used by:**
- `app/orders-tracker/Orders/Orders.tsx`

**Note:** Client-side pagination only. Not suitable for server-paginated lists.

---

## Recommended Refactoring

### Option A: Promote drug-admin-panel Pagination to shared UI

Move `drug-admin-panel/components/Pagination.tsx` to `components/UI/TablePagination/` (or replace the existing `components/UI/Pagination/`). This way all pages can import from a shared, well-named location.

**Steps:**
1. Create `components/UI/TablePagination/TablePagination.tsx` with the drug-admin-panel code
2. Move styles to `components/UI/TablePagination/styles.module.css`
3. Migrate consumers one at a time:
   - drug-admin-panel → re-export from new location
   - users-locations → update import
   - insurance-plans → replace local copy
   - hospital-system-agreements → update import
   - Optionally migrate older pages (appointments, jobs, etc.)
4. Deprecate the old `components/UI/Pagination` once all consumers are migrated

### Option B: Keep as-is, import from drug-admin-panel

Quick fix for hospital-system-agreements: import the existing component from drug-admin-panel. The import path is misleading but functional.

**Trade-off:** Fast to implement but increases coupling to a feature module.

---

## Decision

**TBD** — Decide whether to do Option A (proper refactor) as a separate ticket or Option B (quick import) for MLID-1548.
