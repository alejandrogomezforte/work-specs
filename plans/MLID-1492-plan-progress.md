# MLID-1492 — Ref Tracker: Add 340B & Pharmacy Eligible on "New Orders"

## Epic Workflow

This template follows the [SW Workflow](../dx/sw-workflow.md). For each sub-task, run the workflow cycle once and update this document to track progress. Read this document first to understand overall context before starting each cycle.

> **Important:** `docs/agomez/` is gitignored. Never stage, commit, or version plan files, analysis docs, or workflow docs. They are local working documents only.

### Workflow References

| Document | Purpose |
|----------|---------|
| [sw-workflow.md](../dx/sw-workflow.md) | Master workflow: the full Phase 1–5 cycle (context → analysis → plan → implement → deliver) |
| [epic-workflow.md](../dx/epic-workflow.md) | Epic plan template: the template this document was created from |
| [individual-task-workflow.md](../dx/individual-task-workflow.md) | Sub-task plan template: use for each sub-task's implementation plan if needed |
| [epic-git-branch-strategy.md](../dx/epic-git-branch-strategy.md) | Git branch strategy: epic branch, sub-task branches, merge/PR flow |

### Workflow Overrides for This Epic

**Phase 1 — Skip Jira ticket reading (Step 1):**
All Jira context has been gathered and consolidated into this document. Do NOT read individual Jira tickets for sub-tasks. Instead, use:
1. **This document** as the primary context source (scope, acceptance criteria, key files per task)
2. **Analysis documents** for detailed technical reference:
   - [340B-eligibility.md](../analysis/340B-eligibility.md) — query-time pipeline stages, logic, edge cases
   - [pharmacy-eligibility.md](../analysis/pharmacy-eligibility.md) — query-time pipeline stages, logic, edge cases
   - [how-new-orders-query-works.md](../analysis/how-new-orders-query-works.md) — full pipeline breakdown

**Phase 2 — Codebase analysis still required:**
Each sub-task should still investigate relevant source files to confirm patterns, imports, and current state before planning.

---

## Purpose

Add computed **340B Eligibility** and **Pharmacy Eligibility** display-only columns to the New Orders DataGrid in the Referral Tracker. These calculations help Infusion Guides (IGs) determine whether to process an order through a hospital system's 340B program or through the pharmacy workflow.

**Approach: Query-time computation.** Instead of storing eligibility as fields on each order and maintaining them through triggers/background jobs, the values are computed dynamically via `$lookup` + `$addFields` stages appended to the existing aggregation pipeline. This eliminates all triggers, background jobs, stored fields, and new indexes on `neworders`.

---

## Scope

- **New orders**: 340B + Pharmacy columns (D1, D2)
- **Maintenance orders**: 340B column only (D3)
- **Display-only columns** — no filtering or sorting (future enhancement)
- **Feature flag gated**: `ORDER_ELIGIBILITY` controls all eligibility columns (server + client)

---

## Domain Summary

### 340B Eligibility (3 outcomes)

| Outcome | Condition |
|---------|-----------|
| **"Waiting on Insurance Input"** | Provider NPI matches a hospital system, but patient has no primary active insurance |
| **"Hospital System Name(s)"** | Provider NPI matches hospital system(s) **AND** primary insurance = `"Commercial"` |
| **"No"** | Provider NPI not in any hospital system, **OR** insurance is not `"Commercial"` |

### Pharmacy Eligibility (3 outcomes)

| Outcome | Condition |
|---------|-----------|
| **"Waiting on Insurance Input"** | Patient has no primary active insurance |
| **"Yes"** | Drug is NOT a treatment **AND** `drug.pharmacyEligible` is **not** explicitly `false` **AND** `site.isPharmacyEligible` = true **AND** insurance condition met |
| **"No"** | Otherwise (including when drug `is_treatment == true`) |

**Insurance condition for Pharmacy Eligible:**
- `payer_type_name` = `"Medicare"` or `"Medicare Advantage"` → always passes, **OR**
- `payer_type_name` = `"Commercial"` or `"Medicaid"` **AND** `insurancePlan.isPharmacyEligible` = true

---

## Architecture

### Pipeline placement

Eligibility stages append to `expandIdsAggregate` inside `$facet.data`, **after `$skip/$limit`** and after all existing lookups. Only 25–100 orders per page get processed.

