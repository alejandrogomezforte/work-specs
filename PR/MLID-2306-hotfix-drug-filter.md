# [MLID-2306] fix(prior-auth): revise order-scoped drug filter to direct mapping and add not-enough-evidence passthrough

## Summary

- Replaced the cross-comparison fuzzy logic (all order terms matched against all intake drug fields) with a direct mapping: `brandDrugName` compared against `lidrugs.name` (brand-to-brand), `genericDrugName` compared against `lidrugs.genericName` (generic-to-generic).
- Added a **not-enough-evidence passthrough**: when the intake has no `brandDrugName` and no `liDrugId`, and the order has no `genericName` (making neither comparison possible), the document is shown rather than hidden — there is no basis to exclude it.
- Updated unit tests to cover the direct-mapping behaviour and the new passthrough case.

## Root Cause

The original fuzzy logic performed a cross-comparison: both order terms (brand name and generic name) were tested against both intake fields (`brandDrugName` and `genericDrugName`). This produced 4 clauses that could match across drug name types.

The problem emerged for a specific combination: an intake that has only `genericDrugName` (no `brandDrugName`, no `liDrugId`) paired with an order whose `lidrugs` record has no `genericName`. Under the old logic, the cross-comparison generates clauses that test the order's brand name against the intake's `genericDrugName` — e.g. `"gammaked"` in `"vedolizumab"` — which fails. The old passthrough only fires when the intake has **both** name fields empty, so it did not help here. The result was the document being hidden with no mechanism to surface it.

The fix introduces a "not-enough-evidence" rule: if neither a brand-to-brand nor a generic-to-generic comparison can be made, the document is shown by default.

## Changes Overview

- **Files changed**: 2
- **Lines added**: +69
- **Lines removed**: -35

## Files Changed

| File | Change | Description |
|------|--------|-------------|
| `apps/web/services/priorAuth/query.ts` | Modified | Replace cross-comparison fuzzy logic with direct brand-to-brand and generic-to-generic mapping; add not-enough-evidence passthrough clause |
| `apps/web/services/priorAuth/__tests__/query.orderScoped.test.ts` | Modified | Update tests to verify direct-mapping behaviour and both passthrough paths |

## What Changed and Why

### `query.ts` — `buildOrderScopedPriorAuthMatch`

**Before:** `orderTerms` was built from both order name fields and then `.flatMap`-ed against both intake fields, producing up to 4 fuzzy clauses:

```
brand order term → brandDrugName
brand order term → genericDrugName   ← cross (wrong)
generic order term → brandDrugName   ← cross (wrong)
generic order term → genericDrugName
```

**After:** each order name field is tested only against its matching intake field:

```
orderDrugName    → brandDrugName      (brand-to-brand)
orderDrugGenericName → genericDrugName (generic-to-generic)
```

When `orderDrugGenericName` is empty (the order's drug has no generic name), the generic-vs-generic clause is replaced by a **not-enough-evidence passthrough**: an additional check that `brandDrugName` is also absent on the intake. If both sides of the only available comparison are missing, there is nothing meaningful to compare and the document is shown.

**Decision:** the not-enough-evidence passthrough is intentionally conservative — it only fires when evidence on both sides is absent. If either the intake's brand name or the order's generic name is present, the normal comparison path runs and can still exclude the document.

### `query.orderScoped.test.ts`

Three test descriptions and expectations were updated:

- `'fuzzy-matches brand-to-brand and generic-to-generic when both order names are set'` — expects 3 clauses (passthrough + brand fuzzy + generic fuzzy), no cross terms.
- `'replaces the generic fuzzy clause with a not-enough-evidence passthrough when the order has no generic name'` — expects 3 clauses (passthrough + brand fuzzy + brand-absent check).
- `'keeps only the passthrough when the order has no name at all'` — expects 2 clauses (complete passthrough + not-enough-evidence passthrough).

All 9 tests pass.

## Commits

| Hash | Message |
|------|---------|
| `44f0a8c` | [MLID-2306] - fix(prior-auth): revise order-scoped drug filter to direct mapping and add not-enough-evidence passthrough |

## Test Plan

- [ ] Navigate to order `6a2fdc92ff38201a9d83fe75` → Prior Auth tab → verify the vedolizumab document (`6972643d21a76dbc70ff5bdc`) appears (Cases 2 and 3 from postmortem)
- [ ] Confirm a total of 2 documents appear for that order (vedolizumab + OCRELIZUMAB)
- [ ] Navigate to an order where `lidrugs.genericName` is populated and verify only matching intake documents appear (drug filter still rejects non-matching intakes)
- [ ] Run unit tests: `npm run test -- query.orderScoped`

## Jira

- [MLID-2306](https://localinfusion.atlassian.net/browse/MLID-2306)
