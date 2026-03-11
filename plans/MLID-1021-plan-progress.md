# MLID-1021 - Epic Progress Tracker

## Purpose

This file tracks the implementation progress of the **MLID-1021 (Hospital System Agreements)** epic. It shows the status of each subtask, whether its code has been merged into the epic integration branch (`epic/MLID-1021-hospital-systems-340B`), and whether the deliverable group it belongs to has been submitted as a PR to `develop` for staging deployment and QA testing.

For the full deliverable grouping rationale and git branch strategy, see [MLID-1021-git-branch-strategy.md](./MLID-1021-git-branch-strategy.md).

---

## Deliverable 1: Hospital List View

| Task ID | Summary | Status | Branch | Merged to Epic | Notes |
|---------|---------|:------:|--------|:--------------:|-------|
| MLID-1543 | Create MongoDB schemas for Hospital System Agreements | Done | `feature/MLID-1543-create-mongodb-schema` | Yes | |
| MLID-1544 | Create API endpoints for Hospital System CRUD operations | Done | `feature/MLID-1544-hospital-system-crud-api` | Yes | Includes service layer |
| MLID-1547 | Create service layer for Hospital System management | Done | `feature/MLID-1547-hospital-system-service-layer` | Yes | Added getProviders + bulkUpdateProviders (5 other functions already in MLID-1544) |
| MLID-1559 | Add Hospital System entry to main navigation menu | Done | `feature/MLID-1559-hospital-system-nav-entry` | Yes | Icon duplication fixed: Provider Mgmt → 🧑‍⚕️, Insurance Plans → 🛡️ |
| MLID-1548 | Create Hospital System Agreements list page | Done | `feature/MLID-1548-hospital-list-page` | Yes | Also extended shared Pagination with First/Last buttons (MUI icons) |
| MLID-1549 | Implement search functionality on hospital list | Done | `feature/MLID-1549-hospital-search-functionality` | Yes | Native input + SearchIcon, 500ms debounce, URL persistence |
| MLID-1550 | Create Add New Hospital modal | Done | `feature/MLID-1550-add-new-hospital-modal` | Yes | Client service, modal component, page wiring, security hardening (type guard + 1024 max length) |

**D1 PR to develop:** Completed

---

## Deliverable 2: Hospital Detail View

| Task ID | Summary | Status | Branch | Merged to Epic | Notes |
|---------|---------|:------:|--------|:--------------:|-------|
| MLID-1546 | Create API endpoint for Provider NPI list retrieval | Done | `feature/MLID-1546-provider-npi-api` | Yes | GET /api/hospitals/:id/providers with pagination + search; also added input sanitization to GET /api/hospitals |
| MLID-1551 | Create Hospital Detail page layout | Done | `feature/MLID-1551-hospital-detail-page` | Yes | Breadcrumb, header card, tabs (Providers/Bulk Update), client service fetchHospitalSystemById, 17 tests |
| MLID-1552 | Create Edit Hospital modal | Done | `feature/MLID-1552-edit-hospital-modal` | Yes | Refactored AddNewHospitalModal → AddEditHospitalModal (dual-mode: add/edit), added updateHospitalSystem client service, wired into detail page |
| MLID-1553 | Implement Providers tab with NPI table | Done | `feature/MLID-1553-providers-tab-npi-table` | Yes | NPI table with pagination, search, URL param persistence |
| MLID-1741 | Implement feature flag for "Hospital System" menu entry | Done | `feature/MLID-1741-hospital-systems-feature-flag` | Yes | Flag `HOSPITAL_SYSTEMS`, hide nav when OFF (no route blocking needed) |

**D2 PR to develop:** Completed

---

## QA Hotfixes

| Task ID | Summary | Status | Branch | Merged to Epic | Notes |
|---------|---------|:------:|--------|:--------------:|-------|
| MLID-1551 | Display user name instead of email in "Updated By" | Done | `fix/MLID-1551-hospital-detail-show-username` | Yes | Batch-resolve updatedBy email → user name via Users collection lookup; affects list page + detail page; graceful fallback (name → email → N/A); 96 tests, all files >80% coverage |
| - | Remove deprecated `providerNPIs` field from model | Done | `fix/MLID-1021-remove-providerNPIs-field` | Yes | Prevents Mongoose from creating empty `providerNPIs: []` on new docs |

---

## Deliverable 3: Bulk Update

