# MLID-2015 — Exclude Treatments from Pharmacy Eligibility

## Task Reference

- **Jira**: [MLID-2015](https://localinfusion.atlassian.net/browse/MLID-2015)
- **Story Points**: 2
- **Branch**: `feature/MLID-2015-pharmacy-exclude-treatments`
- **Base Branch**: `epic/MLID-1492-order-eligibility`
- **Parent Epic**: [MLID-1492](https://localinfusion.atlassian.net/browse/MLID-1492)
- **Status**: In QA

---

## Summary

Orders whose drug is a treatment (`lidrugs.is_treatment == true`) should never be pharmacy eligible. Add a new `$switch` branch to the `pharmacyEligible` computation in `getEligibilityPipelineStages()` that short-circuits to `"No"` when the drug is a treatment. No new lookups, no type changes, no UI changes.

---

## Codebase Analysis

- **Pipeline location**: `getEligibilityPipelineStages()` in `ordersTracker.ts` (lines 335–527). The `pharmacyEligible` `$addFields` stage is Stage 5 (lines 434–521).
- **Current `$switch` branches**: (1) No primary insurance → "Waiting on Insurance Input", (2) All conditions met → "Yes", (3) Default → "No".
- **Field access**: At this pipeline stage, `$drug` is the expanded `LIDrug` document (set at lines 754–758). `$drug.is_treatment` is valid. Need `$ifNull` guard because `drug` can remain a raw string when `drugObjectId` is null — same pattern used for `$drug.pharmacyEligible`.
- **LiDrug model**: `is_treatment?: boolean` (interface line 41), `is_treatment: { type: Boolean, default: false }` (schema line 109).
- **Existing tests**: Tests at lines 1884–1967 only check that `$addFields` with key `pharmacyEligible` exists — no branch-level assertions yet.
- **No type changes needed**: `pharmacyEligible?: K extends 'new' ? string : never` already covers the output.

---

## Implementation Steps

### Step 1 — Add `is_treatment` branch to pharmacy `$switch`

- **File**: `apps/web/services/mongodb/ordersTracker.ts`
- **What**: Insert a new branch at index 1 in the `pharmacyEligible` `$switch.branches` array (after the insurance-missing check, before the Yes computation):

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

**Why `$ifNull` with `false`:** The `drug` field may remain a raw string (unresolved ObjectId) when `drugObjectId` is null, so `$drug.is_treatment` would be missing. Defaulting to `false` means unresolved drugs are not treated as treatments, preserving existing behavior.

### Step 2 — Add test cases

- **File**: `apps/web/services/mongodb/__tests__/ordersTracker.test.ts`
- **What**: Add two tests to the existing pharmacy eligibility `describe` block.

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/services/mongodb/ordersTracker.ts` | Modify | Add `is_treatment` branch to pharmacy `$switch` |
| `apps/web/services/mongodb/__tests__/ordersTracker.test.ts` | Modify | Add test cases for treatment short-circuit |

---

## Testing Strategy

### TDD Plan

#### RED: Write Failing Tests

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

Insert the `$switch` branch in `getEligibilityPipelineStages()` at the exact position described in Step 1.

#### REFACTOR

No refactoring expected — the change is a single array element insertion. Verify lint and type-check pass.

---

## Security Considerations

None. This is a read-only computation in an existing aggregation pipeline behind the `ORDER_ELIGIBILITY` feature flag. No new endpoints, no new inputs, no PHI implications.

---

## Open Questions

None.
