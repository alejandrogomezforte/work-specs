# MLID-2194 — Data Testing: Pharmacy "Medicare Supplement"

Working notebook for verifying that orders whose primary insurance is `payer_type_name = "Medicare Supplement"` are now flagged `pharmacyEligible: "Yes"` (when the other gates pass) instead of `"No"`.

- **Branch:** `feature/MLID-2194-pharmacy-medicare-supplement`
- **Plan:** `docs/agomez/plans/MLID-2194-pharmacy-medicare-supplement.md`
- **Regex change:** `^\s*medicare(\s+advantage)?\s*$` → `^\s*medicare(\s+(advantage|supplement))?\s*$`
- **Pipeline file:** `apps/web/services/mongodb/ordersTracker.ts:174`

---

## Step 1 — Find patient insurances with "Medicare Supplement"

Run these in **Mongo Compass** against the `patient_insurances` collection.

### 1a. Any record with `payer_type_name = "Medicare Supplement"` (exact, case-sensitive)

```js
{ "payer_type_name": "Medicare Supplement" }
```

### 1b. Same but case-insensitive + tolerates leading/trailing whitespace

This mirrors what the new regex actually accepts.

```js
{ "payer_type_name": { "$regex": "^\\s*medicare\\s+supplement\\s*$", "$options": "i" } }
```

### 1c. Primary + active/undetermined only

These are the only insurances the pipeline considers (see `ordersTracker.ts:371–372`). If 1a/1b find rows but this returns nothing, no order will hit the new branch.

```js
{
  "payer_type_name": "Medicare Supplement",
  "priority": "1",
  "status": { "$in": ["Active", "Undetermined"] }
}
```

### 1d. Sanity check — variants we should ALSO see (already supported pre-change)

Use to confirm we're querying the right collection / field:

```js
{ "payer_type_name": { "$in": ["Medicare", "Medicare Advantage"] } }
```

---

## Findings

| Query | Count | Notes |
|-------|-------|-------|
| 1a (exact "Medicare Supplement", any status) | **2992** | Field exists, lots of historical/secondary records |
| 1b (regex, whitespace-tolerant) | _not run_ | |
| Active + Medicare Supplement (any priority) | **2550** | Mostly secondary (`priority: '2'`) |
| **Active + Medicare Supplement + priority `'1'`** | **96** | Pipeline-eligible pool — sampled below |
| 1d (Medicare / Medicare Advantage baseline) | _not run_ | |

### Sample documents (5 records, status `"Active"`, priority `"1"`, payer_type `"Medicare Supplement"`)

Pulled via MCP `find` against `local-infusion-db.patient_insurances` with `{ "payer_type_name": "Medicare Supplement", "status": "Active", "priority": "1" }`, limit 5.

| # | `_id` | `we_infuse_patient_id` | `insurance_plan_name` | `insurance_company_name` | `priority` | `status` | Pipeline-eligible? |
|---|-------|------------------------|-----------------------|--------------------------|------------|----------|--------------------|
| 1 | `694304dcf0484dead5cae789` | `1007696` | Tricare East \| Select | Tricare | `1` | `Active` | ✅ |
| 2 | `694304e5f0484dead5caecbd` | `911448`  | Carefirst BCBS Medigap | CareFirst | `1` | `Active` | ✅ |
| 3 | `694304e5f0484dead5caecf9` | `768371`  | The Empire Plan | _null_ | `1` | `Active` | ✅ |
| 4 | `694304e8f0484dead5caeeb2` | `927508`  | Tricare East \| Select | Tricare | `1` | `Active` | ✅ |
| 5 | `694304ebf0484dead5caf099` | `1048912` | Tricare East \| Select | Tricare | `1` | `Active` | ✅ |

> "Pipeline-eligible" = matches the `$lookup` predicate at `ordersTracker.ts:371–372` — `priority: '1'` AND `status ∈ ['Active', 'Undetermined']`. All 5 rows above qualify and will be evaluated by the `pharmacyEligible` `$switch`.

### Raw documents

<details>
<summary>Click to expand</summary>

