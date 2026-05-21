# MLID-2192 — Data Testing: Insurance Status "Undetermined"

Working notebook for verifying that orders whose primary insurance has `status = "Undetermined"` (and `priority = "1"`) are now picked up by the `patient_insurances` `$lookup` instead of being silently dropped.

- **Branch:** `fix/MLID-2192-undetermined-insurance`
- **Plan:** `docs/agomez/plans/MLID-2192-undetermined-insurance.md`
- **Code change:** `apps/web/services/mongodb/ordersTracker.ts:372` — `{ $eq: ['$status', 'Active'] }` → `{ $in: ['$status', ['Active', 'Undetermined']] }`
- **Sister change:** [MLID-2194](./MLID-2194-data-testing.md) (Medicare Supplement allow-list)

---

## Behavior change

The fix only widens the `$lookup` predicate. Three downstream consumers see the change:

| Consumer | Before fix | After fix |
|----------|------------|-----------|
| New Orders "Insurance" column (gated by `ORDERS_TRACKER_IG_COLUMNS`) | Blank for Undetermined-status patients | Shows the insurance plan name |
| Pharmacy `$switch` (`pharmacyEligible`, gated by `ORDER_ELIGIBILITY`) | `"Waiting on Insurance Input"` (lookup empty) | Computed normally — `"Yes"`, `"No"`, depending on payer/plan |
| 340B `$switch` (`hospital340BEligible`, no extra flag) | `"Waiting on Insurance Input"` (lookup empty) | Computed normally — `"Yes"` (with hospital match), `"No"` |

For end-to-end testing we want one or more orders whose primary insurance is `priority: '1'` + `status: 'Undetermined'`, then verify the change in all three places.

---

## Step 1 — Find Undetermined + priority `'1'` patient insurances

Run in **Mongo Compass** against `patient_insurances`.

```js
{ "status": "Undetermined", "priority": "1" }
```

> Total in DB: **97**. These are the rows that the lookup was previously dropping.

### Sample documents (5)

Pulled via MCP `find`:

| # | `_id` | `we_infuse_patient_id` | `payer_type_name` | `insurance_plan_name` | `status` | `priority` |
|---|-------|------------------------|-------------------|-----------------------|----------|------------|
| 1 | `694304dcf0484dead5cae74d` | `1045568` | Commercial         | Department of Veterans Affairs    | Undetermined | `1` |
| 2 | `694304dcf0484dead5cae750` | `1046985` | Commercial         | Horizon BCBS of NJ \\ Commercial   | Undetermined | `1` |
| 3 | `694304dcf0484dead5cae760` | `1055264` | Commercial         | Horizon BCBS of NJ \\ Commercial   | Undetermined | `1` |
| 4 | `694304dcf0484dead5cae76b` | `1055681` | Commercial         | Harvard Pilgrim Commercial        | Undetermined | `1` |
| 5 | `694304ddf0484dead5cae78e` | `1052451` | Free-Med Program   | Prolia Amgen Free Drug Program    | Undetermined | `1` |

<details>
<summary>Raw documents (click to expand)</summary>

```json
[
  {
    "_id": "694304dcf0484dead5cae74d",
    "we_infuse_insurance_id": 1611187,
    "we_infuse_patient_id": 1045568,
    "insurance_company_id": null,
    "insurance_company_name": null,
    "insurance_plan_id": 69027,
    "insurance_plan_name": "Department of Veterans Affairs",
    "payer_type_name": "Commercial",
    "priority": "1",
    "status": "Undetermined",
    "member_id": "7346243588",
    "effective_date": null,
    "term_date": null,
    "updated_date": "2025-12-02T00:00:00Z"
  },
  {
    "_id": "694304dcf0484dead5cae750",
    "we_infuse_insurance_id": 1620020,
    "we_infuse_patient_id": 1046985,
    "insurance_company_id": 18,
    "insurance_company_name": "Horizon Blue Cross and Blue Shield",
    "insurance_plan_id": 70040,
    "insurance_plan_name": "Horizon BCBS of NJ \\ Commercial",
    "payer_type_name": "Commercial",
    "priority": "1",
    "status": "Undetermined",
    "member_id": "JGH3HZN48733750",
    "group_id": "808058W",
    "effective_date": null,
    "term_date": null,
    "updated_date": "2025-12-08T00:00:00Z"
  },
  {
    "_id": "694304dcf0484dead5cae760",
    "we_infuse_insurance_id": 1634661,
    "we_infuse_patient_id": 1055264,
    "insurance_company_id": 18,
    "insurance_company_name": "Horizon Blue Cross and Blue Shield",
    "insurance_plan_id": 70040,
    "insurance_plan_name": "Horizon BCBS of NJ \\ Commercial",
    "payer_type_name": "Commercial",
    "priority": "1",
    "status": "Undetermined",
    "member_id": "YHX3HZN90098130",
    "group_id": "031306A",
    "effective_date": null,
    "term_date": null,
    "updated_date": "2025-12-16T00:00:00Z"
  },
  {
    "_id": "694304dcf0484dead5cae76b",
    "we_infuse_insurance_id": 1633995,
    "we_infuse_patient_id": 1055681,
    "insurance_company_id": 95,
    "insurance_company_name": "Harvard Pilgrim",
    "insurance_plan_id": 85609,
    "insurance_plan_name": "Harvard Pilgrim Commercial",
    "payer_type_name": "Commercial",
    "priority": "1",
    "status": "Undetermined",
    "member_id": "HP620782000",
    "effective_date": null,
    "term_date": null,
    "updated_date": "2025-12-16T00:00:00Z"
  },
  {
    "_id": "694304ddf0484dead5cae78e",
    "we_infuse_insurance_id": 1627789,
    "we_infuse_patient_id": 1052451,
    "insurance_company_id": null,
    "insurance_company_name": null,
    "insurance_plan_id": 69066,
    "insurance_plan_name": "Prolia Amgen Free Drug Program",
    "payer_type_name": "Free-Med Program",
    "priority": "1",
    "status": "Undetermined",
    "member_id": "6551053",
    "effective_date": null,
    "term_date": null,
    "updated_date": "2025-12-11T00:00:00Z"
  }
]
```

