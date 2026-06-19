# [MLID-2752] fix(orders-financial-calculation): run admin cost before drug cost

## Summary

Aligns the orders-tracker financial review calculation to match the corrected patient section
behavior. Alberto (MLID-2056, PR #1571, commits 5b7f584 / 07a2079 / 7cbd73c) flipped the leg
processing order on the patient side to Admin-first in May 2026. The orders-tracker side still
runs Drug-first, producing different per-leg results when the deductible or OOP cap is not
already exhausted.

## Jira

- Task: MLID-2752
- Related fix on patient side: MLID-2056 (PR #1571)

## Branch

`fix/MLID-2752-admin-before-drug-calculation-order` off `develop`

## Context

### Why Admin-first is correct

Both legs share a running `remainingDeductible` and `remainingOOP` state. The first leg burns
those credits before the second leg gets them. Alberto confirmed that Admin cost should run first
so the deductible/OOP gaps are consumed by the smaller, recurring infusion fee before the
(typically larger) drug cost is applied. This is the behavior already live in the patient section.

### What changes — one file

`apps/web/services/ordersFinancialCalculation/ordersFinancialCalculation.ts`

In `runOrdersFinancialCalculation` (lines 198–243):

**Before (Drug-first — current state):**
```
drugRun  = process('Drug', drugCost, ..., initialState)
adminRun = (OOP exhausted?) → skip bullet | process('Admin', adminCost, ..., drugRun.newState)
drugResp  = drugRun.patientPays
adminResp = adminRun.patientPays
summary:      Drug Responsibility first, Admin Responsibility second
skip-message: "Skipped calculation because OOP max was reached during Drug."
```

**After (Admin-first — target state matching patient section):**
```
adminRun = process('Admin', adminCost, ..., initialState)
drugRun  = (OOP exhausted?) → skip bullet | process('Drug', drugCost, ..., adminRun.newState)
adminResp = adminRun.patientPays
drugResp  = drugRun.patientPays
summary:      Admin Responsibility first, Drug Responsibility second
skip-message: "Skipped calculation because OOP max was reached during Admin."
```

### Test impact

One existing test becomes semantically stale after the flip:

- `"should skip the Admin leg entirely when remainingOOP hits zero during Drug"` (line 207 in
  `ordersFinancialCalculation.test.ts`). It uses `drugCost: 500, adminCost: 200, oopMax: 50` and
  expects `drugResp = 50, adminResp = 0, breakdown.admin contains "Skipped"`. After the fix Admin
  runs first; Admin consumes the OOP and Drug is the one skipped.

All other existing tests remain green because their `adminCost` defaults to `0`, so the Admin leg
produces no deductible/OOP change and all downstream Drug assertions still hold.

---

## Implementation Steps

### Step 1 — RED: write a failing test for Admin-first OOP-skip behavior

In `apps/web/services/ordersFinancialCalculation/ordersFinancialCalculation.test.ts`, inside the
`OOP cap` describe block, add a new test **before touching any implementation**:

```ts
it('should skip the Drug leg when remainingOOP hits zero during Admin', () => {
  const { result, breakdown } = runOrdersFinancialCalculation(
    makeInputs({
      deductibleMax: 0,
      coinsurance: 100,
      oopMax: 50,
      oopMet: 0,
      adminCost: 500,
      drugCost: 200,
    })
  );

  expect(result.adminResp).toBe(50);
  expect(result.drugResp).toBe(0);
  expect(breakdown.drug.join(' ')).toMatch(/Skipped/i);
});
```

Run to confirm RED:
```
cd apps/web && npx jest --testPathPattern="ordersFinancialCalculation.test" --no-coverage
```
Expected: the new test fails — current code returns `drugResp: 50, adminResp: 0`.

---

### Step 2 — GREEN: flip the leg order in runOrdersFinancialCalculation

In `runOrdersFinancialCalculation` (lines 198–243 of `ordersFinancialCalculation.ts`), apply all
four changes below in one edit:

1. **Run Admin first** — move `processOrdersFinancialCalculationPart('Admin', ...)` to the first
   call, passing `initialRemainingDeductible` and `initialRemainingOOP`.
2. **Check Admin's OOP for the skip condition** — `adminRun.newState.remainingOOP === 0` triggers
   the skip path (not Drug's state).
3. **Run Drug second (or set skip bullet)** — pass `adminRun.newState` as the starting state.
   Update the skip bullet text to:
   `'Skipped calculation because OOP max was reached during Admin. Patient owes $0.'`
4. **Update variable assignments and summary order:**
   - `adminResp = adminRun.patientPays`
   - `drugResp  = drugRun.patientPays`
   - `breakdown.summary`: Admin Responsibility line first, Drug Responsibility line second.

Run to confirm GREEN:
```
cd apps/web && npx jest --testPathPattern="ordersFinancialCalculation.test" --no-coverage
```
Expected: the new test passes; all pre-existing tests also pass.

---

### Step 3 — REFACTOR: update the stale OOP-skip test

Replace the test at line 207 (`"should skip the Admin leg entirely when remainingOOP hits zero
during Drug"`) to reflect Admin-first semantics:

- **Description:** `'should skip the Drug leg entirely when remainingOOP hits zero during Admin'`
- **makeInputs:** swap the cost values — `adminCost: 500, drugCost: 200` (same numbers, different
  fields) so it is Admin that exhausts OOP.
- **Assertions:**
  - `expect(result.adminResp).toBe(50)` (was `result.drugResp`)
  - `expect(result.drugResp).toBe(0)` (was `result.adminResp`)
  - `expect(breakdown.drug.join(' ')).toMatch(/Skipped/i)` (was `breakdown.admin`)

Run full test suite one final time:
```
cd apps/web && npx jest --testPathPattern="ordersFinancialCalculation.test" --no-coverage
```
Expected: all tests green, no stale description remaining.

---

## Acceptance Criteria

- [ ] `runOrdersFinancialCalculation` processes the Admin leg before the Drug leg
- [ ] The skip bullet reads "OOP max was reached during Admin"
- [ ] `breakdown.summary` lists Admin Responsibility before Drug Responsibility
- [ ] All tests in `ordersFinancialCalculation.test.ts` pass
- [ ] `npm run types:check` passes (no type changes expected)
- [ ] `npm run lint` passes (no lint changes expected)

## Resume State

Done. Commit d21a1f1d6 on fix/MLID-2752-admin-before-drug-calculation-order.