```json
[
  {
    "_id": "694304dcf0484dead5cae789",
    "we_infuse_insurance_id": 1535475,
    "we_infuse_patient_id": 1007696,
    "insurance_company_id": 16,
    "insurance_company_name": "Tricare",
    "insurance_plan_id": 75406,
    "insurance_plan_name": "Tricare East | Select",
    "payer_type_name": "Medicare Supplement",
    "priority": "1",
    "status": "Active",
    "member_id": "01302855903",
    "effective_date": "2026-01-01T00:00:00Z",
    "term_date": null,
    "updated_date": "2026-02-09T00:00:00Z",
    "createdAt": "2025-12-17T19:30:36.896Z",
    "updatedAt": "2026-04-23T16:43:14.130Z",
    "lastSyncedAt": "2026-04-23T16:43:14.130Z"
  },
  {
    "_id": "694304e5f0484dead5caecbd",
    "we_infuse_insurance_id": 1335231,
    "we_infuse_patient_id": 911448,
    "insurance_company_id": 59,
    "insurance_company_name": "CareFirst",
    "insurance_plan_id": 81433,
    "insurance_plan_name": "Carefirst BCBS Medigap",
    "payer_type_name": "Medicare Supplement",
    "priority": "1",
    "status": "Active",
    "member_id": "XIM900963358",
    "group_id": "99K1",
    "effective_date": "2026-01-01T00:00:00Z",
    "term_date": null,
    "updated_date": "2026-01-21T00:00:00Z",
    "createdAt": "2025-12-17T19:30:45.354Z",
    "updatedAt": "2026-04-23T16:43:14.130Z",
    "lastSyncedAt": "2026-04-23T16:43:14.130Z"
  },
  {
    "_id": "694304e5f0484dead5caecf9",
    "we_infuse_insurance_id": 1058669,
    "we_infuse_patient_id": 768371,
    "insurance_company_id": null,
    "insurance_company_name": null,
    "insurance_plan_id": 71315,
    "insurance_plan_name": "The Empire Plan",
    "payer_type_name": "Medicare Supplement",
    "priority": "1",
    "status": "Active",
    "member_id": "890850165",
    "group_id": "030500",
    "effective_date": "2025-01-01T00:00:00Z",
    "term_date": null,
    "updated_date": "2025-12-04T00:00:00Z",
    "createdAt": "2025-12-17T19:30:45.691Z",
    "updatedAt": "2026-04-23T16:43:14.130Z",
    "lastSyncedAt": "2026-04-23T16:43:14.130Z"
  },
  {
    "_id": "694304e8f0484dead5caeeb2",
    "we_infuse_insurance_id": 1389635,
    "we_infuse_patient_id": 927508,
    "insurance_company_id": 16,
    "insurance_company_name": "Tricare",
    "insurance_plan_id": 75406,
    "insurance_plan_name": "Tricare East | Select",
    "payer_type_name": "Medicare Supplement",
    "priority": "1",
    "status": "Active",
    "member_id": "01207642801",
    "effective_date": "2024-02-13T00:00:00Z",
    "term_date": null,
    "updated_date": "2026-01-29T00:00:00Z",
    "createdAt": "2025-12-17T19:30:48.241Z",
    "updatedAt": "2026-04-23T16:43:14.130Z",
    "lastSyncedAt": "2026-04-23T16:43:14.130Z"
  },
  {
    "_id": "694304ebf0484dead5caf099",
    "we_infuse_insurance_id": 1618730,
    "we_infuse_patient_id": 1048912,
    "insurance_company_id": 16,
    "insurance_company_name": "Tricare",
    "insurance_plan_id": 75406,
    "insurance_plan_name": "Tricare East | Select",
    "payer_type_name": "Medicare Supplement",
    "priority": "1",
    "status": "Active",
    "member_id": "016551417",
    "effective_date": "2021-10-06T00:00:00Z",
    "term_date": null,
    "updated_date": "2026-03-03T00:00:00Z",
    "createdAt": "2025-12-17T19:30:51.396Z",
    "updatedAt": "2026-04-23T16:43:14.130Z",
    "lastSyncedAt": "2026-04-23T16:43:14.130Z"
  }
]
```

</details>

### Observation

After narrowing to `priority: "1"` + `status: "Active"`, we have **96 pipeline-eligible Medicare Supplement records** in the database — a small but workable pool.

Notable patterns in this sample:
- 3 of the 5 records use `insurance_plan_name = "Tricare East | Select"` (insurance_plan_id `75406`). Tricare classified as a Medicare Supplement payer is data-quality-noteworthy but unrelated to our regex change — the eligibility check is purely on `payer_type_name`.
- The other two use distinct plans: Carefirst BCBS Medigap and The Empire Plan.
- `insurance_company_name` is `null` on row 3 (Empire Plan), so any joinwith `insurance_companies` would miss it. Not relevant for the pharmacy regex path.

---

---

## Step 2 — Order ingredients (drug, patient, site)

To create an order that produces `pharmacyEligible: "Yes"`, we need to combine:
- a **patient** whose primary insurance hits the new Medicare Supplement regex (Step 1 above),
- a **drug** that isn't a treatment and isn't explicitly ineligible,
- a **site/location** that is pharmacy-eligible.

### 2a. Drugs — `lidrugs`

```js
// Mongo Compass — collection: lidrugs
{ "is_treatment": { "$ne": true }, "pharmacyEligible": { "$ne": false } }
```

> Total in DB: **41**. Predicate uses `$ne` so missing/`null` values pass — only literal `is_treatment: true` or `pharmacyEligible: false` are rejected. This matches the pipeline gates at `ordersTracker.ts:140` and `ordersTracker.ts:150–153`.

| # | `_id` | `name` | `is_treatment` | `pharmacyEligible` | Notes |
|---|-------|--------|----------------|--------------------|-------|
| 1 | `6938863d9879dc15c6a1de64` | Skyrizi | `false` | `true` | Tier 1 - audit |
| 2 | `693896d59879dc15c6a1de68` | Asceniv | `false` | `true` | Tier 1 - audit, requires clinical review |
| 3 | `693897189879dc15c6a1de69` | Bivigam | `false` | `true` | Tier 2, requires clinical review |
| 4 | `693897569879dc15c6a1de6a` | Gammagard Liquid | `false` | _missing_ | `pharmacyEligible` field absent → still passes |
| 5 | `6938978c9879dc15c6a1de6b` | Gammaked | _missing_ | _missing_ | Both fields absent → still passes |

