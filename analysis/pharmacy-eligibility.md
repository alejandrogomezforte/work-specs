# Pharmacy Eligibility — Query-Based Analysis

> Part of [MLID-1492 Plan](../plans/MLID-1492-plan-progress.md) — Deliverable 2: Pharmacy Eligible Calculation

---

## Approach

**Query-time computation** — instead of storing a calculated `pharmacyEligible` field on each order and maintaining it through triggers/background jobs, the eligibility is computed dynamically via `$lookup` + `$addFields` stages appended to the existing orders-tracker aggregation pipeline.

This follows the same strategy as the [340B eligibility](./340B-eligibility.md) and shares the `_primaryInsurance` lookup stage.

---

## Involved Collections

Six collections participate in the pharmacy eligibility calculation. Only fields relevant to the calculation are listed.

### `neworders` (source — already queried by pipeline)

| Field | Type | Purpose |
|-------|------|---------|
| `drug` | string | ObjectId hex referencing `lidrugs._id` — after `expandIdsAggregate`, this becomes a `lidrugs` document |
| `site` | string | ObjectId hex referencing `patientLocations._id` — after `expandIdsAggregate`, this becomes a `patientLocations` document |
| `patient` | string | ObjectId hex referencing `patients._id` — after `expandIdsAggregate`, this becomes a `patients` document |

### `lidrugs` (already looked up by existing pipeline)

| Field | Type | Purpose |
|-------|------|---------|
| `_id` | ObjectId | Matched from `order.drug` |
| `is_treatment` | boolean | If `true`, order is automatically not pharmacy eligible |
| `pharmacyEligible` | boolean \| undefined | Whether the drug is pharmacy eligible. **Updated 3/4/2026:** only `false` means ineligible; `true`, `null`, or `undefined` all count as eligible |

### `patientLocations` (already looked up by existing pipeline)

| Field | Type | Purpose |
|-------|------|---------|
| `_id` | ObjectId | Matched from `order.site` |
| `isPharmacyEligible` | boolean \| undefined | Whether the location is pharmacy eligible |

### `patients` (already looked up by existing pipeline)

| Field | Type | Purpose |
|-------|------|---------|
| `_id` | ObjectId | Matched from `order.patient` |
| `we_infuse_id_IV` | number | Bridge to `patient_insurances` |

### `patient_insurances` (SHARED lookup with 340B — already added by 340B stages)

| Field | Type | Purpose |
|-------|------|---------|
| `we_infuse_patient_id` | number | Matched from `patients.we_infuse_id_IV` |
| `priority` | string | `'1'` = primary insurance |
| `status` | string | `'Active'` = current active record |
| `payer_type_name` | string | Insurance type (e.g., `"Medicare"`, `"Commercial"`) |
| `insurance_plan_name` | string | Bridge to `insurance_plans.planName` |

### `insurance_plans` (NEW lookup)

| Field | Type | Purpose |
|-------|------|---------|
| `planName` | string | Matched from `patient_insurances.insurance_plan_name` |
| `isPharmacyEligible` | boolean | Whether the insurance plan is pharmacy eligible |

### Data Flow

```
neworders
│
├── drug (ObjectId) ─► lidrugs._id
│                       └── pharmacyEligible: boolean
│
├── site (ObjectId) ─► patientLocations._id
│                       └── isPharmacyEligible: boolean
│
└── patient (ObjectId) ─► patients._id
                           └── we_infuse_id_IV ─► patient_insurances.we_infuse_patient_id
                                                   (WHERE priority='1' AND status='Active')
                                                   ├── payer_type_name
                                                   └── insurance_plan_name ─► insurance_plans.planName
                                                                               └── isPharmacyEligible: boolean
```

---

## Eligibility Resolution

The pharmacy eligibility calculation returns a **string** with 3 possible outcomes:

| Outcome | Condition |
|---------|-----------|
| `"Waiting on Insurance Input"` | Patient has no primary active insurance |
| `"No"` | Drug is a treatment (`is_treatment == true`) |
| `"Yes"` | Drug is **not** a treatment **AND** **not** explicitly ineligible **AND** location is pharmacy eligible **AND** insurance condition is met |
| `"No"` | Any of the criteria is not met |

> **Updated 3/4/2026:** The drug check changed from "must be explicitly `true`" to "must not be explicitly `false`". Drugs with `pharmacyEligible: null` or `undefined` are now treated as eligible.
>
> **Updated 3/24/2026:** Added treatment check — if the drug is a treatment (`is_treatment == true`), the order is automatically not pharmacy eligible.

### Insurance condition

The insurance check has two paths:

| Payer Type | Condition |
|-----------|-----------|
| `"Medicare"` or `"Medicare Advantage"` | **Auto-pass** — no further check needed |
| `"Commercial"` or `"Medicaid"` | Pass **only if** the matching `insurance_plans` record has `isPharmacyEligible == true` |
| Any other type | **Fail** |

### Resolution diagram

