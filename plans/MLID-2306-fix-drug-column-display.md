# MLID-2306 — Fix: Remove Generic Drug Name from Drug Column Display

## Task Reference

- **Jira**: MLID-2306 (post-merge fix)
- **Branch**: `fix/MLID-2306-drug-column-drop-generic-name` off `develop`
- **Status**: Done — commit `8d2d7b787`, PR doc at `docs/agomez/PR/MLID-2306-fix-drug-column.md`

---

## Change Description

The Drug column in the Auth Review table (`/orders-tracker/new/[id]/auth-review`)
currently falls back to the OCR-extracted generic drug name when neither the
verified drug name nor the brand drug name is available. The PO wants the generic
drug name removed from this fallback chain.

**Current fallback chain** (`apps/web/services/priorAuth/transforms.ts:95–99`):

```ts
drug:
  resolveVerifiedFieldValue(priorAuthData?.drugName) ||
  ocrData?.service?.brandDrugName?.value ||
  ocrData?.service?.genericDrugName?.value ||    // ← remove this line
  '',
```

**Target fallback chain:**

```ts
drug:
  resolveVerifiedFieldValue(priorAuthData?.drugName) ||
  ocrData?.service?.brandDrugName?.value ||
  '',
```

---

## Files Affected

| File | Action |
|------|--------|
| `apps/web/services/priorAuth/transforms.ts` | Remove `genericDrugName` fallback from the `drug` field (line 98) |
| `apps/web/services/priorAuth/__tests__/transforms.test.ts` | Create — new test file covering the `drug` field fallback chain |

---

## Implementation Steps

### Step 1 — RED: write failing test

**Create** `apps/web/services/priorAuth/__tests__/transforms.test.ts`.

Test cases for the `drug` field in `transformIntakeToPriorAuthLetter`:

```ts
describe('transformIntakeToPriorAuthLetter — drug field', () => {
  it('should use priorAuthData.drugName when present', () => {
    // intake with priorAuthData.drugName = 'Privigen'
    // brandDrugName and genericDrugName also present
    // expect result.drug === 'Privigen'
  });

  it('should fall back to brandDrugName when drugName is absent', () => {
    // intake with no priorAuthData.drugName
    // brandDrugName = 'Privigen (Brand)'
    // genericDrugName = 'IVIG Generic'
    // expect result.drug === 'Privigen (Brand)'
  });

  it('should return empty string when only genericDrugName is present', () => {
    // intake with no priorAuthData.drugName, no brandDrugName
    // genericDrugName = 'IVIG Generic'
    // expect result.drug === ''
  });

  it('should return empty string when all drug fields are absent', () => {
    // intake with no priorAuthData.drugName, no brandDrugName, no genericDrugName
    // expect result.drug === ''
  });
});
```

Run: `npm run test -- transforms.test.ts`
Expected: third test case fails (current code returns `genericDrugName` instead of `''`).

### Step 2 — GREEN: remove the generic name fallback

**Modify** `apps/web/services/priorAuth/transforms.ts`, line 98.

Remove the `ocrData?.service?.genericDrugName?.value ||` line.

Run: `npm run test -- transforms.test.ts`
Expected: all four test cases pass.

### Step 3 — REFACTOR: verify no regressions

```bash
npm run test -- --testPathPattern="priorAuth"
npm run types:check
npm run lint:fix
```

All existing prior-auth tests must still pass.

---

## Scope Notes

- This change only affects the display value for the Drug column in the table.
- The drug-scope filter (`buildOrderScopedPriorAuthMatch`) is independent — it
  matches against both `brandDrugName` and `genericDrugName` for filtering
  purposes and is not changed.
- The `VerifyDataForm` and the `liDrugId` field are not touched.
- The patient prior-auth view (`/patient/[id]/prior-auth`) uses the same
  `transformIntakeToPriorAuthLetter` function, so the Drug column there will
  also drop the generic name fallback — confirm this is acceptable with the PO.
