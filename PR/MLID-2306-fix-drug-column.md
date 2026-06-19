# [MLID-2306] fix(prior-auth): remove genericDrugName from drug column display

## Summary
- The Drug column in the Auth Review table (`/orders-tracker/new/[id]/auth-review`) was falling back to `ocrData.service.genericDrugName.value` when neither the verified drug name nor the brand drug name was available.
- The PO requested that the generic drug name be removed from the display logic. Documents that have only a generic drug name now show an empty cell (rendered as "—" by the UI) instead of the generic name.

## Root Cause

The `transformIntakeToPriorAuthLetter` function in `services/priorAuth/transforms.ts` used a three-level fallback chain for the `drug` display field:

```
priorAuthData.drugName → ocrData.brandDrugName → ocrData.genericDrugName → ''
```

The generic drug name fallback was intentional at the time of implementation (to maximize visible data), but the PO determined it produces misleading output: the generic name is OCR-extracted, often noisy, and unrelated to the drug specifically selected for this order.

## Changes Overview
- **Files changed**: 2
- **Lines added**: +2
- **Lines removed**: -3

## Files Changed

| File | Change | Description |
|------|--------|-------------|
| `apps/web/services/priorAuth/transforms.ts` | Modified | Removed `ocrData?.service?.genericDrugName?.value` from the `drug` field fallback chain |
| `apps/web/services/priorAuth/transforms.test.ts` | Modified | Updated the existing test that asserted the old generic-fallback behaviour; renamed it to reflect the new expectation |

## What Changed and Why

### `transforms.ts` — drug field fallback chain

**Before:**
```ts
drug:
  resolveVerifiedFieldValue(priorAuthData?.drugName) ||
  ocrData?.service?.brandDrugName?.value ||
  ocrData?.service?.genericDrugName?.value ||
  '',
```

**After:**
```ts
drug:
  resolveVerifiedFieldValue(priorAuthData?.drugName) ||
  ocrData?.service?.brandDrugName?.value ||
  '',
```

The new chain is:
1. `priorAuthData.drugName` — human-verified LISA canonical name (set when reviewer completes Step 2 of the wizard)
2. `ocrData.service.brandDrugName.value` — OCR-extracted brand name
3. `''` — empty (UI renders as "—")

**Note:** this function is shared between the order Auth Review table and the patient prior-auth view (`/patient/[id]/prior-auth`). The generic name fallback is dropped in both surfaces.

### `transforms.test.ts` — updated test

The test `'should use genericDrugName when brandDrugName is absent'` previously expected `'adalimumab'` (the generic name). It now expects `''` and is renamed to `'should return empty string when only genericDrugName is present and brandDrugName is absent'` to accurately describe the new behaviour.

## Commits

| Hash | Message |
|------|---------|
| `8d2d7b787` | [MLID-2306] - fix(prior-auth): remove genericDrugName from drug column display fallback |

## Test Plan
- [ ] Open an order in the Auth Review table that has a document with only a generic drug name and no brand drug name — verify the Drug column shows "—" instead of the generic name
- [ ] Open an order with a document that has a brand drug name — verify it still displays correctly
- [ ] Open an order with a reviewed document (verified drug name set) — verify the verified name still takes priority
- [ ] Navigate to a patient's prior-auth view (`/patient/[id]/prior-auth`) — verify the Drug column behaviour is consistent with the above

## Jira
- [MLID-2306](https://localinfusion.atlassian.net/browse/MLID-2306)