```
order
│
├── 1. Does the patient have primary active insurance?
│   └──► patient_insurances (priority='1', status='Active')
│        └── No insurance found → "Waiting on Insurance Input" (stop here)
│
├── 2. Is the drug a treatment?
│   └──► drug.is_treatment == true? → "No" (stop here)
│
├── 3. Is the drug NOT explicitly ineligible?
│   └──► drug.pharmacyEligible != false? (true/null/undefined all pass)
│
├── 4. Is the location pharmacy eligible?
│   └──► site.isPharmacyEligible == true?
│
├── 5. Does the insurance meet pharmacy criteria?
│   ├── Medicare / Medicare Advantage → auto-pass
│   ├── Commercial / Medicaid → check insurance_plans.isPharmacyEligible == true
│   └── Other → fail
│
└── All four checks pass? → "Yes"
    Any check fails?       → "No"
```

---

## Aggregation Pipeline Implementation

### Pipeline Placement

The pharmacy eligibility stages are appended **immediately after** the 340B eligibility stages in `expandIdsAggregate`, sharing the `_primaryInsurance` lookup.

```
$facet:
  data:
    ...sortStage
    $skip / $limit
    ...expandIdsAggregate        ← existing lookups (patient, drug, site, provider)
    ────────────────────
    340B stages:                 ← extracts _providerNpi, looks up hospitalSystems
      _primaryInsurance lookup   ← SHARED — added once
      hospital340BEligible compute
    ────────────────────
    PHARMACY STAGES HERE:        ← uses existing drug, site + shared _primaryInsurance
      _insurancePlan lookup      ← NEW
      pharmacyEligible compute
    ────────────────────
    SHARED CLEANUP               ← unset all temp fields
    ────────────────────
  totalCount: $count
```

### Existing data available (no new lookups needed)

After `expandIdsAggregate` + 340B stages, these fields are already resolved:

| Field | Source | Relevant property |
|-------|--------|-------------------|
| `$drug` | `lidrugs` lookup in expandIdsAggregate | `pharmacyEligible` |
| `$site` | `patientLocations` lookup in expandIdsAggregate | `isPharmacyEligible` |
| `$_primaryInsurance` | Shared lookup from 340B stages | `payer_type_name`, `insurance_plan_name` |

### Stage 1 — Lookup insurance plan

Only needed when payer type is `"Commercial"` or `"Medicaid"`. The `$lookup` always runs but returns empty results for non-matching plan names (which is handled in the `$addFields` computation).

```javascript
{
  $lookup: {
    from: 'insurance_plans',
    let: {
      planName: { $arrayElemAt: ['$_primaryInsurance.insurance_plan_name', 0] }
    },
    pipeline: [
      {
        $match: {
          $expr: { $eq: ['$planName', '$$planName'] }
        }
      },
      { $limit: 1 },
      { $project: { isPharmacyEligible: 1 } }
    ],
    as: '_insurancePlan'
  }
}
```

**Notes:**
- `$$planName` will be `null` if `_primaryInsurance` is empty (no primary insurance), causing the lookup to return no results — this is correct behavior
- `$limit: 1` — plan names should be unique, but defensive limiting
- Only projects `isPharmacyEligible` to minimize data transfer

### Stage 2 — Compute pharmacy eligibility

```javascript
{
  $addFields: {
    pharmacyEligible: {
      $switch: {
        branches: [
          // No primary insurance → "Waiting on Insurance Input"
          {
            case: { $eq: [{ $size: { $ifNull: ['$_primaryInsurance', []] } }, 0] },
            then: 'Waiting on Insurance Input'
          },
          // Drug is a treatment → "No"
          {
            case: { $eq: [{ $ifNull: ['$drug.is_treatment', false] }, true] },
            then: 'No'
          },
          // All four conditions met → "Yes"
          {
            case: {
              $and: [
                // Drug is NOT explicitly ineligible (true/null/undefined all pass, only false fails)
                { $ne: [{ $ifNull: ['$drug.pharmacyEligible', null] }, false] },
                // Location is pharmacy eligible
                { $eq: [{ $ifNull: ['$site.isPharmacyEligible', false] }, true] },
                // Insurance meets criteria
                {
                  $or: [
                    // Medicare / Medicare Advantage → auto-pass
                    {
                      $in: [
                        { $arrayElemAt: ['$_primaryInsurance.payer_type_name', 0] },
                        ['Medicare', 'Medicare Advantage']
                      ]
                    },
                    // Commercial / Medicaid with eligible plan
                    {
                      $and: [
                        {
                          $in: [
                            { $arrayElemAt: ['$_primaryInsurance.payer_type_name', 0] },
                            ['Commercial', 'Medicaid']
                          ]
                        },
                        {
                          $eq: [
                            { $arrayElemAt: ['$_insurancePlan.isPharmacyEligible', 0] },
                            true
                          ]
                        }
                      ]
                    }
                  ]
                }
              ]
            },
            then: 'Yes'
          }
        ],
        // Any condition not met → "No"
        default: 'No'
      }
    }
  }
}
```

