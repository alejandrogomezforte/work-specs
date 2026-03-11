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
   - [340B-eligibility.md](../analysis/340B-eligibility.md) — triggers, data flow, edge cases
   - [pharmacy-eligibility.md](../analysis/pharmacy-eligibility.md) — triggers, data flow, edge cases

**Phase 2 — Codebase analysis still required:**
Each sub-task should still investigate relevant source files to confirm patterns, imports, and current state before planning.

---

## Purpose

Add computed **340B Eligibility** and **Pharmacy Eligibility** indicators to orders in the Referral Tracker. These calculations help Infusion Guides (IGs) determine whether to process an order through a hospital system's 340B program or through the pharmacy workflow, enabling correct and faster order handling.

This epic builds on the Hospital System Agreements feature (MLID-1021) which established the 340B NPI provider lists.

---

## Domain Summary

### 340B Eligibility (3 outcomes)

| Outcome | Condition |
|---------|-----------|
| **"Waiting on Insurance Input"** | Patient has no primary insurance (`priority: '1'`) |
| **"Hospital System Name(s)"** | Order provider NPI matches hospital system NPI list(s) **AND** primary insurance `payer_type_name` = `"Commercial"` |
| **"No"** | Otherwise |

### Pharmacy Eligibility (3 outcomes)

| Outcome | Condition |
|---------|-----------|
| **"Waiting on Insurance Input"** | Patient has no primary insurance (`priority: '1'`) |
| **"Yes"** | `drug.pharmacyEligible` = true **AND** `location.isPharmacyEligible` = true **AND** insurance condition met (see below) |
| **"No"** | Otherwise |

**Insurance condition for Pharmacy Eligible:**
- `payer_type_name` = `"Medicare"` or `"Medicare Advantage"` → always passes, **OR**
- `payer_type_name` = `"Commercial"` or `"Medicaid"` **AND** `insurancePlan.isPharmacyEligible` = true

### When to Recalculate

| Trigger | Affects |
|---------|---------|
| New order created | Both |
| Hospital system NPI list updated | 340B |
| Patient insurance updated | Both |
| Drug `pharmacyEligible` updated | Pharmacy |
| Location `isPharmacyEligible` updated | Pharmacy |
| Insurance plan `isPharmacyEligible` updated | Pharmacy |

---

## Existing Data Structures (already in codebase)

| Field | Model | Collection | File |
|-------|-------|------------|------|
| `pharmacyEligible?: boolean` | LiDrug | `lidrugs` | `models/LiDrug.ts:54` |
| `isPharmacyEligible?: boolean` | Location | `patientLocations` | `types/locationDTO.ts:22` |
| `isPharmacyEligible: boolean` | InsurancePlan | `insurance_plans` | `models/InsurancePlan.ts:15` |
| `providers[].npi` | HospitalSystem | `hospitalSystems` | `models/HospitalSystem.ts:8` |
| `payer_type_name: string` | PatientInsurance | `patient_insurances` | `models/PatientInsurance.ts:21` |
| `priority: string` | PatientInsurance | `patient_insurances` | `models/PatientInsurance.ts:11` |
| `we_infuse_id_IV: number` | Patient | `patients` | `types/patientDTO.ts:103` |

### Data Flow for Calculation

```
Order (neworders / maintenanceorders)
├── provider (NPI string) ──► lookup HospitalSystem.providers[].npi → 340B match
├── patient (ObjectId string) ──► Patient.we_infuse_id_IV
│   └── PatientInsurance (priority='1') ──► payer_type_name
│       └── InsurancePlan (by insurance_plan_name) ──► isPharmacyEligible
├── drug (ObjectId string) ──► LiDrug.pharmacyEligible
└── site (ObjectId string) ──► Location.isPharmacyEligible
```

### Key Order UI Files

| File | Purpose |
|------|---------|
| `app/orders-tracker/[category]/page.tsx` | Page container with tabs |
| `app/orders-tracker/Orders/Orders.tsx` | Generic DataGrid wrapper |
| `app/orders-tracker/Orders/NewOrders.tsx` | New Orders column definitions |
| `app/orders-tracker/Orders/MaintenanceOrders.tsx` | Maintenance Orders column definitions |
| `app/orders-tracker/components/Provider.tsx` | Provider cell renderer |
| `app/orders-tracker/components/Drug.tsx` | Drug cell renderer |
| `app/orders-tracker/components/Patient.tsx` | Patient cell renderer |
| `services/mongodb/ordersTracker.ts` | Order aggregation & CRUD |
| `services/mongodb/patientInsurance.ts` | Patient insurance queries |
| `types/orders.ts` | Order DTOs and types |

