# [MLID-2307] fix(financial-review): require all calculator fields before save

## Summary

QA reported that the Order Details **Financial Review** calculator
(`/orders-tracker/[category]/[id]/financial-review/new`) let a user save a new
calculation with empty fields. The legacy patient calculator
(`/patient/[id]/financial-calculator`) does not have this problem — every field
there is required. This hotfix ports that all-fields-required validation onto the
order surface so the two calculators behave the same way.

## Root cause

In the order calculator, empty inputs were coerced to `0` (`parseFloat(value) || 0`)
and the only guard before saving was `hasAnyInput` — true if **at least one** field
was non-empty. A user could therefore fill a single field and persist a calculation
with the other eight blank (saved as zeros).

The patient calculator uses `react-hook-form` with a `required` rule on all nine
fields, blocking submission and showing a per-field "This field is required" message.

## Changes

**`Calculator.tsx`**
- All nine inputs are now required, reusing `VALIDATION_MSG.isRequired` (the same
  constant the patient form uses).
- Validation runs per field on blur (mirroring the patient form's `mode: "onBlur"`)
  and on the full form when Save is clicked. Save no longer calls the persistence
  hook or navigates while any field is empty.
- Each empty field shows a "This field is required" message via the `@repo/ui`
  `TextField` `error` / `errorMessage` props; the message clears as soon as the
  field is filled.
- Fields are marked `required` (asterisk). The Save button is now always enabled
  (except while saving) so clicking it surfaces the validation errors, replacing
  the previous `hasAnyInput` disable gate.
- Positive-only input masking is unchanged — confirmed intended behavior. Negative
  signs are still stripped on input the same way they were before this change.

**`mocks/repo-ui.tsx`**
- The shared `TextField` test mock now forwards `onBlur` / `required` and renders
  `errorMessage`, so validation state is assertable in tests. Additive change only.

**`Calculator.test.tsx`**
- Existing save-flow tests updated to fill all fields.
- New coverage: empty-form save is blocked and shows all required errors,
  partial-fill save is blocked, blur produces a required error, and the error
  clears once the field is filled.

## Testing

- `Calculator.test.tsx`: 15/15 pass.
- Full `orders-tracker` Jest suite (covers every consumer of the modified
  `repo-ui` mock): 53 suites / 738 tests pass — no regressions.
- `tsc --noEmit`: clean.
- ESLint + Prettier: clean.

## Manual QA

1. Enable the `ORDER_FINANCIAL_CALCULATION` feature flag in `/admin/feature-flags`.
2. Open any order, go to **Financial Review → New Calculation**.
3. Click **Save calculation** with the form empty → save is blocked and every field
   shows "This field is required".
4. Fill only some fields and Save → still blocked; only the empty fields show the error.
5. Tab out of (blur) an empty field → that field shows the required error; typing a
   value clears it.
6. Fill all nine fields and Save → the calculation is saved and the view returns to
   the list.
7. Confirm negative input is rejected: typing `-20` shows `20` (unchanged masking).

## Notes

- Scope is limited to the new-calculation form on the order Financial Review surface.
- No data-layer, schema, or API changes — this is a client-side validation fix.
- Feature flag default remains `false`.