</details>

### Observation — payer_type distribution

The first 5 records are **all non-Medicare**: 4 are `Commercial`, 1 is `Free-Med Program`. To know the expected `pharmacyEligible` outcome for each Commercial record, we need the matching `insurance_plans.isPharmacyEligible`:

```js
// Mongo Compass — collection: insurance_plans
{ "planName": { "$in": [
  "Department of Veterans Affairs",
  "Horizon BCBS of NJ \\ Commercial",
  "Harvard Pilgrim Commercial"
] } }
```

> **Manual seed (2026-05-05):** Originally all three plans had `isPharmacyEligible: false`, which would have made every combination resolve to `pharmacyEligible: "No"`. To get end-to-end `"Yes"` cases for testing, the three plan records were manually updated in MongoDB to `isPharmacyEligible: true`. To revert, re-run an `updateMany` setting the field back to `false`.

| `planName` | `_id` | `isPharmacyEligible` (post-update) |
|------------|-------|------------------------------------|
| Department of Veterans Affairs   | `6977d66701fc42e5479df6a6` | `true` |
| Horizon BCBS of NJ \\ Commercial | `6977d66701fc42e5479df74c` | `true` |
| Harvard Pilgrim Commercial       | `6977d66701fc42e5479df7b1` | `true` |

> "Prolia Amgen Free Drug Program" never gets looked up (its `payer_type_name` is `Free-Med Program`, not Commercial/Medicaid).

**With this manual update in place, 4 of the 5 sample records will resolve to `pharmacyEligible: "Yes"` after the fix** — exercising the Path B branch (Commercial + eligible plan) end-to-end. The 5th record (Saiqua Ferdous → Prolia Free-Med Program) still resolves to `"No"`, which exercises the "fail Gate 4" path.

---

## Step 2 — Order ingredients (drug, patient, site)

### 2a. Drugs — `lidrugs` (same predicate as MLID-2194)

```js
{ "is_treatment": { "$ne": true }, "pharmacyEligible": { "$ne": false } }
```

> Total in DB: **41**.

| # | `_id` | `name` | `is_treatment` | `pharmacyEligible` |
|---|-------|--------|----------------|--------------------|
| 1 | `6938863d9879dc15c6a1de64` | Skyrizi | `false` | `true` |
| 2 | `693896d59879dc15c6a1de68` | Asceniv | `false` | `true` |
| 3 | `693897189879dc15c6a1de69` | Bivigam | `false` | `true` |
| 4 | `693897569879dc15c6a1de6a` | Gammagard Liquid | `false` | _missing_ |
| 5 | `6938978c9879dc15c6a1de6b` | Gammaked | _missing_ | _missing_ |

### 2b. Patients (matching the 5 Undetermined insurances)

```js
{ "we_infuse_id_IV": { "$in": [1045568, 1046985, 1055264, 1055681, 1052451] } }
```

| # | `_id` | `we_infuse_id_IV` | `full_name` | `we_infuse_location_id` | Insurance plan (Step 1) |
|---|-------|-------------------|-------------|-------------------------|--------------------------|
| 1 | `692eea08df5bf0f4dd0c707b` | `1045568` | Jane Barker        | `1983` | Department of Veterans Affairs |
| 2 | `693088eddf5bf0f4dd0cc641` | `1046985` | Nedreta Ajkic      | `1971` | Horizon BCBS of NJ \\ Commercial |
| 3 | `69403ec165036467cc9cc1d2` | `1055264` | Samantha Keown     | `1983` | Horizon BCBS of NJ \\ Commercial |
| 4 | `69407df2f308335625c6cd5d` | `1055681` | Stephanie Chancey  | `2309` | Harvard Pilgrim Commercial |
| 5 | `6939ca54df5bf0f4dd0e4ddd` | `1052451` | Saiqua Ferdous     | `2334` | Prolia Amgen Free Drug Program |

