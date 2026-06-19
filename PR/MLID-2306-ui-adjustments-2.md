# [MLID-2306] fix(auth-review): UI adjustments round 2

## Summary

- Replace plain `<a>` tag with `@repo/ui` `Link` + `OpenInNewOutlinedIcon` for the "View Fax in Spruce" link (removes default purple color, matches mock)
- Fix `navButtons` layout to `justify-content: space-between` so back and forward buttons spread across the full width on all steps
- Add Cancel button on step 1 (multi-step letters) that navigates back to the auth review list
- Show "Letter type" pill in the PDF preview panel header on steps 2 and 3
- Add icons to nav buttons: `ArrowForwardIcon` on Verify Data / Generate Note, `ArrowBackIcon` on back buttons (steps 2 and 3), `CheckIcon` on Finish; back buttons on steps 2/3 changed to `variant="text"` to match mock

## Test plan

- [ ] Open an order with prior-auth documents and navigate to the Auth Review tab
- [ ] Open a multi-step letter (e.g. Approval or Denial)
- [ ] Step 1: confirm Cancel button is on the left, "Verify Data →" on the right; clicking Cancel returns to the list
- [ ] Step 1: confirm "View Fax in Spruce" link uses the correct color and shows the open-in-new icon
- [ ] Advance to step 2: confirm "← Back to Letter Type" is text-style on the left, "Generate Note →" on the right; confirm "Letter type: [X]" pill appears in the PDF header
- [ ] Advance to step 3: confirm "← Back to Edit Fields" on the left, "✓ Finish & Mark Reviewed" on the right; confirm pill still shows
- [ ] Open a single-step letter (e.g. Receipt): confirm only the Finish button with CheckIcon appears
