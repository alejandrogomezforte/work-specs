# MLID-2192 — Insurance Lookup: Include "Undetermined" Status

## Task Reference

- **Jira**: [MLID-2192](https://localinfusion.atlassian.net/browse/MLID-2192)
- **Story Points**: 2
- **Sprint**: MLID Sprint 26
- **Branch**: `fix/MLID-2192-undetermined-insurance`
- **Base Branch**: `develop` (cut from `develop`, not from `feature/MLID-2194-pharmacy-medicare-supplement` — these are independent fixes reviewed separately)
- **Status**: Done — committed `a5427f02`, awaiting PR
- **Epic plan**: None — MLID-2192 is a sub-task of MLID-2097 ("Random Tasks"), a bucket epic with no architectural plan. Treat as standalone.
- **PR doc**: `docs/agomez/PR/MLID-2192.md`

---

## Summary

The `patient_insurances` lookup in `getOrdersAggregate` filters by `status = 'Active'` only, which silently drops insurance records whose status is `'Undetermined'`. This means patients with Undetermined insurance show no primary insurance in the New Orders insurance column and are incorrectly evaluated as ineligible in pharmacy and 340B computations for both New and Maintenance Orders. The fix is a single filter clause change — `{ $eq: ['$status', 'Active'] }` → `{ $in: ['$status', ['Active', 'Undetermined']] }` — in the one shared lookup that feeds all three downstream consumers. All other lookup constraints (priority `'1'`, `$limit: 1`) are preserved unchanged.

---

## Codebase Analysis

- **Fix location**: `apps/web/services/mongodb/ordersTracker.ts`, line 372, inside the `$lookup` against `COLLECTION.PatientInsurances` in `getOrdersAggregate()`.
- **Shared lookup**: This one `$lookup` is executed for both `category === 'new'` and `category === 'maintenance'` — fixing it here covers all three downstream consumers in a single edit:
  1. "Insurance" display column in New Orders UI (`apps/web/app/orders-tracker/Orders/NewOrders.tsx`), gated by `ORDERS_TRACKER_IG_COLUMNS` flag.
  2. Pharmacy eligibility `$switch` in `getEligibilityPipelineStages()` (same file, lines 25–224), gated by `ORDER_ELIGIBILITY` flag.
  3. 340B eligibility `$switch` in the same function, applied to both `new` and `maintenance` categories.
- **Canonical status enum**: WeInfuse OpenAPI spec (`apps/web/apis/weinfuse/openapi.json`, line 898) lists `["Active", "Inactive", "Undetermined"]`. TypeScript union `'Active' | 'Inactive' | 'Undetermined'` is already used in `apps/web/types/patientOrderReview.ts:107`, `apps/web/components/Modules/document-intake/schemas/types.ts:148`, and `apps/web/services/intakes/intakeAiService.ts:93`. "Undetermined" is a real, canonical status — not a typo or transitional value.
- **No schema constraint**: `apps/web/models/PatientInsurance.ts` declares `status: { type: String }` with no enum restriction, so the model accepts any string value.
- **Feature flags**: `ORDER_ELIGIBILITY` and `ORDERS_TRACKER_IG_COLUMNS` are already deployed and wired at both API and UI levels. No flag changes are needed — this is a bug fix, not a new feature.
- **No other status-filtering lookups**: The only place in `apps/web/` that filters `patient_insurances` by status is this one `$lookup`. Other consumers (`patient.ts`, `patientInsurance.ts`, `abbvieDailyStatusReport.ts`) do not filter by status. No spillover changes required.
- **Maintenance Orders UI**: The Maintenance Orders view does not have a "Primary Insurance" display column. The Jira reference to "Maintenance Orders → Primary insurance" means the lookup data feeds 340B eligibility computations for maintenance orders, which already work via the same lookup. Adding a UI column for Maintenance Orders is out of scope for this ticket.
- **Existing tests**: `apps/web/services/mongodb/__tests__/ordersTracker.test.ts` has no assertion on the inner status filter value. The `with includeEligibility` describe block (lines 596–928) checks for stage presence and field names but not the `$status` clause content. No existing test will fail from this change — which is why a new RED-phase test is required.

---

## Implementation Steps

### Step 1 (RED) — Add a failing pipeline-structure test

**File**: `apps/web/services/mongodb/__tests__/ordersTracker.test.ts`

Add a new `describe` block immediately after the closing `}` of `'pharmacyEligible Medicare regex anchoring'` (line 927) and before the closing `}` of `'with includeEligibility'` (line 928). The new block extracts the `patient_insurances` lookup from the executed pipeline and asserts that its inner `$match.$expr.$and` contains the `$in` clause — and does not contain the old `$eq` clause.

Insert the following after line 927:

```typescript
describe('patient_insurances lookup status filter', () => {
  async function getInsuranceLookupAndConditions() {
    const mockAggregate = jest.fn().mockReturnValue({
      toArray: jest.fn().mockResolvedValue([]),
    });
    const mockCollection = jest.fn().mockReturnValue({
      aggregate: mockAggregate,
    });
    mockedConnectToDatabase.mockResolvedValue({
      db: { collection: mockCollection } as unknown as Db,
      client: {} as never,
    });

    const { getOrders } = await import('../ordersTracker');
    await getOrders({ category: 'new' });

    const pipeline: Record<string, unknown>[] = mockAggregate.mock.calls[0][0];

    const insuranceLookup = pipeline.find(
      (s) =>
        '$lookup' in s &&
        s.$lookup &&
        (s.$lookup as Record<string, unknown>).from === 'patient_insurances'
    ) as {
      $lookup: {
        pipeline: {
          $match: {
            $expr: {
              $and: unknown[];
            };
          };
        }[];
      };
    };

    return insuranceLookup.$lookup.pipeline[0].$match.$expr.$and;
  }

  it('should filter patient_insurances by $in ["Active", "Undetermined"] — not $eq "Active"', async () => {
    const andConditions = await getInsuranceLookupAndConditions();

    const hasInClause = andConditions.some(
      (c) =>
        typeof c === 'object' &&
        c !== null &&
        '$in' in c &&
        JSON.stringify((c as Record<string, unknown>).$in) ===
          JSON.stringify(['$status', ['Active', 'Undetermined']])
    );
    expect(hasInClause).toBe(true);

    const hasOldEqClause = andConditions.some(
      (c) =>
        typeof c === 'object' &&
        c !== null &&
        '$eq' in c &&
        JSON.stringify((c as Record<string, unknown>).$eq) ===
          JSON.stringify(['$status', 'Active'])
    );
    expect(hasOldEqClause).toBe(false);
  });
});
```

Run from `apps/web/`:

```bash
npm run test -- ordersTracker
```

Expected: 1 failure — `'should filter patient_insurances by $in ["Active", "Undetermined"] — not $eq "Active"'` fails because the production code still uses `$eq`.

---

### Step 2 (GREEN) — Widen the status filter

**File**: `apps/web/services/mongodb/ordersTracker.ts`

**Edit line 372:**

Before:

```typescript
              { $eq: ['$status', 'Active'] },
```

After:

```typescript
              { $in: ['$status', ['Active', 'Undetermined']] },
```

Run from `apps/web/`:

```bash
npm run test -- ordersTracker
```

Expected: all tests green.

---

### Step 3 (REFACTOR) — Verify quality gates

No refactoring needed — the change is a single clause swap with no structural complexity. Run from `apps/web/`:

```bash
npm run types:check
npm run lint:fix
```

Expected: clean. No manual browser verification is required — this is a pure backend aggregation change. Active-status patients are unaffected (behavior identical). Undetermined-status patients now flow through the same existing display and eligibility logic that Active-status patients already use; no new UI paths are introduced.

---

## Files Affected

| File                                                        | Action | Description                                                                                                                                  |
| ----------------------------------------------------------- | ------ | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `apps/web/services/mongodb/ordersTracker.ts`                | Modify | Change line 372 from `$eq: ['$status', 'Active']` to `$in: ['$status', ['Active', 'Undetermined']]`                                          |
| `apps/web/services/mongodb/__tests__/ordersTracker.test.ts` | Modify | Add new `describe("patient_insurances lookup status filter")` block asserting the `$in` clause is present and the old `$eq` clause is absent |

---

## Testing Strategy

**New test (RED-phase, must fail before Step 2):**

- `'patient_insurances lookup status filter > should filter patient_insurances by $in ["Active", "Undetermined"] — not $eq "Active"'` — asserts the `$and` array contains `{ $in: ['$status', ['Active', 'Undetermined']] }` and does not contain `{ $eq: ['$status', 'Active'] }`.

**Existing tests that must remain green (no changes to these):**

- All `'with includeEligibility'` pipeline-shape tests (lines 607–781) — check stage presence by `from`/field name, not the status clause. Unaffected.
- `'pharmacyEligible $switch treatment gate'` describe block (lines 783–839) — checks branch structure and `$ifNull` default. Unaffected.
- `'pharmacyEligible Medicare regex anchoring'` describe block (lines 841–927) — checks regex content for the Medicare allow-list. Unaffected.
- `apps/web/app/orders-tracker/Orders/MaintenanceOrders.test.tsx` — does not assert insurance status. Unaffected.

**No integration test or manual browser verification needed.** The fix changes which insurance records are fetched; the display and eligibility logic that consumes those records is unchanged. The only observable difference is that Undetermined-status patients now show their insurance and are evaluated correctly — which is the desired behavior stated in the ticket.

---

## Security Considerations

None. This is a read-only aggregation pipeline change. No new input surfaces are introduced. No PHI is written or newly exposed — Undetermined-status patients will now see the same already-rendered insurance plan name display that Active-status patients already see, which is correct behavior per the ticket. The fix is behind the already-deployed `ORDER_ELIGIBILITY` and `ORDERS_TRACKER_IG_COLUMNS` feature flags with no flag wiring changes.

---

## Open Questions

None.

---

## Confirmed: 340B coverage verified (2026-05-06)

Product Owner asked whether the same `Undetermined`-status acceptance should be applied to the 340B eligibility column. Investigation confirmed it already is — no further code change required.

**Why:** The patient_insurances `$lookup` in `getOrdersAggregate` (`apps/web/services/mongodb/ordersTracker.ts:362–386`) is the **single shared source** of `$patient.primaryInsurance` for both `$switch` blocks downstream:

- 340B `$switch` (`hospital340BEligible`, lines 60–103) reads `$patient.primaryInsurance` and `$patient.primaryInsurance.payer_type_name`.
- Pharmacy `$switch` (`pharmacyEligible`, lines 127–217) reads the same field.

Grepping for `patient_insurances` / `PatientInsurances` across `ordersTracker.ts` returns exactly one match — the lookup at line 362. So the line 372 widening from `{ $eq: ['$status', 'Active'] }` to `{ $in: ['$status', ['Active', 'Undetermined']] }` covers all three downstream consumers in a single edit, exactly as the original Codebase Analysis section predicted.

**Empirical confirmation:** During testing of order `69fa76b753008d6c2de629c2` (Jane Barker, VA Commercial, Undetermined), reassigning the order forced the cache to recompute. The `pharmacyEligible` value flipped from `"Waiting on Insurance Input"` to `"Yes"` — proving the same lookup is feeding both `$switch` blocks the now-visible Undetermined insurance row. (340B for that order resolved to `"No"` for unrelated reasons — provider NPI not matched in `hospitalSystems`.)

**Defensive test added** in branch `fix/MLID-2192-340b-test-coverage`: a new `describe` block in `apps/web/services/mongodb/__tests__/ordersTracker.test.ts` — `'hospital340BEligible $switch reads from patient.primaryInsurance'` — locks down the contract that 340B's `$switch` references `$patient.primaryInsurance.payer_type_name`. If a future refactor disconnects 340B from the shared lookup, the test fails immediately.

**No production code change.** Pure test addition + this plan annotation.