### 2c. Sites / locations — `patientLocations`

Same predicate as MLID-2194 — `{ "isPharmacyEligible": true }`. Total in DB: **53**.

For these 5 patients, all home locations are pharmacy-eligible (queried via `{ "isPharmacyEligible": true, "we_infuse_id": { "$in": [1983, 1971, 2309, 2334] } }`):

| # | `_id` | `location_name` | `we_infuse_id` | `state` | `isPharmacyEligible` |
|---|-------|-----------------|----------------|---------|----------------------|
| 1 | `67b7083a8532e12fdce9d627` | Toms River, New Jersey  | `1983` | NJ | `true` |
| 2 | `67b7083a8532e12fdce9d61f` | Morristown, New Jersey  | `1971` | NJ | `true` |
| 3 | `688b08df8fb88b8065423006` | Shrewsbury, Massachusetts | `2309` | MA | `true` |
| 4 | `68a01fab4843f9a3df65bcdb` | Edison, New Jersey      | `2334` | NJ | `true` |

---

## Suggested combinations (5)

Each row pairs a patient with their pharmacy-eligible home location and an arbitrary drug. **After the manual seed of the 3 Commercial plans to `isPharmacyEligible: true`, combos 1–4 now resolve to `pharmacyEligible: "Yes"`.** Combo 5 still resolves to `"No"` (different failure mode — payer type isn't Commercial/Medicaid, so the plan flip can't help).

| # | Patient | `we_infuse_id_IV` | Site (home location) | Site `_id` | Suggested drug | Drug `_id` | Expected `pharmacyEligible` |
|---|---------|-------------------|----------------------|------------|----------------|------------|------------------------------|
| 1 | Jane Barker        | `1045568` | Toms River, NJ (`1983`)    | `67b7083a8532e12fdce9d627` | Skyrizi          | `6938863d9879dc15c6a1de64` | ✅ `"Yes"` — VA plan now `isPharmacyEligible: true` |
| 2 | Nedreta Ajkic      | `1046985` | Morristown, NJ (`1971`)    | `67b7083a8532e12fdce9d61f` | Asceniv          | `693896d59879dc15c6a1de68` | ✅ `"Yes"` — Horizon plan now `isPharmacyEligible: true` |
| 3 | Samantha Keown     | `1055264` | Toms River, NJ (`1983`)    | `67b7083a8532e12fdce9d627` | Bivigam          | `693897189879dc15c6a1de69` | ✅ `"Yes"` — Horizon plan now `isPharmacyEligible: true` |
| 4 | Stephanie Chancey  | `1055681` | Shrewsbury, MA (`2309`)    | `688b08df8fb88b8065423006` | Gammagard Liquid | `693897569879dc15c6a1de6a` | ✅ `"Yes"` — Harvard Pilgrim plan now `isPharmacyEligible: true` |
| 5 | Saiqua Ferdous     | `1052451` | Edison, NJ (`2334`)        | `68a01fab4843f9a3df65bcdb` | Gammaked         | `6938978c9879dc15c6a1de6b` | ❌ `"No"` — Free-Med Program payer type fails Gate 4 regardless of plan |

---

## Next steps

1. Pick one of combos 1–4 (any of the 4 patients with Commercial coverage) and create an order with the suggested drug + site.
2. Open the order in the New Orders view; verify:
   - The "Insurance" column shows the plan name (was blank pre-fix — proves MLID-2192 lookup widening).
   - The pharmacy eligibility column shows `"Yes"` (was `"Waiting on Insurance Input"` pre-fix — proves the lookup result is now flowing into the `$switch` and Path B is wired correctly).
   - The 340B eligibility column shows `"No"` or `"Yes"` (was `"Waiting on Insurance Input"` pre-fix).
3. Optionally also create combo 5 (Saiqua Ferdous) to confirm a `"No"` outcome on the same Undetermined-status path — exercises Gate 4 failure (payer type mismatch).
4. **After testing**, revert the manual seed:
   ```js
   db.insurance_plans.updateMany(
     { "_id": { "$in": [
         ObjectId("6977d66701fc42e5479df74c"),
         ObjectId("6977d66701fc42e5479df6a6"),
         ObjectId("6977d66701fc42e5479df7b1")
     ]}},
     { "$set": { "isPharmacyEligible": false } }
   )
   ```