### 2b. Patients (matching the 5 pipeline-eligible insurances from Step 1)

```js
// Mongo Compass — collection: patients
{ "we_infuse_id_IV": { "$in": [1007696, 911448, 768371, 927508, 1048912] } }
```

> All 5 patients found. The `we_infuse_id_IV` is the join key into `patient_insurances.we_infuse_patient_id`.

| # | `_id` | `we_infuse_id_IV` | `full_name` | `we_infuse_location_id` | Insurance from Step 1 |
|---|-------|-------------------|-------------|-------------------------|-----------------------|
| 1 | `6807cc0684836fa1a2683a0a` | `768371`  | Leslie Kahn        | `1834` | The Empire Plan |
| 2 | `68484f9097b77dc62b1cd9b9` | `911448`  | Shuwanda Williams  | `1931` | Carefirst BCBS Medigap |
| 3 | `685ef18a795a64b0fd3c27d2` | `927508`  | Lorrie Monger      | `2085` | Tricare East \| Select |
| 4 | `68e814d8629c6322a9ec2f90` | `1007696` | Sara Shine         | `1931` | Tricare East \| Select |
| 5 | `6932ec7bdf5bf0f4dd0d3a2e` | `1048912` | Rebecca Whiteley   | `2085` | Tricare East \| Select |

### 2c. Sites / locations — `patientLocations`

```js
// Mongo Compass — collection: patientLocations
{ "isPharmacyEligible": true }
```

> Total in DB: **53**. The site gate at `ordersTracker.ts:157–161` requires `isPharmacyEligible === true` (missing/`null` defaults to `false`).

| # | `_id` | `location_name` | `we_infuse_id` | `state` | `isPharmacyEligible` |
|---|-------|-----------------|----------------|---------|----------------------|
| 1 | `67b7083a8532e12fdce9d618` | Alexandria, Virginia    | `1931` | VA | `true` |
| 2 | `67b7083a8532e12fdce9d619` | Augusta, Maine          | `1229` | ME | `true` |
| 3 | `67b7083a8532e12fdce9d61a` | Bangor, Maine           | `1645` | ME | `true` |
| 4 | `67b7083a8532e12fdce9d61b` | Bedford, New Hampshire  | `1186` | NH | `true` |
| 5 | `67b7083a8532e12fdce9d61c` | Concord, New Hampshire  | `1223` | NH | `true` |

---

## Suggested combinations

After confirming the home locations of all 5 patients are also pharmacy-eligible (`{ "isPharmacyEligible": true, "we_infuse_id": { "$in": [1834, 2085] } }` returned both Emerson NJ and Fredericksburg VA), we can pair every patient with their natural home site — the cleanest test path:

| # | Patient | `we_infuse_id_IV` | Site (home location) | Site `_id` | Suggested drug | Drug `_id` |
|---|---------|-------------------|----------------------|------------|----------------|------------|
| 1 | Shuwanda Williams | `911448`  | Alexandria, VA (`we_infuse_id 1931`)        | `67b7083a8532e12fdce9d618` | Skyrizi          | `6938863d9879dc15c6a1de64` |
| 2 | Sara Shine        | `1007696` | Alexandria, VA (`we_infuse_id 1931`)        | `67b7083a8532e12fdce9d618` | Asceniv          | `693896d59879dc15c6a1de68` |
| 3 | Leslie Kahn       | `768371`  | Emerson, NJ (`we_infuse_id 1834`)           | `67b7083a8532e12fdce9d61d` | Bivigam          | `693897189879dc15c6a1de69` |
| 4 | Lorrie Monger     | `927508`  | Fredericksburg, VA (`we_infuse_id 2085`)    | `67b7083a8532e12fdce9d629` | Gammagard Liquid | `693897569879dc15c6a1de6a` |
| 5 | Rebecca Whiteley  | `1048912` | Fredericksburg, VA (`we_infuse_id 2085`)    | `67b7083a8532e12fdce9d629` | Gammaked         | `6938978c9879dc15c6a1de6b` |

> The drug-to-row mapping above is arbitrary; any of the 5 drugs from section 2a can pair with any of the 5 patients. Spreading them out just gives us coverage across drug records (including the two with missing `pharmacyEligible` field — Gammagard Liquid and Gammaked — which exercise the `$ne: false` "missing field counts as eligible" path).

---

## Next steps

1. Create an order in the UI selecting one of the suggested patient + site + drug combinations.
2. Verify in `neworders` that the saved record has `patient`, `drug`, `site` ObjectIds matching what we picked.
3. Run the `getEligibilityPipelineStages()` slice (or the full orders-tracker pipeline) in Compass against that one order and confirm `pharmacyEligible === "Yes"`.