**Logic flow:**
1. If `_primaryInsurance` is empty → `"Waiting on Insurance Input"`
2. If drug is a treatment → `"No"`
3. If drug eligible + location eligible + insurance criteria met → `"Yes"`
4. Otherwise → `"No"`

**Edge cases handled by `$ifNull` defaults:**
- `drug` is a string (not found in lidrugs) → `is_treatment` is undefined → defaults to `false` → passes; `pharmacyEligible` is undefined → defaults to `null` → passes (not explicitly `false`)
- `site` is null (not found) → `isPharmacyEligible` is undefined → defaults to `false`
- Insurance plan not found → `isPharmacyEligible` is undefined → lookup returns empty → not `true` → fails check

### Stage 3 — Cleanup ALL temporary fields (shared)

This stage cleans up all temporary fields from both 340B and pharmacy computations:

```javascript
{
  $unset: ['_providerNpi', '_340b_hospitals', '_primaryInsurance', '_insurancePlan']
}
```

### Complete stage sequence (both eligibilities combined)

```javascript
const eligibilityStages = [
  // === 340B STAGES ===
  // 1. Extract provider NPI
  { $addFields: { _providerNpi: { $ifNull: ['$provider.npi', { $toString: '$provider' }] } } },

  // 2. Lookup hospital systems
  { $lookup: { from: 'hospitalSystems', ..., as: '_340b_hospitals' } },

  // === SHARED STAGE ===
  // 3. Lookup primary insurance
  { $lookup: { from: 'patient_insurances', ..., as: '_primaryInsurance' } },

  // === 340B COMPUTE ===
  // 4. Compute 340B eligibility
  { $addFields: { hospital340BEligible: { $switch: { ... } } } },

  // === PHARMACY STAGES ===
  // 5. Lookup insurance plan
  { $lookup: { from: 'insurance_plans', ..., as: '_insurancePlan' } },

  // 6. Compute pharmacy eligibility
  { $addFields: { pharmacyEligible: { $switch: { ... } } } },

  // === SHARED CLEANUP ===
  // 7. Remove all temporary fields
  { $unset: ['_providerNpi', '_340b_hospitals', '_primaryInsurance', '_insurancePlan'] },
];
```

These stages are appended at the end of `expandIdsAggregate` in `getOrdersAggregate()` (file: `services/mongodb/ordersTracker.ts`).

---

## Performance Considerations

### Display-only (current plan)

When eligibility is a **display-only column** (no filtering or sorting):

| Metric | Value |
|--------|-------|
| Orders processed per page load | 25–100 (after `$skip/$limit`) |
| New $lookup: insurance_plans | 1 per order (indexed by `planName`) |
| Reused from 340B: _primaryInsurance | Already computed — no additional cost |
| Reused from pipeline: drug, site | Already computed — no additional cost |
| Additional pipeline stages (pharmacy-only) | 2 (1 lookup + 1 addFields) |
| **Impact** | **Negligible** — 1 extra lookup on top of 340B stages |

The pharmacy eligibility adds minimal cost because it reuses the `_primaryInsurance` lookup from 340B and the `drug`/`site` lookups from the existing pipeline. The only new lookup is `insurance_plans`, which is a small collection matched by `planName`.

### Filter/sort support (future enhancement)

Same consideration as 340B — if users need to filter or sort by the eligibility column, the lookup stages would need to move before the `$facet` stage. This is a future enhancement.

### Recommended indexes on lookup collections

| Index | Collection | Purpose |
|-------|-----------|---------|
| `{ planName: 1 }` | `insurance_plans` | Speed up plan name matching |
| `{ we_infuse_patient_id: 1, priority: 1, status: 1 }` | `patient_insurances` | Speed up primary insurance lookup (shared with 340B) |

**Note:** No new indexes on `neworders` are needed.

---

## Comparison with Stored-Field Approach

| Aspect | Stored Field (old) | Query-Time (new) |
|--------|-------------------|------------------|
| Data freshness | Stale until trigger fires | Always current |
| Triggers needed | 7 (order CRUD, drug flag, location flag, plan flag, insurance webhook, plan webhook) | 0 |
| Background jobs | 5 (drug, location, plan flag, insurance webhook, plan webhook) | 0 |
| New indexes on `neworders` | 4 (`drug`, `site`, `patient`, `provider`) | 0 |
| Stored field on `neworders` | `pharmacyEligible` (string) | None |
| Query cost per page load | 0 (pre-computed) | 1 extra lookup on 25–100 orders (on top of 340B) |
| Implementation complexity | Very high (service + 7 triggers + 5 jobs + tests) | Low (pipeline stages only) |
| Deepest reverse lookup chain | 3-hop (plan → patient_insurances → patients → orders) | N/A — no reverse lookups |
| Filter/sort support | Built-in (stored field) | Requires pipeline restructuring |
