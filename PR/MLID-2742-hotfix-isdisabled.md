# [MLID-2742] fix(admin-integrations): correct `isDisabled` → `disabled` on StyledInput SFTP fields

> Follow-up hotfix to the MLID-2742 epic (the main epic PR is separate). Reuses the MLID-2742 ticket; no new ticket. Branches off `develop`, targets `develop`.

## Summary

- `StyledInput` spreads its props straight onto a native `<input>`. Several admin integration pages passed `isDisabled={...}`, which is **not** a valid HTML input attribute — so the field never actually disabled. The prop landed on the DOM as an unrecognized attribute (React logged a warning), and the SFTP / API-key inputs stayed editable while a save or test-connection request was in flight.
- Fixes the correct attribute (`isDisabled` → `disabled`) on the 14 `StyledInput` call sites across five admin integration pages.
- This is "bug family 1" of the `StyledInput` cleanup surfaced during MLID-2769 (the same class of bug already fixed on AbbVie and Kedrion inside the epic). "Bug family 2" is deferred — see below.

## Root Cause

`StyledInput` is a thin wrapper that spreads whatever props it receives onto a native input:

```tsx
<input {...inputProps} ... />
```

Its type is permissive (an index signature `[prop: string]: unknown`), so TypeScript did not catch that `isDisabled` is not a real input attribute. The valid attribute is `disabled`. Passing `isDisabled` therefore did nothing — the field stayed enabled, and React emitted an "unrecognized attribute" warning.

The `Button` components on the same pages have their own `isDisabled` prop (a valid custom-`Button` prop) and are correct — they were left untouched.

## Changes Overview

- **Files changed**: 6
- **Lines added**: +23
- **Lines removed**: -23

## Files Changed

| File | Change | Description |
|------|--------|-------------|
| `apps/web/app/admin/integrations/alnylam/page.tsx` | Modified | 5 `StyledInput` fields: `isDisabled` → `disabled` |
| `apps/web/app/admin/integrations/argenx/page.tsx` | Modified | 5 `StyledInput` fields: `isDisabled` → `disabled` |
| `apps/web/app/admin/integrations/spruce/page.tsx` | Modified | 2 `StyledInput` fields: `isDisabled` → `disabled` |
| `apps/web/app/admin/integrations/jotform/page.tsx` | Modified | 1 `StyledInput` field: `isDisabled` → `disabled` |
| `apps/web/app/admin/integrations/notion/page.tsx` | Modified | 1 `StyledInput` field: `isDisabled` → `disabled` |
| `apps/web/components/UI/StyledInput/StyledInput.tsx` | Modified | Comment-only: updated to note the `isDisabled` family is fixed; permissive type retained for the remaining `error`-prop family |

## What Changed and Why

Each fixed field was passing `isDisabled={isUpdating || isTesting}` (or `isDisabled={isUpdating}`) to a `StyledInput`. Renamed to `disabled={...}` so the field is genuinely disabled while a request is in flight.

`StyledInput.tsx` itself is unchanged in behavior — only its explanatory comment was updated. Its type stays permissive (`InputHTMLAttributes` + index signature) because a second, unrelated latent-bug family (below) still relies on it. Tightening the type to strict `InputHTMLAttributes` — which would catch this whole class of mistake at compile time — is deferred until that family is resolved.

## Verification

- **Strict-typing pass**: `StyledInput`'s type was temporarily tightened to strict `InputHTMLAttributes`; `tsc` confirmed all 14 fixed sites now type-check as valid `disabled`, and that **no `Button` was affected** (a Button changed to `disabled` would have surfaced a new error — none did). The permissive type was then restored.
- **Type-check**: 0 errors. **ESLint**: 0 errors (3 pre-existing unused-`err` warnings, untouched). **Prettier**: clean.
- **Full Jest suite**: failures observed locally were environmental timeout flakes on the dev machine (failure count escalated across repeated full runs while runtime grew — a resource-exhaustion signature, not a regression). None of the failing suites touch these pages or `StyledInput`, and all pass in isolation. CI on clean infra is the authoritative signal.

## Commits

| Hash | Message |
|------|---------|
| `e33e70d82` | [MLID-2742] - fix(admin-integrations): correct isDisabled -> disabled on StyledInput SFTP fields |

## Test Plan

For each of the five admin integration pages (`/admin/integrations/{alnylam,argenx,spruce,jotform,notion}`), as an Admin/Auditor user:

- [ ] Fill the required SFTP / API-key fields, then click **Test Connection** or **Save Settings**.
- [ ] While the request is in flight, confirm the input fields are now **disabled** (grayed out, reject input) — before this fix they stayed editable.
- [ ] Confirm the browser console no longer logs an "unrecognized `isDisabled` attribute" warning for those inputs.

## Follow-up (separate ticket, not in this PR)

"Bug family 2": an unsupported custom `error` prop is passed to `StyledInput` at ~10 sites in `AddEditLocationModal` and `FormItem/variants`. Unlike the `isDisabled` rename, this needs a design decision (add an error-state variant to `StyledInput`, use a different component, or drop the prop). Deferred to a dedicated cleanup ticket, after which `StyledInput`'s type can be tightened to strict `InputHTMLAttributes` to prevent recurrence.

## Jira

- [MLID-2742](https://localinfusion.atlassian.net/browse/MLID-2742)
