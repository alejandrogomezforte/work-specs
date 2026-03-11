# MLID-1858 — 340B pipeline stages

## Task Reference

- **Jira**: [MLID-1858](https://localinfusion.atlassian.net/browse/MLID-1858)
- **Story Points**: 3
- **Branch**: `feature/MLID-1858-340b-pipeline-stages`
- **Base Branch**: `epic/MLID-1492-order-eligibility`
- **Status**: To Do
- **Epic Plan**: [MLID-1492-plan-progress.md](./MLID-1492-plan-progress.md)

---

## Summary

Add 340B eligibility pipeline stages to the orders tracker aggregation pipeline. When `includeEligibility` is `true` and `category === 'new'`, append `$lookup` + `$addFields` stages after `expandIdsAggregate` inside `$facet.data`. This includes the shared primary insurance lookup (reused by D2-T1 pharmacy).

---

## Codebase Analysis

- **`getOrdersAggregate()`** (ordersTracker.ts:330–769): Builds the full aggregation pipeline. Paginated path uses `$facet.data` with `[...sortStage, $skip, $limit, ...expandIdsAggregate]`. Insertion point is after `...expandIdsAggregate` at ~line 746.
- **`getOrders()`** (ordersTracker.ts:815–920): Calls `getOrdersAggregate()` at line 904. Only caller that should pass `includeEligibility: true`.
- **`GetOrdersParams<K>`** (ordersTracker.ts:802–813): Type for `getOrders()` params — needs `includeEligibility?: boolean`.
- **Other callers of `getOrdersAggregate()`**: `getOrderById` (line 795), `createOrder` (line 983), `updateOrder` (line 1050), `getOrdersByPatient` (line 1101/1109) — all default `includeEligibility` to `false`.
- **Test file** (`__tests__/ordersTracker.test.ts`): Uses `mockAggregate.mock.calls[N][0]` to verify pipeline shape. Mock pattern: `jest.fn().mockReturnValue({ toArray: jest.fn().mockResolvedValue([...]) })`.
- **Analysis doc** (`docs/agomez/analysis/340B-eligibility.md`): Full stage definitions with `$switch` logic.

---

## Implementation Steps

### Step 1 — RED: Write failing tests

- **File**: `apps/web/services/mongodb/__tests__/ordersTracker.test.ts`
- **Tests**:
  1. `getOrders` with `category='new'` + `includeEligibility=true` → pipeline contains eligibility `$lookup` stages (`hospitalSystems`, `patient_insurances`) and `$addFields: { hospital340BEligible }`
  2. `getOrders` with `category='new'` + `includeEligibility=false` → pipeline does NOT contain eligibility stages
  3. `getOrders` with `category='maintenance'` + `includeEligibility=true` → pipeline does NOT contain eligibility stages (maintenance excluded)

### Step 2 — GREEN: Add includeEligibility param + pipeline stages

- **File**: `apps/web/services/mongodb/ordersTracker.ts`
- **Changes**:
  1. Add `includeEligibility?: boolean` to `getOrdersAggregate()` params and to `GetOrdersParams<K>`
  2. Create helper function `get340BEligibilityStages(): Document[]` returning stages 1–5:
     - `$addFields: { _providerNpi }` — extract NPI from provider doc or string
     - `$lookup: hospitalSystems` — match NPI in `providers[].npi` where `deletedAt == null`
     - `$lookup: patient_insurances` — primary active insurance (shared with pharmacy)
     - `$addFields: { hospital340BEligible }` — `$switch` computation
     - `$unset: ['_providerNpi', '_340b_hospitals', '_primaryInsurance']` — cleanup temp fields
  3. In paginated `$facet.data` array, conditionally spread: `...(category === 'new' && includeEligibility ? get340BEligibilityStages() : [])`
  4. Pass `includeEligibility` through `getOrders()` → `getOrdersAggregate()`

### Step 3 — REFACTOR: Clean up and verify

- Ensure all callers compile (no breaking changes — param is optional, defaults to `false`)
- Verify types with `npm run types:check`

### Step 4 — Verify quality

- `cd apps/web && npx jest --testPathPattern="ordersTracker" --no-coverage`
- `npm run types:check`
- `cd apps/web && npx prettier --write services/mongodb/ordersTracker.ts services/mongodb/__tests__/ordersTracker.test.ts`
- `cd apps/web && npx eslint --fix services/mongodb/ordersTracker.ts services/mongodb/__tests__/ordersTracker.test.ts`

---

## Pipeline Stages (340B — D1-T2)

```javascript
// Stage 1: Extract provider NPI
{ $addFields: { _providerNpi: { $ifNull: ['$provider.npi', { $toString: '$provider' }] } } }

// Stage 2: Lookup hospital systems by NPI
{ $lookup: {
    from: 'hospitalSystems',
    let: { npi: '$_providerNpi' },
    pipeline: [
      { $match: { $expr: { $and: [
        { $eq: ['$deletedAt', null] },
        { $in: ['$$npi', '$providers.npi'] }
      ] } } },
      { $project: { name: 1 } }
    ],
    as: '_340b_hospitals'
} }

// Stage 3: Lookup primary active insurance (SHARED with pharmacy D2-T1)
{ $lookup: {
    from: 'patient_insurances',
    let: { weInfuseId: '$patient.we_infuse_id_IV' },
    pipeline: [
      { $match: { $expr: { $and: [
        { $eq: ['$we_infuse_patient_id', '$$weInfuseId'] },
        { $eq: ['$priority', '1'] },
        { $eq: ['$status', 'Active'] }
      ] } } },
      { $limit: 1 },
      { $project: { payer_type_name: 1, insurance_plan_name: 1 } }
    ],
    as: '_primaryInsurance'
} }

// Stage 4: Compute 340B eligibility
{ $addFields: { hospital340BEligible: { $switch: {
    branches: [
      { case: { $eq: [{ $size: '$_340b_hospitals' }, 0] }, then: 'No' },
      { case: { $eq: [{ $size: { $ifNull: ['$_primaryInsurance', []] } }, 0] }, then: 'Waiting on Insurance Input' },
      { case: { $eq: [{ $arrayElemAt: ['$_primaryInsurance.payer_type_name', 0] }, 'Commercial'] },
        then: { $reduce: { input: '$_340b_hospitals.name', initialValue: '',
          in: { $cond: [{ $eq: ['$$value', ''] }, '$$this', { $concat: ['$$value', ', ', '$$this'] }] }
        } }
      }
    ],
    default: 'No'
} } } }

// Stage 5: Cleanup temp fields
{ $unset: ['_providerNpi', '_340b_hospitals', '_primaryInsurance'] }
```

> **Note for D2-T1:** When pharmacy stages are added, Stage 5 will be modified to keep `_primaryInsurance` until after pharmacy computation, and `$unset` will be consolidated into a single final cleanup stage.

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/services/mongodb/ordersTracker.ts` | Modify | Add `includeEligibility` param, `get340BEligibilityStages()` helper, conditional pipeline spread |
| `apps/web/services/mongodb/__tests__/ordersTracker.test.ts` | Modify | Add tests for eligibility pipeline inclusion/exclusion |

---

## Testing Strategy

- **Unit tests**: Verify pipeline shape via `mockAggregate.mock.calls[0][0]` — check for presence/absence of eligibility stages based on `category` and `includeEligibility`
- **No integration tests**: Pipeline correctness against real data is validated by E2E after deployment

---

## Security Considerations

- No PHI exposure — pipeline only computes eligibility labels, no new data leaves the system
- Feature flag gated — stages only execute when `ORDER_ELIGIBILITY` is enabled (API route check in D1-T4)

---

## Open Questions

- None — pipeline stages are fully specified in the analysis doc.
