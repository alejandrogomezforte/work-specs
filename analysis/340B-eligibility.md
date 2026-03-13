# 340B Eligibility — Query-Based Analysis

> Part of [MLID-1492 Plan](../plans/MLID-1492-plan-progress.md) — Deliverable 1: 340B Calculation

---

## Approach

**Query-time computation** — instead of storing a calculated `hospital340BEligible` field on each order and maintaining it through triggers/background jobs, the eligibility is computed dynamically via `$lookup` + `$addFields` stages appended to the existing orders-tracker aggregation pipeline.

**Key advantages:**
- No stored field on `neworders` — zero data staleness
- No triggers or background jobs — zero maintenance overhead
- No new indexes on `neworders` — the lookups operate on the related collections
- Always reflects the latest state of hospital systems, patients, and insurances

---

## Involved Collections

Four collections participate in the 340B eligibility calculation. Only fields relevant to the calculation are listed.

### `neworders` (source — already queried by pipeline)

| Field | Type | Purpose |
|-------|------|---------|
| `provider` | string | 10-digit NPI (e.g., `"1234567890"`) — after `expandIdsAggregate`, this becomes a `cmsnpiproviders` document or the original string |
| `patient` | string | ObjectId hex referencing `patients._id` — after `expandIdsAggregate`, this becomes a `patients` document |

### `hospitalSystems` (NEW lookup)

| Field | Type | Purpose |
|-------|------|---------|
| `name` | string | Hospital system name (displayed in result) |
| `providers` | `{ npi: string, name: string }[]` | NPI list to match against order provider |
| `deletedAt` | Date \| null | `null` = active (soft-delete filter) |

### `patient_insurances` (NEW lookup — shared with pharmacy eligibility)

| Field | Type | Purpose |
|-------|------|---------|
| `we_infuse_patient_id` | number | Matched from `patients.we_infuse_id_IV` |
| `priority` | string | `'1'` = primary insurance |
| `status` | string | `'Active'` = current active record |
| `payer_type_name` | string | Insurance type (e.g., `"Commercial"`) |
| `insurance_plan_name` | string | Plan name (used by pharmacy eligibility) |

### `patients` (already looked up by existing pipeline)

| Field | Type | Purpose |
|-------|------|---------|
| `_id` | ObjectId | Matched from `order.patient` |
| `we_infuse_id_IV` | number | Bridge to `patient_insurances` |

### Data Flow

```
neworders
│
├── provider (NPI string) ───────────────► hospitalSystems.providers[].npi
│                                           (WHERE deletedAt == null)
│
└── patient (ObjectId) ─► patients._id
                           └── we_infuse_id_IV ─► patient_insurances.we_infuse_patient_id
                                                   (WHERE priority = '1' AND status = 'Active')
```

---

## Eligibility Resolution

The 340B eligibility calculation returns a **string** with 3 possible outcomes:

| Outcome | Condition |
|---------|-----------|
| `"Waiting on Insurance Input"` | Provider NPI is in a hospital system list, but the patient has no primary active insurance |
| `"Hospital Name 1, Hospital Name 2"` | Provider NPI is in a hospital system list **AND** primary active insurance type is `"Commercial"` |
| `"No"` | Provider NPI is not in any hospital system list, **OR** primary insurance type is not `"Commercial"` |

### Resolution diagram

```
                        ┌─────────────────────┐
                        │   Order enters      │
                        │   340B check        │
                        └──────────┬──────────┘
                                   │
                                   ▼
                  ┌──────────────────────────────────┐
                  │  Is the provider NPI in any      │
                  │  active hospital system?          │
                  │  (deletedAt == null)              │
                  └───────────┬────────────┬─────────┘
                              │            │
                           No │            │ Yes (matched hospitals)
                              ▼            ▼
               ┌──────────────────┐   ┌─────────────────────────┐
               │ "No"             │   │  Does the patient have  │
               │  ■ STOP          │   │  a primary active       │
               └──────────────────┘   │  insurance?             │
                                      └──────┬──────────┬───────┘
                                             │          │
                                          No │          │ Yes
                                             ▼          ▼
                                  ┌─────────────────┐  ┌──────────────────┐
                                  │ "Waiting on     │  │  Is the payer    │
                                  │  Insurance      │  │  type            │
                                  │  Input"         │  │  "Commercial"?   │
                                  │  ■ STOP         │  └──────┬─────┬────┘
                                  └─────────────────┘         │     │
                                                           No │     │ Yes
                                                              ▼     ▼
                                                     ┌──────────┐  ┌───────────────────┐
                                                     │ "No"     │  │ "Hospital ABC,    │
                                                     │  ■ STOP  │  │  Hospital XYZ"    │
                                                     └──────────┘  │  (joined names)   │
                                                                   │  ■ STOP           │
                                                                   └───────────────────┘
```