| Task ID | Summary | Status | Branch | Merged to Epic | Notes |
|---------|---------|:------:|--------|:--------------:|-------|
| MLID-1545 | Create API endpoint for Provider NPI bulk update | Done | `feature/MLID-1545-bulk-update-providers-api` | Yes | PUT /api/hospitals/:id/providers/bulk — replaces full provider list, NPI validation, 15 tests |
| MLID-1557 | Implement NPI validation logic | Done | `feature/MLID-1555-csv-upload-bulk-npi` | Yes | Implemented as shared utility in MLID-1555: `utils/validation/npiValidation.ts` (isValidNpi, validateNpiList), 19 tests, 100% coverage |
| MLID-1554 | Create Bulk Update tab layout | Done | `feature/MLID-1554-bulk-update-tab-layout` | Yes | Warning banner, CSV upload drop zone (CloudUpload icon), paste NPI textarea, Cancel/Confirm buttons, 16 unit tests + 2 integration tests, 100% coverage |
| MLID-1555 | Implement CSV file upload for bulk NPI update | Done | `feature/MLID-1555-csv-upload-bulk-npi` | Yes | FileDropzone CSV upload, paste NPI textarea with live validation, NPI validation utility (shared), CSV parser (1-col/2-col), ValidationSummary component, Confirm & Replace button enabled on valid NPIs, 69 tests, 100% coverage |
| MLID-1556 | Implement paste NPI list functionality | Done | `feature/MLID-1556-paste-npi-list` | Yes | MUI TextField multiline, CR/CRLF line endings, Figma alignment (FileDropzone icon, valid badge, button layout), mutual input exclusivity, 37 tests |
| MLID-1558 | Implement Confirm & Replace flow | Done | `feature/MLID-1558-confirm-replace-flow` | Yes | Wired Confirm & Replace button: API call with loading/success/error toasts, onSuccess navigates to Providers tab, API route hardened (NPI trim, 10K max, field stripping), bulkReplaceProviders client service, 17 new tests, 100% stmt coverage |

**D3 PR to develop:** In QA (PR merged to develop, awaiting QA feedback Monday 2/23)

---

## Task List

### Deliverable 1: Hospital List View (17 SP)
- MLID-1543 — Create MongoDB schemas for Hospital System Agreements (2 SP)
- MLID-1544 — Create API endpoints for Hospital System CRUD operations (3 SP)
- MLID-1547 — Create service layer for Hospital System management (3 SP)
- MLID-1559 — Add Hospital System entry to main navigation menu (2 SP)
- MLID-1548 — Create Hospital System Agreements list page (3 SP)
- MLID-1549 — Implement search functionality on hospital list (2 SP)
- MLID-1550 — Create Add New Hospital modal (2 SP)

### Deliverable 2: Hospital Detail View (12 SP)
- MLID-1546 — Create API endpoint for Provider NPI list retrieval (2 SP)
- MLID-1551 — Create Hospital Detail page layout (3 SP)
- MLID-1552 — Create Edit Hospital modal (2 SP)
- MLID-1553 — Implement Providers tab with NPI table (3 SP)
- MLID-1741 — Implement feature flag for "Hospital System" menu entry (2 SP)

### Deliverable 3: Bulk Update (15 SP)
- MLID-1545 — Create API endpoint for Provider NPI bulk update (3 SP)
- MLID-1557 — Implement NPI validation logic (2 SP)
- MLID-1554 — Create Bulk Update tab layout (2 SP)
- MLID-1555 — Implement CSV file upload for bulk NPI update (3 SP)
- MLID-1556 — Implement paste NPI list functionality (3 SP)
- MLID-1558 — Implement Confirm & Replace flow (2 SP)

---

## Agent Workflow for Next Tasks

When picking up the next task from this epic, follow these steps:

1. **Act as an expert SWE** in React, Next.js, and MongoDB.
2. **Read this document** as the main plan to understand the overall progress and context.
3. **Read the git branch strategy** from `docs/agomez/plans/epic-git-branch-strategy.md`.
4. **Read the Branch task in Jira** when asked to work on a new task, read the ticket in Jira to get context.
5. **Create a feature branch** for the next task (base branch should be `epic/MLID-1021-hospital-systems-340B`).
6. **Analyze the web app** — explore the existing codebase, components, services, and patterns relevant to the task.
7. **Design an implementation plan** and write it to `docs/agomez/plans/MLID-[number].md`.
8. **Wait for approval** on the plan before making any code modifications or implementations.
9. **Ensure test coverage** — all created or updated files must have at least **80% test coverage**.
10. **Check types before commit** — run `npm run types:check` from `apps/web/` and fix any errors before committing.

---

## Non-Code Tasks (already completed)

| Task ID | Summary | Status |
|---------|---------|:------:|
| MLID-1241 | Create Design for 340B Hospital System - Provider Upload | Done |
| MLID-1422 | Grooming & Story Points | Done |
