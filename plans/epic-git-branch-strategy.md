# MLID-1021 - Git Branch Strategy & Deliverable Grouping

## Epic Overview

**Epic:** MLID-1021 - Referral Tracker - 340B Table for Lookup (Hospital System Agreements)

The epic contains 18 subtasks. Not every subtask produces a QA-testable deliverable on its own (e.g., a MongoDB schema or a service layer can't be tested by a human in staging). Tasks are grouped into **3 deliverables**, each representing a deployable screen/feature that QA can test end-to-end.

---

## All Subtasks (by Jira key)

| Key | Summary | SP | Status | Deliverable |
|-----|---------|:--:|--------|:-----------:|
| MLID-1241 | Create Design for 340B Hospital System - Provider Upload | 3 | Done | - |
| MLID-1422 | Grooming & Story Points | 2 | Done | - |
| MLID-1543 | Create MongoDB schemas for Hospital System Agreements | 2 | In Dev | D1 |
| MLID-1544 | Create API endpoints for Hospital System CRUD operations | 3 | In Dev | D1 |
| MLID-1547 | Create service layer for Hospital System management | 3 | To Do | D1 (*) |
| MLID-1559 | Add Hospital System entry to main navigation menu | 2 | To Do | D1 |
| MLID-1548 | Create Hospital System Agreements list page | 3 | To Do | D1 |
| MLID-1549 | Implement search functionality on hospital list | 2 | To Do | D1 |
| MLID-1550 | Create Add New Hospital modal | 2 | To Do | D1 |
| MLID-1546 | Create API endpoint for Provider NPI list retrieval | 2 | To Do | D2 |
| MLID-1551 | Create Hospital Detail page layout | 3 | To Do | D2 |
| MLID-1552 | Create Edit Hospital modal | 2 | To Do | D2 |
| MLID-1553 | Implement Providers tab with NPI table | 3 | To Do | D2 |
| MLID-1545 | Create API endpoint for Provider NPI bulk update | 3 | To Do | D3 |
| MLID-1557 | Implement NPI validation logic | 2 | To Do | D3 |
| MLID-1554 | Create Bulk Update tab layout | 2 | To Do | D3 |
| MLID-1555 | Implement CSV file upload for bulk NPI update | 3 | To Do | D3 |
| MLID-1556 | Implement paste NPI list functionality | 3 | To Do | D3 |
| MLID-1558 | Implement Confirm & Replace flow | 2 | To Do | D3 |

> (*) **Note on MLID-1547:** The service layer (`apps/web/services/mongodb/hospitalSystem.ts`) was already created as part of MLID-1544. MLID-1547 may be redundant or may need to cover additional service logic (e.g., NPI-specific helpers). Review before starting — if fully covered, close it with a note.

---

## Deliverable Groups

### D1: Hospital List View (17 SP)

**What QA can test:** Navigate to Hospital System Agreements from the left nav, see a paginated table of hospitals, search by name, create a new hospital via modal.

| Key | Summary | SP |
|-----|---------|:--:|
| MLID-1543 | Create MongoDB schemas for Hospital System Agreements | 2 |
| MLID-1544 | Create API endpoints for Hospital System CRUD operations | 3 |
| MLID-1547 | Create service layer for Hospital System management | 3 |
| MLID-1559 | Add Hospital System entry to main navigation menu | 2 |
| MLID-1548 | Create Hospital System Agreements list page | 3 |
| MLID-1549 | Implement search functionality on hospital list | 2 |
| MLID-1550 | Create Add New Hospital modal | 2 |

**QA test scenarios:**
- Corporate user sees "Hospital Systems" in left nav; non-Corporate user does not
- List page shows paginated table (Hospital name, # Providers, Last Updated, Updated By)
- Search filters hospitals by name in real-time
- "Add New" button opens modal, validates name, creates hospital, row appears in list
- Pagination controls work (page navigation, results per page)

---

### D2: Hospital Detail View (10 SP)

**What QA can test:** Click a hospital name from the list, see the detail page with header, breadcrumb, edit name via modal, and see a paginated Providers tab with NPI list.

| Key | Summary | SP |
|-----|---------|:--:|
| MLID-1546 | Create API endpoint for Provider NPI list retrieval | 2 |
| MLID-1551 | Create Hospital Detail page layout | 3 |
| MLID-1552 | Create Edit Hospital modal | 2 |
| MLID-1553 | Implement Providers tab with NPI table | 3 |

**QA test scenarios:**
- Clicking hospital name navigates to detail page
- Header shows hospital name, provider count, last updated info
- Breadcrumb navigation back to list works
- "Edit Hospital" opens modal, validates name uniqueness, updates name
- Providers tab shows paginated NPI table with search

---

### D3: Bulk Update (15 SP)

**What QA can test:** On the hospital detail page, switch to the Bulk Update tab, upload a CSV or paste NPIs, see validation errors, confirm & replace the provider list.

| Key | Summary | SP |
|-----|---------|:--:|
| MLID-1545 | Create API endpoint for Provider NPI bulk update | 3 |
| MLID-1557 | Implement NPI validation logic | 2 |
| MLID-1554 | Create Bulk Update tab layout | 2 |
| MLID-1555 | Implement CSV file upload for bulk NPI update | 3 |
| MLID-1556 | Implement paste NPI list functionality | 3 |
| MLID-1558 | Implement Confirm & Replace flow | 2 |

**QA test scenarios:**
- Bulk Update tab shows warning about replacing entire list
- CSV upload via drag-and-drop or click-to-upload works
- Paste NPI list from Excel works
- NPI validation catches invalid formats (not 10 digits, non-numeric)
- File size limit (5MB) enforced
- "Confirm & Replace" replaces entire provider list, redirects to Providers tab
- Last Updated and Updated By metadata reflect the change

---

## Git Branch Strategy

### Branch Hierarchy

```
develop (deployed to staging)
  |
  +-- epic/MLID-1021-hospital-systems-340B          (epic integration branch)
        |
        +-- feature/MLID-1543-...                    (sub-task branch)
        +-- feature/MLID-1544-...                    (sub-task branch)
        +-- feature/MLID-XXXX-...                    (sub-task branches)
```

### Flow

```
1. Sub-task branches are created FROM the epic branch
2. Sub-task branches are merged INTO the epic branch (via merge --no-ff or squash)
3. When a deliverable group is complete on the epic branch → PR from epic → develop
4. After PR merges to develop → rebase epic branch on develop before starting next deliverable
```

### Step-by-Step Workflow

#### Starting a new sub-task:
```bash
git checkout epic/MLID-1021-hospital-systems-340B
git pull origin epic/MLID-1021-hospital-systems-340B
git checkout -b feature/MLID-XXXX-short-description
# ... do work, commit ...
```

#### Finishing a sub-task (merge into epic):
```bash
git checkout epic/MLID-1021-hospital-systems-340B
git merge --no-ff feature/MLID-XXXX-short-description
git push origin epic/MLID-1021-hospital-systems-340B
```

#### Finishing a deliverable (PR to develop):
```bash
# All sub-tasks for the deliverable are merged into the epic branch
# Create PR: epic/MLID-1021-hospital-systems-340B → develop
gh pr create --base develop --head epic/MLID-1021-hospital-systems-340B \
  --title "[MLID-1021] D1: Hospital List View" \
  --body "..."
```

#### After PR merges (prepare for next deliverable):
```bash
git checkout develop
git pull origin develop
git checkout epic/MLID-1021-hospital-systems-340B
git rebase develop
git push origin epic/MLID-1021-hospital-systems-340B --force-with-lease
```

### Current State & Catch-Up

The sub-task branches `feature/MLID-1543-...` and `feature/MLID-1544-...` were created before this strategy existed. To align them:

```bash
# 1. Make sure epic branch has latest develop
git checkout epic/MLID-1021-hospital-systems-340B
git rebase develop

# 2. Merge existing sub-task branches into epic
git merge --no-ff feature/MLID-1543-create-mongodb-schema
git merge --no-ff feature/MLID-1544-hospital-system-crud-api

# 3. Push epic branch
git push origin epic/MLID-1021-hospital-systems-340B

# 4. Continue with remaining D1 sub-tasks from epic branch
```

### Branch Naming Convention

| Type | Pattern | Example |
|------|---------|---------|
| Epic | `epic/MLID-1021-hospital-systems-340B` | (already exists) |
| Sub-task | `feature/MLID-XXXX-short-description` | `feature/MLID-1548-hospital-list-page` |

### PR Naming Convention

| Deliverable | PR Title |
|-------------|----------|
| D1 | `[MLID-1021] feat: Hospital List View (D1 - nav, list page, search, add modal)` |
| D2 | `[MLID-1021] feat: Hospital Detail View (D2 - detail page, edit modal, providers tab)` |
| D3 | `[MLID-1021] feat: Bulk Update (D3 - CSV upload, paste NPIs, confirm & replace)` |

---

## Deliverable Dependency Order

```
D1 (Hospital List View)  →  D2 (Hospital Detail View)  →  D3 (Bulk Update)
```

Each deliverable depends on the previous one. They must be merged to develop in order. QA tests each deliverable in staging before the next one is started (or at least before the next PR is merged).

---

## Summary

| Deliverable | Tasks | Total SP | PR Target | What QA Tests |
|:-----------:|:-----:|:--------:|:---------:|---------------|
| D1 | 7 tasks | 17 | develop | Nav entry, list page, search, add hospital |
| D2 | 4 tasks | 10 | develop | Detail page, edit hospital, providers NPI tab |
| D3 | 6 tasks | 15 | develop | Bulk update tab, CSV upload, paste, validate, confirm |
| **Total** | **17 tasks** | **42** | | |

> The 18th task (MLID-1241 - Design) is already Done and has no code deliverable.
> MLID-1422 (Grooming) is also Done with no code deliverable.