**3 gates (in order):**

| Gate | Check | Pass condition | Fail result |
|------|-------|----------------|-------------|
| 1. Hospital match | Provider NPI in active hospital system | At least 1 match (deletedAt == null) | "No" |
| 2. Insurance exists | Patient has primary active insurance | Found (priority='1', status='Active') | "Waiting on Insurance Input" |
| 3. Payer type | Insurance type is Commercial | `payer_type_name == "Commercial"` | "No" |

All 3 gates must pass to reach **matched hospital names** (comma-joined). Any single failure short-circuits to its respective result.

---

## Aggregation Pipeline Implementation

### Pipeline Placement

The eligibility stages are appended to `expandIdsAggregate` inside the `$facet.data` pipeline, **after all existing lookups** and **after `$skip/$limit`**. This means:

1. Only the page-sized batch of orders (typically 25–100) is processed
2. The `patient`, `drug`, `site`, and `provider` documents are already resolved
3. Minimal performance impact — equivalent to adding 2 extra lookups per page load

```
$facet:
  data:
    ...sortStage
    $skip / $limit           ← pagination (25–100 orders)
    ...expandIdsAggregate    ← existing lookups (patient, drug, site, provider, etc.)
    ────────────────────
    NEW STAGES HERE:         ← 340B + pharmacy eligibility computation
    ────────────────────
  totalCount: $count
```

### Stage 1 — Extract provider NPI

After `expandIdsAggregate`, the `provider` field is either a `cmsnpiproviders` document (if found) or the original NPI string (if not). We need the NPI as a string for the hospital systems lookup.

```javascript
{
  $addFields: {
    _providerNpi: {
      $ifNull: ['$provider.npi', { $toString: '$provider' }]
    }
  }
}
```

### Stage 2 — Lookup matching hospital systems

Find all active hospital systems whose providers list includes this NPI.

```javascript
{
  $lookup: {
    from: 'hospitalSystems',
    let: { npi: '$_providerNpi' },
    pipeline: [
      {
        $match: {
          $expr: {
            $and: [
              { $eq: ['$deletedAt', null] },
              { $in: ['$$npi', '$providers.npi'] }
            ]
          }
        }
      },
      { $project: { name: 1 } }
    ],
    as: '_340b_hospitals'
  }
}
```

**Notes:**
- `$in: ['$$npi', '$providers.npi']` checks if the NPI exists in the array of provider NPI strings within each hospital system document
- `{ $eq: ['$deletedAt', null] }` matches both `deletedAt: null` and documents where the field doesn't exist
- One NPI can match 0, 1, or many hospital systems
- Only the `name` field is projected (minimizes data transfer)

### Stage 3 — Lookup primary active insurance (SHARED with pharmacy)

This lookup is shared between 340B and pharmacy eligibility. It should only appear once in the pipeline.

```javascript
{
  $lookup: {
    from: 'patient_insurances',
    let: { weInfuseId: '$patient.we_infuse_id_IV' },
    pipeline: [
      {
        $match: {
          $expr: {
            $and: [
              { $eq: ['$we_infuse_patient_id', '$$weInfuseId'] },
              { $eq: ['$priority', '1'] },
              { $eq: ['$status', 'Active'] }
            ]
          }
        }
      },
      { $limit: 1 },
      { $project: { payer_type_name: 1, insurance_plan_name: 1 } }
    ],
    as: '_primaryInsurance'
  }
}
```

**Notes:**
- `$patient.we_infuse_id_IV` is available because the patients lookup in `expandIdsAggregate` has already run and been unwound
- `$limit: 1` — there should be at most one primary active insurance per patient
- Projects both `payer_type_name` (used by 340B) and `insurance_plan_name` (used by pharmacy)

### Stage 4 — Compute 340B eligibility

```javascript
{
  $addFields: {
    hospital340BEligible: {
      $switch: {
        branches: [
          // No hospital match → "No"
          {
            case: { $eq: [{ $size: '$_340b_hospitals' }, 0] },
            then: 'No'
          },
          // Hospital match but no insurance → "Waiting on Insurance Input"
          {
            case: { $eq: [{ $size: { $ifNull: ['$_primaryInsurance', []] } }, 0] },
            then: 'Waiting on Insurance Input'
          },
          // Hospital match + Commercial insurance → hospital names
          {
            case: {
              $eq: [
                { $arrayElemAt: ['$_primaryInsurance.payer_type_name', 0] },
                'Commercial'
              ]
            },
            then: {
              $reduce: {
                input: '$_340b_hospitals.name',
                initialValue: '',
                in: {
                  $cond: [
                    { $eq: ['$$value', ''] },
                    '$$this',
                    { $concat: ['$$value', ', ', '$$this'] }
                  ]
                }
              }
            }
          }
        ],
        // Hospital match but non-Commercial insurance → "No"
        default: 'No'
      }
    }
  }
}
```