```
$facet.data:
  sort → skip → limit
  ...expandIdsAggregate (existing: users, site, patient, drug, provider, providerOffice)
  ┌──── NEW (when category='new'|'maintenance' AND includeEligibility=true) ─┐
  │ $addFields: _providerNpi (extract NPI from provider doc or string)       │
  │ $lookup: hospitalSystems (match NPI → 340B)                              │
  │ $lookup: patient_insurances (primary active insurance — shared)           │
  │ $addFields: hospital340BEligible ($switch computation)                   │
  │ $lookup: insurance_plans (plan pharmacy flag — for pharmacy)             │
  │ $addFields: pharmacyEligible ($switch computation)                       │
  │ $unset: [_providerNpi, _340b_hospitals, _primaryInsurance, _insurancePlan]│
  └──────────────────────────────────────────────────────────────────────────┘
```

### Feature flag gating at two levels

- **Server**: `FeatureFlagService.getFeatureFlag(ORDER_ELIGIBILITY)` in API route → passed as `includeEligibility` to `getOrders()` → `getOrdersAggregate()`
- **Client**: `useFeatureFlag(ORDER_ELIGIBILITY)` in `page.tsx` → passed as `showEligibilityColumns` to `getNewOrdersColumns()`

### Key design decisions

1. **No stored fields** on `neworders` — zero staleness, zero triggers
2. **No new indexes** on `neworders` — lookups happen on related collections
3. **No background jobs** — computation is per-page, per-request
4. **Shared `_primaryInsurance` lookup** — used by both 340B and pharmacy
5. **Conditional pipeline stages** — only added when `category === 'new' || 'maintenance'` AND flag is on
6. **Scope**: New orders (340B + pharmacy) and maintenance orders (340B only)
7. **Linting**: Always use `npx eslint --fix <files>` targeting only changed files (see [sw-workflow.md Step 8](../dx/sw-workflow.md#step-8--verify-quality))
8. **Formatting**: Always use `npx prettier --write <files>` targeting only changed files. **Never run `npm run format`** — it reformats the entire repo and pollutes the git diff with unrelated changes.

---

## Deliverable 1: 340B Eligible Column (8 Story Points)

Shared infrastructure (feature flag, types, API gating) + 340B pipeline stages + UI column.

| Task ID | Jira | Summary | SP | Status | Branch | Merged to Epic | Notes |
|---------|------|---------|----|--------|--------|----------------|-------|
| D1-T1 | MLID-1857 | Feature flag + types | 1 | Done | feature/MLID-1857-feature-flag-types | Yes | Commit 5462c4f3 |
| D1-T2 | MLID-1858 | 340B pipeline stages (includes shared insurance lookup) | 3 | Done | feature/MLID-1858-340b-pipeline-stages | Yes | Commit e5c7dd3a |
| D1-T3 | MLID-1859 | UI column + feature flag wiring | 2 | Done | feature/MLID-1859-340b-ui-column | Yes | Commit f750849b |
| D1-T4 | MLID-1860 | API route feature flag gating | 2 | Done | feature/MLID-1860-api-route-feature-flag-gating | Yes | Commit 2e884630 |

**D1 Status: Delivered to QA**

### D1-T1: Feature flag + types (1 SP)

**Scope:**
- Add `ORDER_ELIGIBILITY` to `FeatureFlag` enum + `DEFAULT_FEATURE_FLAGS` (default: `false`)
- Add `hospital340BEligible?: string` and `pharmacyEligible?: string` to the `Extra` type in `FullOrder<K>`, gated to `K extends 'new'`

**Key files:**
- `apps/web/utils/featureFlags/featureFlags.ts`
- `apps/web/types/orders.ts` (lines 118–127, Extra type)

**Type change:**
```typescript
// In the Extra type of FullOrder<K>:
hospital340BEligible?: K extends 'new' ? string : never;
pharmacyEligible?: K extends 'new' ? string : never;
```

**Acceptance criteria:**
- `FeatureFlag.ORDER_ELIGIBILITY` exists with default `false`
- `FullOrder<'new'>` has both optional string fields
- `FullOrder<'maintenance'>` and `FullOrder<'lead'>` do not have them (typed as `never`)
- Existing code compiles without modification
- Tests: feature flag default, type compilation

### D1-T2: 340B pipeline stages (3 SP)

**Scope:**
- Add `includeEligibility?: boolean` param to `getOrdersAggregate()` and `GetOrdersParams`
- Create `getEligibilityPipelineStages(): Document[]` returning the full stage sequence
- In `expandIdsAggregate` (after line 654), append stages when `category === 'new' && includeEligibility`
- Pass `includeEligibility` through from `getOrders()`

**Key files:**
- `apps/web/services/mongodb/ordersTracker.ts` (getOrdersAggregate, getOrders)
- `apps/web/services/mongodb/__tests__/ordersTracker.test.ts`

**Pipeline stages (340B + shared):**
1. `$addFields: { _providerNpi }` — `$ifNull: ['$provider.npi', { $toString: '$provider' }]`
2. `$lookup: hospitalSystems` — match NPI in `providers[].npi` where `deletedAt == null`
3. `$lookup: patient_insurances` — match `we_infuse_patient_id` = `patient.we_infuse_id_IV`, `priority='1'`, `status='Active'`
4. `$addFields: { hospital340BEligible }` — `$switch` with branches per business logic

**Call sites for `getOrdersAggregate()` that need `includeEligibility` param (all default to `false`):**
- `getOrderById` (line ~795) — single order detail view
- `getOrders` (line ~904) — **only one that passes `true`**
- Any other internal callers

**Acceptance criteria:**
- Pipeline includes eligibility stages when `category='new'` + flag enabled
- Pipeline excludes stages when `category='maintenance'` even if flag enabled
- Pipeline excludes stages when flag disabled
- Tests verify pipeline shape using `mockAggregate.mock.calls[0][0]`

### D1-T3: UI column + feature flag wiring (2 SP)

**Scope:**
- In `page.tsx`: call `useFeatureFlag(ORDER_ELIGIBILITY)`, pass to `getNewOrdersColumns({ showEligibilityColumns })`
- In `NewOrders.tsx`: add `showEligibilityColumns` param, add 340B column with spread-ternary
- Column: `field: 'hospital340BEligible'`, `headerName: '340B Eligible'`, `width: 200`, `filterable: false`, `sortable: false`

**Key files:**
- `apps/web/app/orders-tracker/[category]/page.tsx`
- `apps/web/app/orders-tracker/Orders/NewOrders.tsx`
- `apps/web/app/orders-tracker/Orders/NewOrders.test.tsx`

**Acceptance criteria:**
- Column visible when `showEligibilityColumns: true`
- Column absent when `showEligibilityColumns: false` (default)
- `valueGetter` returns `row.hospital340BEligible ?? ''`
- Tests for column presence/absence and valueGetter

### D1-T4: API route feature flag gating (2 SP)

**Scope:**
- In `GET /api/orders/new`: check `FeatureFlagService.getFeatureFlag(ORDER_ELIGIBILITY)`, pass `includeEligibility` to `getOrders()`

**Key files:**
- `apps/web/app/api/orders/new/route.ts`
- `apps/web/app/api/orders/new/route.test.ts`

**Acceptance criteria:**
- When flag is on: `getOrders()` called with `includeEligibility: true`
- When flag is off: `getOrders()` called with `includeEligibility: false`
- Tests mock `FeatureFlagService.getFeatureFlag` and verify pass-through

---

## Deliverable 2: Pharmacy Eligible Column (5 Story Points)

Pharmacy pipeline stages + UI column. Builds on D1's shared infrastructure.

| Task ID | Jira | Summary | SP | Status | Branch | Merged to Epic | Notes |
|---------|------|---------|----|--------|--------|----------------|-------|
| D2-T1 | MLID-1861 | Pharmacy pipeline stages | 3 | Done | feature/MLID-1861-pharmacy-pipeline-stages | Yes | Commit cc5b7f1a |
| D2-T2 | MLID-1862 | Pharmacy UI column | 2 | Done | feature/MLID-1862-pharmacy-ui-column | Yes | Commit 3a6c9e5d |

**D2 Status: In QA**

**Related Bug Fixes & Enhancements:**

| Task ID | Jira | Summary | SP | Status | Branch | Merged to Epic | Notes |
|---------|------|---------|----|--------|--------|----------------|-------|
| BF-1 | MLID-1955 | Location isPharmacyEligible default fix | 2 | In Development | fix/MLID-1955-location-pharmacy-eligible-default | No | |
| EN-1 | MLID-2015 | Pharmacy eligibility: exclude treatments (is_treatment) | 2 | In QA | feature/MLID-2015-pharmacy-exclude-treatments | No | PR pushed, waiting for QA |

### D2-T1: Pharmacy pipeline stages (3 SP)

**Scope:**
- Extend `getEligibilityPipelineStages()` with:
  1. `$lookup: insurance_plans` — match `planName` from `_primaryInsurance.insurance_plan_name`
  2. `$addFields: { pharmacyEligible }` — `$switch` computation
  3. Update `$unset` to include `_insurancePlan`

**Key files:**
- `apps/web/services/mongodb/ordersTracker.ts`
- `apps/web/services/mongodb/__tests__/ordersTracker.test.ts`

**Pharmacy logic (uses existing resolved fields):**
- `$drug.pharmacyEligible` — already resolved by expandIdsAggregate. **Updated 3/4/2026:** only `false` = ineligible; `true`/`null`/`undefined` = eligible
- `$site.isPharmacyEligible` — already resolved by expandIdsAggregate
- `$_primaryInsurance` — already looked up by 340B stages (D1-T2)
- `$_insurancePlan.isPharmacyEligible` — NEW lookup

**Acceptance criteria:**
- Pipeline includes `$lookup` to `insurance_plans`
- `pharmacyEligible` computed correctly for all 3 outcomes
- Medicare/Medicare Advantage auto-pass
- Commercial/Medicaid check plan flag
- Tests verify pipeline stages and computed values

### D2-T2: Pharmacy UI column (2 SP)

**Scope:**
- Add pharmacy column in same spread-ternary block as 340B in `NewOrders.tsx`
- Column: `field: 'pharmacyEligible'`, `headerName: 'Pharmacy Eligible'`, `width: 170`, `filterable: false`, `sortable: false`

**Key files:**
- `apps/web/app/orders-tracker/Orders/NewOrders.tsx`
- `apps/web/app/orders-tracker/Orders/NewOrders.test.tsx`

**Acceptance criteria:**
- Column visible when `showEligibilityColumns: true`
- Positioned after 340B column
- `valueGetter` returns `row.pharmacyEligible ?? ''`
- Tests for column presence, ordering, and valueGetter

---

## Enhancement: MLID-2015 — Exclude Treatments from Pharmacy Eligibility (2 SP)

### Scope

Orders whose drug is a treatment (`lidrugs.is_treatment == true`) should never be pharmacy eligible. Add a new `$switch` branch to the `pharmacyEligible` computation in `getEligibilityPipelineStages()` that short-circuits to `"No"` when the drug is a treatment. No new lookups, no type changes, no UI changes.

### Key Files

| File | Action | Description |
|------|--------|-------------|
| `apps/web/services/mongodb/ordersTracker.ts` | Modify | Add `is_treatment` branch to pharmacy `$switch` |
| `apps/web/services/mongodb/__tests__/ordersTracker.test.ts` | Modify | Add test cases for treatment short-circuit |

### Implementation Details

**Insertion point:** Line 447 of `ordersTracker.ts`, between the "No primary insurance" branch (line 440-446) and the "All conditions met → Yes" branch (line 447-513).

**New `$switch` branch to insert:**

```javascript
// Drug is a treatment → "No" (treatments are never pharmacy eligible)
{
  case: {
    $eq: [{ $ifNull: ['$drug.is_treatment', false] }, true],
  },
  then: 'No',
},
```

**After insertion, the `$switch` branches become:**

1. No primary insurance → `"Waiting on Insurance Input"`
2. Drug is a treatment → `"No"` **(NEW)**
3. All conditions met (drug eligible + location eligible + insurance type) → `"Yes"`
4. Default → `"No"`

**Why `$ifNull` with `false`:** The `drug` field may remain a raw string (unresolved ObjectId) when `drugObjectId` is null, so `$drug.is_treatment` would be missing. Defaulting to `false` means unresolved drugs are not treated as treatments, which preserves existing behavior.

### TDD Plan

#### RED: Write Failing Tests

Add two test cases to the existing `describe('Pharmacy Eligibility')` block in `ordersTracker.test.ts`:

**Test 1: `should include is_treatment check as second branch in pharmacyEligible $switch`**
- Arrange: standard mock setup with `includeEligibility: true`, `category: 'new'`
- Act: call `getOrders()`
- Assert: find the `$addFields` stage with `pharmacyEligible`, inspect `$switch.branches[1]`
  - `branches[1].case` should contain `$eq` with `$ifNull: ['$drug.is_treatment', false]` and `true`
  - `branches[1].then` should be `'No'`

**Test 2: `should have is_treatment branch before the Yes computation branch`**
- Arrange: same as above
- Act: call `getOrders()`
- Assert: verify ordering — `branches[0].then` is `'Waiting on Insurance Input'`, `branches[1].then` is `'No'`, `branches[2].then` is `'Yes'`

#### GREEN: Add the Branch

Insert the `$switch` branch in `getEligibilityPipelineStages()` at the exact position described above. Minimal change — one new object in the `branches` array.

#### REFACTOR

No refactoring expected — the change is a single array element insertion. Verify lint and type-check pass.

### Acceptance Criteria

- Orders with a treatment drug (`is_treatment == true`) always show `"No"` for pharmacy eligibility, regardless of location, insurance, or drug `pharmacyEligible` flag
- Orders with a non-treatment drug (`is_treatment == false` or `undefined`) behave exactly as before
- Unit tests cover the treatment short-circuit scenario
- The `is_treatment` branch is the second branch (index 1) in the `$switch`, after the insurance check and before the Yes computation

### Security Considerations

None. This is a read-only computation in an existing aggregation pipeline behind the `ORDER_ELIGIBILITY` feature flag. No new endpoints, no new inputs, no PHI implications.

### Open Questions

None.

---

## Task List

### Deliverable 1: 340B Eligible Column (8 SP)

- MLID-1857 — Feature flag + types (1 SP)
- MLID-1858 — 340B pipeline stages (3 SP)
- MLID-1859 — UI column + feature flag wiring (2 SP)
- MLID-1860 — API route feature flag gating (2 SP)

### Deliverable 2: Pharmacy Eligible Column (5 SP)

- MLID-1861 — Pharmacy pipeline stages (3 SP)
- MLID-1862 — Pharmacy UI column (2 SP)

### Deliverable 3: Maintenance Table Extensions (3 SP)

- MLID-1975 — Add 340B column to Maintenance table (3 SP) — [Plan](./MLID-1975-340b-maintenance-column.md)

### Bug Fixes & Enhancements (4 SP)

- MLID-1955 — Locations missing isPharmacyEligible field show incorrect eligibility (2 SP) — [Plan](./MLID-1955-location-pharmacy-eligible-default.md)
- MLID-2015 — Pharmacy eligibility: exclude treatments (is_treatment) (2 SP) — [Plan](./MLID-2015-pharmacy-exclude-treatments.md)

---

## Implementation Order

```
D1-T1 (feature flag + types)
  ↓
D1-T2 (340B pipeline) ←→ D1-T4 (API route gating)  [can be parallel]
  ↓
D1-T3 (340B UI column)
  ↓
D2-T1 (pharmacy pipeline)
  ↓
D2-T2 (pharmacy UI column)
  ↓
D3 — MLID-1975 (340B on maintenance table)
```

---

## Deliverable 3: 340B Column on Maintenance Table (3 SP)

Extends the 340B Eligible column to the Maintenance Orders table. Reuses the existing `ORDER_ELIGIBILITY` feature flag and `getEligibilityPipelineStages()` pipeline. Pharmacy column is NOT included for maintenance in this task.

| Task ID | Jira | Summary | SP | Status | Branch | Merged to Epic | Notes |
|---------|------|---------|----|--------|--------|----------------|-------|
| D3 | MLID-1975 | Add 340B column to Maintenance table | 3 | To Do | feature/MLID-1975-340b-maintenance-column | No | |

### D3: 340B on Maintenance (3 SP)

**Scope:**
- Widen pipeline category guard from `'new'` to `'new' || 'maintenance'`
- Add `FeatureFlagService.getFeatureFlag(ORDER_ELIGIBILITY)` to maintenance API route
- Widen `hospital340BEligible` type to include `'maintenance'`
- Add `showEligibilityColumns` param to `getMaintenanceOrdersColumns()` + 340B column
- Wire `isOrderEligibilityEnabled` through `page.tsx`

**Key files:**
- `apps/web/services/mongodb/ordersTracker.ts` (pipeline guard)
- `apps/web/app/api/orders/maintenance/route.ts` (feature flag)
- `apps/web/types/orders.ts` (type widening)
- `apps/web/app/orders-tracker/Orders/MaintenanceOrders.tsx` (UI column)
- `apps/web/app/orders-tracker/[category]/page.tsx` (wiring)

**Approach:** Option B --- compute full eligibility pipeline (340B + pharmacy) for maintenance, but only display 340B in the UI. Avoids parameterizing the shared pipeline function. When pharmacy is later added to maintenance, it is purely a UI change.

**Acceptance criteria:**
- 340B column visible on maintenance table when `ORDER_ELIGIBILITY` flag is enabled
- 340B column absent when flag is disabled
- Pharmacy column is NOT shown on maintenance (explicit negative test)
- Pipeline includes eligibility stages for maintenance when flag is enabled
- API route checks feature flag and passes `includeEligibility` to `getOrders()`
- Dual enforcement: server (API route) + client (page.tsx)

---

## Epic Reference

- **Jira**: [MLID-1492](https://localinfusion.atlassian.net/browse/MLID-1492)
- **Total Story Points**: 20 (D1: 8, D2: 5, D3: 3, Bug Fixes & Enhancements: 4)
- **Epic Branch**: `epic/MLID-1492-order-eligibility`
- **Deliverables**: 3

---

## Non-Code Tasks

| Task ID | Summary | Status |
|---------|---------|:------:|
| - | Create Jira sub-tasks for all deliverable items | Done |
| - | Update MEMORY.md with new epic context | Done |