---

## Deliverable 1: 340B Eligibility (18 Story Points)

End-to-end implementation of the 340B Eligibility column — backend calculation, API integration, frontend display, and recalculation triggers specific to 340B.

| Task ID | Jira | Summary | Story Points | Status |
|---------|------|---------|:--:|:------:|
| D1-T1 | [MLID-1816](https://localinfusion.atlassian.net/browse/MLID-1816) | Foundation: types, feature flag, indexes | 2 | Done |
| D1-T2 | [MLID-1817](https://localinfusion.atlassian.net/browse/MLID-1817) | Core 340B calculation service | 3 | Done |
| D1-T3 | [MLID-1818](https://localinfusion.atlassian.net/browse/MLID-1818) | 340B inline triggers + UI column | 5 | Done |
| D1-T4 | [MLID-1819](https://localinfusion.atlassian.net/browse/MLID-1819) | 340B background jobs: hospital system triggers | 5 | Done |
| D1-T5 | [MLID-1820](https://localinfusion.atlassian.net/browse/MLID-1820) | 340B background job: patient insurance webhook | 3 | To Do |

### D1-T1: Foundation — types, feature flag, indexes (2 Story Points)

Shared infrastructure for both eligibility columns.

**Scope**:
- Add `hospital340BEligible?: string` and `pharmacyEligible?: string` to `NewOrderDTO` (`types/orders.ts`)
- Add `ORDER_ELIGIBILITY` to `FeatureFlag` enum (`utils/featureFlags/featureFlags.ts`, default: `false`)
- Create migration script for 4 indexes on `neworders`: `{ provider: 1 }`, `{ patient: 1 }`, `{ drug: 1 }`, `{ site: 1 }`

**Acceptance criteria**:
- Feature flag exists with default `false`
- All 4 indexes created on `neworders`
- Existing code compiles without modification (fields are optional)
- Tests for feature flag default value and migration script

### D1-T2: Core 340B calculation service (3 Story Points)

Implement the 340B eligibility calculation as a testable service.

**Scope**:
- Create `services/orderEligibility/` directory:
  - `calculate340B.ts` — pure function: `calculate340BEligibility({ providerNpi, hospitalSystems, primaryInsurance })` → string
  - `dataFetchers.ts` — `getHospitalSystemsForNpi(npi)`, `getPatientPrimaryInsurance(patientId)` (2-hop: patients → patient_insurances)
  - `orderEligibilityService.ts` — orchestrator: `computeAndSet340BForOrder(order)` (fetch → calculate → return result)

**Acceptance criteria**:
- Pure function returns correct result for all 3 outcomes (hospital names / "Waiting on Insurance Input" / "No")
- Data fetchers handle edge cases (no patient, no insurance, multiple hospital matches)
- 100% test coverage on pure calculation + data fetchers

### D1-T3: 340B inline triggers + UI column (5 Story Points)

Compute 340B eligibility on order create/update and display it in the New Orders DataGrid.

**Scope**:
- **POST** `/api/orders/new`: after `createOrder()`, call `computeAndSet340BForOrder()` and `$set` on order doc(s)
- **PUT** `/api/orders/new`: if `provider` or `patient` changed, recompute `hospital340BEligible`
- Protocol partner order gets same eligibility value
- All gated behind `ORDER_ELIGIBILITY` feature flag
- Add read-only column to `NewOrders.tsx`:
  - `field: 'hospital340BEligible'`, `headerName: '340B Eligible'`
  - Sortable, filterable, not editable
  - Conditionally included based on feature flag

**Key files**: `app/api/orders/new/route.ts`, `services/orderEligibility/orderEligibilityService.ts`, `app/orders-tracker/Orders/NewOrders.tsx`

**Acceptance criteria**:
- New orders get `hospital340BEligible` computed and stored
- Updated orders recalculate when `provider` or `patient` changes
- Column visible when flag on, hidden when flag off
- Skips calculation entirely when flag is disabled

### D1-T4: 340B background jobs — hospital system triggers (5 Story Points)

Create Pulse job infrastructure and implement recalculation triggers for hospital NPI list updates and hospital renames.

**Scope**:
- **Pulse job infrastructure**:
  - Job data types in `services/jobs/types.ts`
  - Job definitions + handlers in `services/jobs/definitions/orderEligibility.ts`
  - Register in `definitions/index.ts`
- **Hospital NPI update** ([analysis: Trigger 3](../analysis/340B-eligibility.md#trigger-3--hospital-system-npi-list-updated)):
  - In `bulkUpdateProviders()`: fetch old NPI list before update, compute diff (`added ∪ removed`)
  - If diff empty → no-op; otherwise schedule `recalculate340BByNpi` job
  - Job queries `neworders WHERE provider IN [affectedNpis]`, recalculates each order
  - Trigger file: `services/orderEligibility/triggers/hospitalNpiUpdate.ts`
- **Hospital rename** ([analysis: Trigger 5](../analysis/340B-eligibility.md#trigger-5--hospital-system-name-renamed)):
  - In `updateHospitalSystem()` API route: detect name change (fetch before update)
  - If name unchanged → no-op; otherwise schedule `recalculate340BRename` job
  - Job queries `neworders WHERE provider IN [all system NPIs]`, recalculates each order
  - Trigger file: `services/orderEligibility/triggers/hospitalRename.ts`

**Key files**: `services/mongodb/hospitalSystem.ts`, `app/api/hospitals/[id]/providers/bulk/route.ts`, `app/api/hospitals/[id]/route.ts`

**Acceptance criteria**:
- NPI diff correctly identifies added/removed NPIs; unchanged NPIs don't trigger recalc
- Same-list re-upload results in zero recalculations
- Hospital rename only triggers when name actually changed
- Jobs process orders in batches with per-order error handling
- Feature flag gated

### D1-T5: 340B background job — patient insurance webhook (3 Story Points)

Recalculate eligibility for affected patients' orders when insurance data changes via Looker webhook.

**Scope**:
- After `webhookUpsertPatientInsurances()` completes, extract unique `we_infuse_patient_id` values from payload
- Resolve to `patients._id` via `we_infuse_id_IV` lookup
- Schedule `recalculateEligibilityByPatient` job (shared job — computes both 340B and pharmacy)
- Job queries `neworders WHERE patient IN [patientIds]`, recalculates eligibility for each order
- Trigger file: `services/orderEligibility/triggers/patientInsuranceChange.ts`
- [Analysis: Trigger 4](../analysis/340B-eligibility.md#trigger-4--patient-insurance-updated-via-webhook)

**Key files**: `services/webhooks/handlers/util/webhookUpsertPatientInsurances.ts` (or its caller)

**Acceptance criteria**:
- Deduplicates patients from webhook payload before scheduling
- Reverse lookup (we_infuse_patient_id → patients._id) works correctly
- Job recalculates 340B for all affected orders (pharmacy is no-op until D2-T4)
- Feature flag gated

**D1 PR to develop:** Not yet

---

## Deliverable 2: Pharmacy Eligibility (19 Story Points)

End-to-end implementation of the Pharmacy Eligible column — backend calculation, API integration, frontend display, and recalculation triggers specific to Pharmacy Eligibility.

| Task ID | Jira | Summary | Story Points | Status |
|---------|------|---------|:--:|:------:|
| D2-T1 | [MLID-1821](https://localinfusion.atlassian.net/browse/MLID-1821) | Core pharmacy calculation service | 3 | To Do |
| D2-T2 | [MLID-1822](https://localinfusion.atlassian.net/browse/MLID-1822) | Pharmacy inline triggers + UI column | 3 | To Do |
| D2-T3 | [MLID-1823](https://localinfusion.atlassian.net/browse/MLID-1823) | Pharmacy background jobs: admin flag triggers | 5 | To Do |
| D2-T4 | [MLID-1824](https://localinfusion.atlassian.net/browse/MLID-1824) | Pharmacy background jobs: webhook triggers | 5 | To Do |
| D2-T5 | [MLID-1825](https://localinfusion.atlassian.net/browse/MLID-1825) | Backfill existing orders | 3 | To Do |

### D2-T1: Core pharmacy calculation service (3 Story Points)

Implement the pharmacy eligibility calculation as a testable service with all required data fetchers.

**Scope**:
- `services/orderEligibility/calculatePharmacy.ts` — pure function:
  - `calculatePharmacyEligibility({ drugPharmacyEligible, locationPharmacyEligible, primaryInsurance, insurancePlanPharmacyEligible })` → string
  - Insurance logic: Medicare/Medicare Advantage → auto-pass; Commercial/Medicaid → need `insurance_plans.isPharmacyEligible === true`
- Add to `services/orderEligibility/dataFetchers.ts`:
  - `getDrugPharmacyEligible(drugId)` — lookup `lidrugs._id`
  - `getLocationPharmacyEligible(locationId)` — lookup `patientLocations._id`
  - `getInsurancePlanPharmacyEligible(planName)` — lookup `insurance_plans.planName` (case-insensitive)

**Acceptance criteria**:
- Returns `"Waiting on Insurance Input"` when no primary insurance
- Returns `"Yes"` only when all 3 checks pass (drug + location + insurance)
- Medicare/Medicare Advantage auto-pass without plan lookup
- Commercial/Medicaid require plan flag check (3-hop)
- Returns `"No"` when any check fails or fields are undefined
- 100% test coverage

### D2-T2: Pharmacy inline triggers + UI column (3 Story Points)

Extend order create/update to compute pharmacy eligibility alongside 340B, and add UI column.

**Scope**:
- Add `computeAndSetPharmacyForOrder()` orchestrator in `orderEligibilityService.ts`
- Modify order API routes to compute BOTH eligibilities (share patient insurance lookup):
  - **POST**: compute both `hospital340BEligible` and `pharmacyEligible`
  - **PUT**: if `drug`, `site`, or `patient` changed → recompute pharmacy
- Add read-only column to `NewOrders.tsx`:
  - `field: 'pharmacyEligible'`, `headerName: 'Pharmacy Eligible'`
  - Positioned after 340B column, feature flag gated

**Acceptance criteria**:
- Both fields computed on order create
- Pharmacy recalculated when `drug`, `site`, or `patient` changes
- Patient insurance fetched once, shared between both calculations
- Column visible/hidden based on feature flag

### D2-T3: Pharmacy background jobs — admin flag triggers (5 Story Points)

Recalculate pharmacy eligibility when drug, location, or insurance plan flags are toggled via admin UI.

**Scope**:
- **Drug flag** ([analysis: Trigger 3](../analysis/pharmacy-eligibility.md#trigger-3--drug-pharmacyeligible-flag-updated)): when `LiDrug.pharmacyEligible` changes in `updateLiDrug()`, find orders by `{ drug: drugId }`, schedule `recalculatePharmacyByDrug` job
- **Location flag** ([analysis: Trigger 4](../analysis/pharmacy-eligibility.md#trigger-4--location-ispharmacyeligible-flag-updated)): when `Location.isPharmacyEligible` changes in `updateLocationInDB()`, find orders by `{ site: locationId }`, schedule `recalculatePharmacyByLocation` job
- **Insurance plan flag** ([analysis: Trigger 5](../analysis/pharmacy-eligibility.md#trigger-5--insurance-plan-ispharmacyeligible-flag-updated-ui)): when `InsurancePlan.isPharmacyEligible` changes in `updateInsurancePlanPharmacyEligible()`, 3-hop reverse lookup (plan → `patient_insurances` → `patients` → `neworders`), schedule `recalculatePharmacyByPlan` job
- Trigger files in `services/orderEligibility/triggers/`

**Key files**: `services/mongodb/liDrugs.ts`, `services/locations/locationDbService.ts`, `services/insurancePlans/insurancePlanService.ts`, respective API routes

**Acceptance criteria**:
- Each trigger only fires when the relevant flag actually changed (not on every update)
- Drug/location triggers use the new `{ drug: 1 }` / `{ site: 1 }` indexes
- Insurance plan trigger correctly resolves 3-hop reverse lookup
- Jobs handle batch processing with per-order error handling
- Feature flag gated

### D2-T4: Pharmacy background jobs — webhook triggers (5 Story Points)

Handle pharmacy recalculation for patient insurance and insurance plan webhooks.

**Scope**:
- **Patient insurance webhook** ([analysis: Trigger 6](../analysis/pharmacy-eligibility.md#trigger-6--patient-insurance-updated-via-webhook)): extend the shared `recalculateEligibilityByPatient` job (from D1-T5) to also compute `pharmacyEligible` for each affected order
- **Insurance plan webhook** ([analysis: Trigger 7](../analysis/pharmacy-eligibility.md#trigger-7--insurance-plan-ispharmacyeligible-updated-via-webhook)):
  - Before `bulkWrite` in `webhookUpsertInsurancePlans()`: snapshot existing `isPharmacyEligible` values
  - After `bulkWrite`: compare old vs new, identify plans whose flag changed
  - For each changed plan → same 3-hop reverse lookup as D2-T3 (plan → patients → orders)
  - Schedule `recalculatePharmacyByPlan` job per changed plan
  - Trigger file: `services/orderEligibility/triggers/insurancePlanWebhook.ts`

**Key files**: `services/jobs/definitions/orderEligibility.ts`, `services/webhooks/handlers/util/webhookUpsertInsurancePlans.ts`

**Acceptance criteria**:
- Patient insurance webhook now recalculates both 340B and pharmacy
- Insurance plan diff correctly identifies changed `isPharmacyEligible` values
- No-op when no plans changed (common case for webhook syncs)
- 3-hop reverse lookup works for bulk plan changes
- Feature flag gated

### D2-T5: Backfill existing orders (3 Story Points)

One-time job to compute and store both eligibility fields for all existing `neworders`.

**Scope**:
- Create `services/jobs/definitions/orderEligibilityBackfill.ts`:
  - Pulse job that processes all `neworders` in batches (100-500 per batch)
  - Uses `bulkWrite` for efficient updates
  - Progress logging every N orders
  - Per-order error handling (skip and continue)
  - Only fills missing values (idempotent, re-runnable)
- Optional: admin API endpoint `POST /api/admin/order-eligibility/backfill` to trigger

**Acceptance criteria**:
- All existing orders get both eligibility fields computed
- Batch processing with configurable batch size
- Safe to re-run (doesn't overwrite existing values unless force mode)
- Logs progress and error summary
- Feature flag gated

**D2 PR to develop:** Not yet

---

## Architecture Decisions

1. **Stored fields** on `neworders` documents (not computed at read time):
   - `hospital340BEligible: string` — hospital name(s), `"Waiting on Insurance Input"`, or `"No"`
   - `pharmacyEligible: string` — `"Yes"`, `"Waiting on Insurance Input"`, or `"No"`

2. **New service directory** `services/orderEligibility/`:
   - `calculate340B.ts` — pure calculation function (no DB access, fully unit-testable)
   - `calculatePharmacy.ts` — pure calculation function
   - `dataFetchers.ts` — all MongoDB lookups (hospital systems, patient insurance, drug, location, plan)
   - `orderEligibilityService.ts` — orchestrator: fetch data → calculate → return result
   - `triggers/` — one file per trigger source (hospitalNpiUpdate, hospitalRename, drugFlagUpdate, locationFlagUpdate, planFlagUpdate, patientInsuranceChange, insurancePlanWebhook)

3. **Inline vs background job**:
   - **Inline** (in the API request): order create/update — only 1-2 orders affected, trivial cost
   - **Background Pulse job**: all fan-out triggers — schedule via `scheduleImmediateJob()`

4. **Shared patient insurance webhook**: one job (`recalculateEligibilityByPatient`) computes BOTH 340B and pharmacy for affected orders, avoiding double-processing of the same webhook

5. **Diff strategies** to minimize unnecessary recalculations:
   - Hospital NPI bulk update: compute `added ∪ removed` NPIs, only recalculate orders tied to changed NPIs
   - Insurance plan webhook: snapshot `isPharmacyEligible` before bulkWrite, only recalculate for plans whose flag changed

6. **Feature flag**: `ORDER_ELIGIBILITY` gates all calculation logic, background jobs, and UI columns

7. **4 indexes on `neworders`** (created in one migration):
   - `{ provider: 1 }` — hospital NPI triggers
   - `{ patient: 1 }` — patient insurance triggers
   - `{ drug: 1 }` — drug flag trigger
   - `{ site: 1 }` — location flag trigger

8. **Scope**: `neworders` collection only (not `maintenanceorders`)

9. **Linting**: Always use `npx eslint --fix <files>` targeting only changed files. Never use `npm run lint` (`next lint --fix`) — it auto-fixes the entire app and modifies unrelated files. See [sw-workflow.md Step 8](../dx/sw-workflow.md#step-8--verify-quality).

---

## Epic Reference

- **Jira**: [MLID-1492](https://localinfusion.atlassian.net/browse/MLID-1492)
- **Total Story Points**: 37 (D1: 18 — 5 tasks, D2: 19 — 5 tasks)
- **Epic Branch**: `epic/MLID-1492-order-eligibility`
- **Deliverables**: 2
- **Related Epic**: [MLID-1021](https://localinfusion.atlassian.net/browse/MLID-1021) (Hospital System Agreements)

---

## Non-Code Tasks

| Task ID | Summary | Status |
|---------|---------|:------:|
| - | ~~Create Jira sub-tasks for all deliverable items~~ | Done |
| - | Update MEMORY.md with new epic context | To Do |
| - | ~~Revise architecture decisions for per-column deliverable approach~~ | Done |