**Logic flow:**
1. If `_340b_hospitals` is empty → `"No"` (provider NPI not in any hospital system)
2. If `_primaryInsurance` is empty → `"Waiting on Insurance Input"` (hospital match exists but can't verify insurance)
3. If insurance is `"Commercial"` → join hospital names (e.g., `"Hospital ABC, Hospital XYZ"`)
4. Otherwise → `"No"` (hospital match exists but insurance is not Commercial)

### Stage 5 — Cleanup temporary fields

```javascript
{
  $unset: ['_providerNpi', '_340b_hospitals']
}
```

**Note:** `_primaryInsurance` is NOT cleaned up here — it is shared with the pharmacy eligibility computation that follows. Cleanup of `_primaryInsurance` happens after pharmacy eligibility is computed.

### Complete stage sequence

```javascript
const eligibility340BStages = [
  // 1. Extract provider NPI
  { $addFields: { _providerNpi: { $ifNull: ['$provider.npi', { $toString: '$provider' }] } } },

  // 2. Lookup hospital systems
  { $lookup: { from: 'hospitalSystems', let: { npi: '$_providerNpi' }, pipeline: [...], as: '_340b_hospitals' } },

  // 3. Lookup primary insurance (SHARED — only add once)
  { $lookup: { from: 'patient_insurances', let: { weInfuseId: '$patient.we_infuse_id_IV' }, pipeline: [...], as: '_primaryInsurance' } },

  // 4. Compute 340B eligibility
  { $addFields: { hospital340BEligible: { $switch: { ... } } } },

  // 5. Cleanup 340B-specific temp fields (NOT _primaryInsurance)
  { $unset: ['_providerNpi', '_340b_hospitals'] },
];
```

---

## Performance Considerations

### Display-only (current plan)

When eligibility is a **display-only column** (no filtering or sorting):

| Metric | Value |
|--------|-------|
| Orders processed per page load | 25–100 (after `$skip/$limit`) |
| New $lookup per order: hospitalSystems | 1 (scans `hospitalSystems` collection — small, ~10–50 docs) |
| New $lookup per order: patient_insurances | 1 (indexed lookup by `we_infuse_patient_id`) |
| Additional pipeline stages | 5 (2 lookups + 1 addFields NPI + 1 addFields compute + 1 unset) |
| **Impact** | **Negligible** — comparable to existing lookups already in pipeline |

The `hospitalSystems` collection is small (tens of documents at most), so the lookup is fast even without special indexing. The `patient_insurances` lookup benefits from any existing index on `we_infuse_patient_id`.

### Filter/sort support (future enhancement)

If users need to filter or sort by the eligibility column, the lookup stages would need to move **before the `$facet`** stage. This means:

- All matching orders (not just the page) get the additional lookups
- The dynamic lookup system in `getFilterAndSortAggregateQuery()` would need to be extended
- Performance depends on total order count, not page size

This is a **future enhancement** — start with display-only, add filter/sort if needed.

### Recommended indexes on lookup collections

| Index | Collection | Purpose |
|-------|-----------|---------|
| `{ 'providers.npi': 1, deletedAt: 1 }` | `hospitalSystems` | Speed up NPI matching (optional — collection is small) |
| `{ we_infuse_patient_id: 1, priority: 1, status: 1 }` | `patient_insurances` | Speed up primary insurance lookup |

**Note:** No new indexes on `neworders` are needed — unlike the stored-field approach, there are no reverse lookups or fan-out queries.

---

## Comparison with Stored-Field Approach

| Aspect | Stored Field (old) | Query-Time (new) |
|--------|-------------------|------------------|
| Data freshness | Stale until trigger fires | Always current |
| Triggers needed | 5 (order create/update, hospital NPI update, hospital rename, insurance webhook) | 0 |
| Background jobs | 3 (hospital NPI, hospital rename, insurance webhook) | 0 |
| New indexes on `neworders` | 2 (`provider`, `patient`) | 0 |
| Stored field on `neworders` | `hospital340BEligible` (string) | None |
| Query cost per page load | 0 (pre-computed) | 2 extra lookups on 25–100 orders |
| Implementation complexity | High (service + triggers + jobs + tests) | Low (pipeline stages only) |
| Filter/sort support | Built-in (stored field) | Requires pipeline restructuring |
