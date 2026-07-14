# MLID-2769 — Swap MaskedInput and MuiPhoneField internals to the new @repo/ui MaskInput

## Task Reference

- **Jira**: [MLID-2769](https://localinfusion.atlassian.net/browse/MLID-2769)
- **Story Points**: Not set in Jira at the subtask level (parent MLID-2742 carries the total)
- **Branch**: `feature/MLID-2769-swap-wrappers-to-maskinput`
- **Base Branch**: `feature/MLID-2742-inhouse-masked-input` (the parent integration branch, off `develop`, already pushed to origin). Merge this subtask branch back into it — **no per-subtask pull request.** The only pull request is `feature/MLID-2742-inhouse-masked-input` → `develop`, opened after all 3 subtasks land.
- **Status**: To Do (Jira status: IN DEVELOPMENT at the parent level)
- **Parent task**: MLID-2742 (In-house masked-input component in `@repo/ui`) — a Task, not a formal epic; there is no epic branch.
- **Depends on**: MLID-2768 (Headless `MaskInput` component + `useMaskedCaret` hook in `@repo/ui`) — **done, merged into `feature/MLID-2742-inhouse-masked-input`.** `MaskInput` is already exported from `@repo/ui` (`packages/ui/src/index.ts:159`: `export { MaskInput, type MaskInputProps } from './components';`) and its default output already matches the fork's `_`-padded placeholder shape, so this task is a pure internal swap.
- **Sibling subtask (not part of this task)**: MLID-2770 — remove the `@mona-health/react-input-mask` dependency and the ambient type shim (`apps/web/types/mona-health-react-input-mask.d.ts`). **Out of scope here** — this task keeps the dependency installed and the shim in place; MuiPhoneField's own import of it is what gets removed, not the package/shim themselves.
- **Commit format**: `[MLID-2769] - type(scope): description`, no `Co-Authored-By`.

---

## Summary

`MaskedInput` and `MuiPhoneField` are the two app-level wrappers that currently render the third-party `@mona-health/react-input-mask` fork (as a child-clone in `MaskedInput`, as a bare adapter input in `MuiPhoneField`). This task points both wrappers at the new in-house `@repo/ui` `MaskInput` component instead, without changing either wrapper's public props or touching any of their downstream consumers. A third file, `StyledInput`, only imports a *type* from the fork and is updated so it no longer needs to, clearing the way for MLID-2770 to delete the fork's type shim.

---

## Codebase Analysis

**What MLID-2768 already delivered** (verified in `packages/ui/src/components/MaskInput/MaskInput.tsx`): a headless `forwardRef<HTMLInputElement, MaskInputProps>` that renders a bare `<input>` (via `StyledMaskInput`, an Emotion `styled('input')`). `MaskInputProps` = `Omit<React.InputHTMLAttributes<HTMLInputElement>, 'value' | 'onChange' | 'type'>` extended with `mask: string` (required), `alwaysShowMask?: boolean` (default `false`), `value?: string`, `onChange?: React.ChangeEventHandler<HTMLInputElement>`, `onFocus?`, `onBlur?`, `type?: string` (default `'text'`). Internally it destructures `mask`, `alwaysShowMask`, `value`, `onChange`, `onFocus`, `onBlur`, `type`, `onKeyDown`, `onPaste`, `onCut`, and spreads everything else (`...rest` — e.g. `className`, `placeholder`, `name`, `id`, `onClick`, `disabled`) straight onto the underlying input. It merges any consumer-supplied `onKeyDown`/`onPaste`/`onCut` with its own internal caret-management handlers rather than overwriting them. `onChange` fires a synthetic `ChangeEvent` whose `event.target.value` is the fully-masked string (identical contract to the fork). Import specifier: `import { MaskInput, type MaskInputProps } from '@repo/ui';`.

**File 1 — `apps/web/components/UI/MaskedInput/MaskedInput.tsx`** (structural rework, verified current source):
```tsx
export const MaskedInput = forwardRef<HTMLInputElement, InputMaskProps>(
  ({ mask, alwaysShowMask = false, placeholder, value, onChange, onBlur, onFocus, type = 'text', ...maskProps }, ref) => (
    <InputMask ref={ref} mask={mask} alwaysShowMask={alwaysShowMask} value={value} onChange={onChange} onBlur={onBlur} onFocus={onFocus} {...maskProps}>
      <StyledInput type={type} placeholder={placeholder} className={styles.input} />
    </InputMask>
  )
);
```
The fork's `InputMask` takes a single child element and clones it with the masked props + its own ref (a fork-specific pattern documented in the file's own comment). `MaskInput` is headless and does not support child-cloning — it *is* the input. The rework renders `MaskInput` directly, folding `placeholder`/`className`/`type` into passthrough props and moving the `styles.module.css` `.input` class from `StyledInput`'s `className` onto `MaskInput`'s own `className`. `StyledInput` is dropped from this file entirely (it becomes an internal-only dependency of nothing in this file). The props type changes from the fork's permissive `InputMaskProps` (which carries an index signature, `maskChar`, `maskPlaceholder`, and a render-child signature — none of which any call site uses) to a clean own type built on `MaskInputProps`, keeping a `...rest` passthrough so arbitrary native input props keep flowing through.

Real call sites (verified, **do not edit**):
- `apps/web/components/UI/DatePicker/DatePicker.tsx` — `<MaskedInput ref={ref} mask="99/99/9999" placeholder={placeholder} onChange={handleChange} onClick={openPicker} />`. Passes `ref`, `onClick` — both must keep flowing through. **NOTE: this file is orphaned dead code** — no route or component in `apps/web` renders `UI/DatePicker/DatePicker` (the DatePickers actually used in the app come from `@repo/ui` or `@mui/x-date-pickers`, and the app's masked date entry is `MuiDatePicker.tsx`, a different component). It is still a compile-time consumer of `MaskedInput`, so the reworked `MaskedInput` MUST keep compiling against its `ref`/`onClick`/`onChange` usage (the `...rest` passthrough covers this) — but there is **no browser route to verify it** (see Affected Routes). Do not delete it in this task; flag for a possible future cleanup.
- `apps/web/components/Layout/GlobalSearch/components/patientDynamicFormItems.tsx` line 86 — `<MaskedInput mask="99/99/9999" placeholder="MM/DD/YYYY" {...field} />` (react-hook-form `field` = `value`, `onChange`, `onBlur`, `name`, `ref`).
- same file, line 115 — `<MaskedInput mask="999999" placeholder="123456" {...field} />`.
None of the 3 real call sites pass `alwaysShowMask`, `maskChar`, `maskPlaceholder`, or a render-child — confirming those fork-only props are safe to drop from the type.

**File 2 — `apps/web/components/UI/MUI/MuiPhoneField.tsx`** (direct adapter swap, verified current source): `PhoneMaskAdapter` is `forwardRef<HTMLInputElement, React.InputHTMLAttributes<HTMLInputElement>>` destructuring `{ value, onChange, onBlur, onFocus, ...rest }`, rendering `<InputMask ref={ref} mask={PHONE_MASK} value={(value as string) ?? ''} onChange={onChange as ...} onBlur={onBlur} onFocus={onFocus} {...rest} />` as MUI `TextField`'s `slotProps.input.inputComponent`. `MaskInput` is already the bare-input shape this slot expects (its inner `StyledMaskInput` renders a plain `<input>`), so the swap is `InputMask` → `MaskInput`, keeping `PHONE_MASK = '+1 (999) 999-9999'` as the hardcoded `mask` prop. The public `MuiPhoneFieldProps<T>` (react-hook-form abstraction: `label`, `name`, `control`, `watch`, `required`, `errorHelperText`) does not change. `EMPTY_MASK_DIGITS`/`hasUserInput` logic in the outer component is untouched — it operates on the value string, not on which mask engine produced it.

Real call sites (verified, **do not edit**):
- `apps/web/components/Modules/document-intake/forms/OrderForm.tsx` — 2 usages (lines ~1899, ~1907).
- `apps/web/components/Modules/document-intake/steps/patient-lookup-modes/PatientCreateMode.tsx` — 1 usage (line ~328).
All 3 pass only `MuiPhoneFieldProps` (never touch `PhoneMaskAdapter` directly).

**File 3 — `apps/web/components/UI/StyledInput/StyledInput.tsx`** (type-only change, verified current source):
```tsx
import { InputState } from '@mona-health/react-input-mask';
export const StyledInput = forwardRef<HTMLInputElement, InputState>(
  ({ placeholder, type = 'text', ...inputProps }: InputState, ref) => (
    <input {...inputProps} ref={ref} type={type} placeholder={placeholder} className={styles.input} />
  )
);
```
`InputState` is defined only in the local ambient shim `apps/web/types/mona-health-react-input-mask.d.ts` as `InputHTMLAttributes<HTMLInputElement> & { [prop: string]: unknown }` — deliberately permissive so it "avoids touching call sites," per that shim file's own comment. The shim is deleted in MLID-2770, so `StyledInput` must stop depending on it now. **This surfaces a real, pre-existing latent bug — see Resolved Decision 2 below.**

**Latent bug found (verified):** `apps/web/app/admin/integrations/abbvie/page.tsx` (5 call sites: lines 549, 563, 577, 592, 616) and `apps/web/app/admin/integrations/kedrion/page.tsx` (5 call sites: lines 484, 498, 512, 527, 551) all pass `isDisabled={isUpdating || isTesting}` to `<StyledInput>`. `isDisabled` is not a valid `<input>` attribute — `StyledInput`'s destructure only pulls out `placeholder`/`type`, so `isDisabled` falls into `...inputProps` and is spread directly onto the native `<input>` element, producing an unrecognized-DOM-attribute React warning and **never actually disabling the field**. It only type-checks today because of the shim's index signature (`[prop: string]: unknown`). Their two test files (`abbvie/page.test.tsx`, `kedrion/page.test.tsx`) fully mock `@/components/UI` — including a hand-rolled `StyledInput` stub that reads `isDisabled` and forwards it as `disabled={isDisabled}` on a plain `<input>` — so the bug is invisible to Jest today (only the "Save Settings" `Button` is asserted with `toBeDisabled()`/`not.toBeDisabled()`; no test asserts the SFTP text fields' disabled state).

**Existing MuiPhoneField test** (`apps/web/components/UI/MUI/MuiPhoneField.test.tsx`, verified): `jest.mock('@mona-health/react-input-mask', () => ({ default: forwardRef(...) }))` stubs the fork as a plain `<input>` that spreads `value`/`onChange`/`onBlur`/`onFocus`/`mask`/`...rest`. All 9 test cases interact with the field via `screen.getByTestId('phone-input')` (a prop injected by the mocked MUI `TextField`, forwarded through `PhoneMaskAdapter`'s `...rest`) — none of them depend on fork-specific behavior beyond "a controlled input that accepts value/onChange/onBlur." Once the adapter imports `MaskInput` instead of the fork, this `jest.mock('@mona-health/react-input-mask', ...)` block becomes dead (nothing in the source imports that module anymore) and must be removed; `@repo/ui` resolves to real source under Jest (`apps/web/jest.config.js`, `moduleNameMapper: { '^@repo/ui$': '<rootDir>/../../packages/ui/src/index.ts', ... }`), so the real `MaskInput` renders — it is pure and needs no mock. The `@mui/material`, `./MuiCommon`, and `./useOcrHighlight` mocks are unaffected and stay.

**`MaskedInput` has no existing test file** (`apps/web/components/UI/MaskedInput/` currently contains only `MaskedInput.tsx`, `index.ts`, `styles.module.css`) — this task adds one as a regression net, since `apps/web` runs Jest for real (unlike `packages/ui`, which has Vitest test files written but no wired runner).

**`MaskedInputExtended` is not in scope.** It surfaced in an initial grep for `MaskedInput` only because its name contains that substring — it is built on `cleave.js`, has no relationship to `@mona-health/react-input-mask`, and is used by `calculatorDynamicFormItems.tsx` for currency/percent masks (verified: that file imports `MaskedInputExtended`, not `MaskedInput`). Not touched by this task or by MLID-2770.

**Consumer count note:** the ticket's acceptance criteria says "all 9 downstream consumers work unchanged." Verified real production JSX usages: `DatePicker.tsx` (1× `MaskedInput`), `patientDynamicFormItems.tsx` (2× `MaskedInput`), `OrderForm.tsx` (2× `MuiPhoneField`), `PatientCreateMode.tsx` (1× `MuiPhoneField`) — **4 production files, 6 real call sites.** See Resolved Decisions for how to treat the "9" figure.

---

## Implementation Steps

Each step is a single commit. Commit format: `[MLID-2769] - type(scope): description`.

### Step 1 — RED: expose the `isDisabled` latent bug with a real (unmocked) `StyledInput` assertion

- **Files**:
  - `apps/web/app/admin/integrations/abbvie/page.test.tsx` (modify)
  - `apps/web/app/admin/integrations/kedrion/page.test.tsx` (modify)
- **What**: In both test files, change the `jest.mock('@/components/UI', () => ({ ... }))` factory from a fully-hand-rolled mock to `{ ...jest.requireActual('@/components/UI'), Alert: ..., BackButton: ..., Button: ..., Card: ..., LoadingSpinner: ... }` — i.e. spread the real module and only override the components whose behavior needs stubbing, letting the **real** `StyledInput` render. Add one new test per file: render the page in its "updating" state (however `isUpdating` is already driven in existing tests — reuse that setup) and assert the SFTP Host input (`screen.getByPlaceholderText(...)` or the page's existing `input-{placeholder}`-style query, adapted since the real `StyledInput` doesn't add a `data-testid`) is disabled: `expect(hostInput).toBeDisabled()`. Run it — it fails today, because `isDisabled` is spread onto the DOM as an unrecognized attribute and the real `<input>` has no `disabled` attribute. This proves the bug with a real assertion instead of asserting against the source code by inspection. Commit type: `test`.

### Step 2 — GREEN: rename `isDisabled` → `disabled` at the 10 `StyledInput` call sites

- **Files**:
  - `apps/web/app/admin/integrations/abbvie/page.tsx` (modify — 5 call sites)
  - `apps/web/app/admin/integrations/kedrion/page.tsx` (modify — 5 call sites)
- **What**: Change `isDisabled={isUpdating || isTesting}` to `disabled={isUpdating || isTesting}` on all 10 `<StyledInput>` JSX tags (the `Button` components on the same pages already correctly use `isDisabled` as their own prop name — a `@repo/ui`/custom-`Button` convention — and are **not** touched; only `StyledInput` usages change). Run Step 1's new tests — both now pass. Commit type: `fix` (this is a real production bug fix: the fields were never actually disabled during an in-flight update/test).

### Step 3 — REFACTOR: `StyledInput` stops depending on the fork's `InputState` type

- **Files**:
  - `apps/web/components/UI/StyledInput/StyledInput.tsx` (modify)
- **What**: Replace `import { InputState } from '@mona-health/react-input-mask'` with the plain `React.InputHTMLAttributes<HTMLInputElement>` type (no index signature), applied to both the `forwardRef` generic and the destructured props parameter. Since Step 2 already eliminated the one call-site pattern (`isDisabled`) that relied on the shim's permissive index signature, this type tightening introduces no new compile errors anywhere in the codebase. Run `npm run types:check` (from `apps/web/`) to confirm. Run Step 1's tests again — still green (they don't touch typing). Commit type: `refactor`.

### Step 4 — RED (regression guard): drop the fork mock from `MuiPhoneField.test.tsx`

- **Files**:
  - `apps/web/components/UI/MUI/MuiPhoneField.test.tsx` (modify)
- **What**: Remove the `jest.mock('@mona-health/react-input-mask', () => ({ ... }))` block and its accompanying comment. Run the suite against the still-fork-importing `MuiPhoneField.tsx` (unchanged in this step). Expected outcome: the real `@mona-health/react-input-mask` fork now renders in place of the removed mock. All 9 existing test cases are expected to keep passing, since none of them assert fork-internal behavior — they only interact through `screen.getByTestId('phone-input')`, a prop injected by the (already-mocked) MUI `TextField`, forwarded through `PhoneMaskAdapter`'s `...rest`. **If any assertion unexpectedly fails here, stop and investigate before proceeding** — it means an assumption about the current external contract is wrong, and the swap in Step 5 would silently change behavior instead of preserving it. This step is the regression net for Step 5, not a feature-adding test — record the actual outcome (all-green or a specific failure) before moving on. Commit type: `test`.

### Step 5 — REFACTOR: swap `PhoneMaskAdapter`'s `InputMask` → `MaskInput`

- **Files**:
  - `apps/web/components/UI/MUI/MuiPhoneField.tsx` (modify)
- **What**: Replace `import InputMask from '@mona-health/react-input-mask'` with `import { MaskInput } from '@repo/ui'`. In `PhoneMaskAdapter`, replace the rendered `<InputMask ref={ref} mask={PHONE_MASK} value={...} onChange={...} onBlur={onBlur} onFocus={onFocus} {...rest} />` with `<MaskInput ref={ref} mask={PHONE_MASK} value={(value as string) ?? ''} onChange={onChange as React.ChangeEventHandler<HTMLInputElement>} onBlur={onBlur} onFocus={onFocus} {...rest} />` — same prop shape, new component. `PHONE_MASK`, `EMPTY_MASK_DIGITS`, and all outer `MuiPhoneField` logic (`hasUserInput`, error/hint display, `Controller` wiring) are untouched. Run Step 4's test file — must pass with zero test modification (this is the GREEN confirmation that behavior is preserved). Commit type: `refactor`.

### Step 6 — RED: new `MaskedInput.test.tsx` regression net

- **Files**:
  - `apps/web/components/UI/MaskedInput/MaskedInput.test.tsx` (new)
- **What**: Create the first test file for `MaskedInput` (none exists today). Using React Testing Library + `@testing-library/user-event`, cover the real contract this component must preserve:
  - renders an `<input>` element and forwards a `ref` to the underlying DOM node;
  - forwards `placeholder` and `className` (assert the `styles.input` CSS Modules class ends up on the rendered `<input>`, matching the current `.input` hover/focus styling contract);
  - forwards `type` (e.g. render with `type="text"`, assert `input.type === 'text'`);
  - typing into a controlled instance with `mask="99/99/9999"` produces the correctly masked value in `onChange`'s `event.target.value` (e.g. typing `"01152026"` results in `"01/15/2026"`, matching `DatePicker.tsx`'s real usage);
  - typing into a controlled instance with `mask="999999"` produces the correctly masked (digit-only, no fixed characters) value (matching `patientDynamicFormItems.tsx`'s real usage);
  - an `onClick` handler passed through (matching `DatePicker.tsx`'s real usage) still fires.
  This is a **characterization/regression test**, not a feature-adding one — since the current fork-based `MaskedInput.tsx` already satisfies this exact external contract (that's the whole point of the swap: identical behavior, different engine), these assertions are expected to already pass against the current source. Run them against the unmodified `MaskedInput.tsx` first and confirm they pass — that establishes the regression net *before* touching the internals; the Storybook-and-browser-verified `MaskInput` engine from MLID-2768 already guarantees the masked-output shape matches. If anything unexpectedly fails here, investigate before proceeding, same as Step 4. Commit type: `test`.

### Step 7 — REFACTOR: rework `MaskedInput` to render `@repo/ui`'s `MaskInput` directly

- **Files**:
  - `apps/web/components/UI/MaskedInput/MaskedInput.tsx` (modify)
- **What**: Remove the `@mona-health/react-input-mask` import and the `StyledInput` import. Define a new own props type (e.g. `type MaskedInputProps = MaskInputProps` or a thin wrapper if any narrowing is needed — decide the minimal shape at implementation time; it must at minimum keep `mask`, `alwaysShowMask`, `placeholder`, `value`, `onChange`, `onBlur`, `onFocus`, `type`, `className`, and an index-free `...rest` passthrough for arbitrary native input props like `onClick` and react-hook-form's spread). Render `<MaskInput ref={ref} mask={mask} alwaysShowMask={alwaysShowMask} placeholder={placeholder} value={value} onChange={onChange} onBlur={onBlur} onFocus={onFocus} type={type} className={styles.input} {...rest} />` — a single element, no child-clone. Keep `MaskedInput.displayName = 'MaskedInput'`. Run Step 6's tests — must pass with zero test modification (GREEN confirmation). Commit type: `refactor`.

### Step 8 — Verification pass: types, lint, format, full test run

- **Files**: none expected (fix-up only if a check fails)
- **What**: From `apps/web/`, run `npx tsc --noEmit` (or `npm run types:check`), `npm run lint`, and `npx prettier --check` on all files touched in Steps 1–7. Run the full affected test slice: `npx jest MaskedInput MuiPhoneField abbvie/page kedrion/page`. Confirm no unused imports remain (`InputMask`, `InputState`, `StyledInput` should no longer appear in `MuiPhoneField.tsx` or `MaskedInput.tsx`; the fork package itself and its type shim remain installed/present for MLID-2770 to remove). Confirm the untouched consumers still compile: `DatePicker.tsx`, `patientDynamicFormItems.tsx`, `OrderForm.tsx`, `PatientCreateMode.tsx`. If any check fails, fix in a follow-up commit of the appropriate type (`fix`) rather than amending prior commits. This step produces a commit only if fixes were needed.

---

## Files Affected

| File | Action | Description |
|------|--------|--------------|
| `apps/web/app/admin/integrations/abbvie/page.test.tsx` | Modify | Partial-mock `@/components/UI` (real `StyledInput`), add disabled-state regression test |
| `apps/web/app/admin/integrations/kedrion/page.test.tsx` | Modify | Partial-mock `@/components/UI` (real `StyledInput`), add disabled-state regression test |
| `apps/web/app/admin/integrations/abbvie/page.tsx` | Modify | Fix latent bug: `isDisabled` → `disabled` on 5 `<StyledInput>` call sites |
| `apps/web/app/admin/integrations/kedrion/page.tsx` | Modify | Fix latent bug: `isDisabled` → `disabled` on 5 `<StyledInput>` call sites |
| `apps/web/components/UI/StyledInput/StyledInput.tsx` | Modify | Type source: fork's `InputState` → `React.InputHTMLAttributes<HTMLInputElement>` |
| `apps/web/components/UI/MUI/MuiPhoneField.test.tsx` | Modify | Remove dead `jest.mock('@mona-health/react-input-mask', ...)` block |
| `apps/web/components/UI/MUI/MuiPhoneField.tsx` | Modify | `PhoneMaskAdapter`: fork's `InputMask` → `@repo/ui`'s `MaskInput` |
| `apps/web/components/UI/MaskedInput/MaskedInput.test.tsx` | Create | Render, ref-forwarding, className/placeholder/type passthrough, masked-typing (date + 6-digit), onClick passthrough |
| `apps/web/components/UI/MaskedInput/MaskedInput.tsx` | Modify | Render `@repo/ui`'s `MaskInput` directly; drop fork import and `StyledInput` child-clone pattern |

Not modified (verified real consumers, kept working unchanged by design):
`apps/web/components/UI/DatePicker/DatePicker.tsx`, `apps/web/components/Layout/GlobalSearch/components/patientDynamicFormItems.tsx`, `apps/web/components/Modules/document-intake/forms/OrderForm.tsx`, `apps/web/components/Modules/document-intake/steps/patient-lookup-modes/PatientCreateMode.tsx`, `apps/web/components/UI/MaskedInputExtended/MaskedInputExtended.tsx` (unrelated, cleave.js-based), `apps/web/types/mona-health-react-input-mask.d.ts` (removed in MLID-2770, not here).

---

## Affected Routes

These are the browser routes where the swapped inputs (and the `isDisabled` fix) actually render. Use them for the manual verification steps in Testing Strategy.

| Route (URL) | What renders there | Mask / change | Gating / preconditions |
|-------------|--------------------|---------------|------------------------|
| `/intakes/[id]/review` | `OrderForm` — "Provider Phone" + "Provider Fax" | Phone `+1 (999) 999-9999` (`MuiPhoneField`) | Intake `type` must be `spruceFax` or `eOrder` (gates the "Review Documents" entry button); reach wizard step 3 → "Order" section tab |
| `/intakes/[id]/review` | `PatientCreateMode` — "Mobile" | Phone `+1 (999) 999-9999` (`MuiPhoneField`) | On wizard step 1, click the "Create Patient" mode tab |
| Global header (any authenticated route except `/`) | GlobalSearch → "Create Patient's Profile" drawer — "Date of Birth" | Date `99/99/9999` (`MaskedInput`) | None (no role/flag gate); type a no-match query to reveal the empty-state "Create Patient's Profile" button |
| Global header (same drawer) | Same drawer — "WeInfuse ID (IV)" | 6-digit `999999` (`MaskedInput`) | Same drawer, a few fields below Date of Birth |
| `/admin/integrations/abbvie` | SFTP Host/Port/Username/Password/Remote Path (`StyledInput`) | `isDisabled` → `disabled` bug fix (not a mask) | Admin/Auditor role only (`hasAdminPermissions`), else redirected to `/`; unauthenticated → `/login` |
| `/admin/integrations/kedrion` | Same 5 SFTP fields (`StyledInput`) | `isDisabled` → `disabled` bug fix | Same as abbvie |
| _(no route)_ | `UI/DatePicker/DatePicker.tsx` (orphaned dead code) | Date `99/99/9999` (`MaskedInput`) | **Not browser-reachable** — nothing renders it. Compile-only consumer; verified by type-check, not by browser. |

**Route reachability note:** the phone masks live behind the document-intake wizard (`/intakes/[id]/review`), which needs a real intake of the right `type`; the date + 6-digit masks are reachable from **any** signed-in page via the global header search drawer (much easier to reach — good primary check). The `DatePicker.tsx` date consumer is dead code with no route, so the date mask is verified through the GlobalSearch drawer instead.

---

## Testing Strategy

- **Unit tests (Jest, real execution — `apps/web` runs Jest for real, unlike `packages/ui`'s unwired Vitest)**:
  - `MuiPhoneField.test.tsx`: all 9 existing assertions preserved verbatim; only the dead fork mock is removed (Step 4). `@repo/ui` resolves to real *source* under Jest via `moduleNameMapper` (`^@repo/ui$` → `packages/ui/src/index.ts`), so the real `MaskInput` renders with no mock needed — it is pure (no network/DB/external service).
  - `MaskedInput.test.tsx` (new): render + ref forwarding + `className`/`placeholder`/`type` passthrough; masked typed-output for the two masks actually used in production (`99/99/9999`, `999999`); `onClick` passthrough. Written and confirmed passing *before* the internals are reworked (characterization test), then confirmed still passing *after* (Step 6 → Step 7).
  - `abbvie/page.test.tsx` / `kedrion/page.test.tsx`: partial-mock `@/components/UI` (spread `jest.requireActual`, override only `Alert`/`BackButton`/`Button`/`Card`/`LoadingSpinner`) so the real `StyledInput` renders; new assertion that the SFTP Host field is actually `disabled` while `isUpdating`/`isTesting` is true. Existing assertions (e.g. `toBeDisabled()`/`not.toBeDisabled()` on the Save button) are unaffected since `Button` stays mocked.
- **Consumer tests unaffected, not touched**: `OrderForm.*.test.tsx`, `PatientCreateMode.test.tsx` all mock `MuiPhoneField` at the module boundary already — verify this remains true, but no edits expected.
- **Mock boundaries**: `MaskInput`/`useMaskedCaret` themselves need no mocking anywhere (pure, no external dependencies) — this is what makes removing the fork mock in `MuiPhoneField.test.tsx` safe. The fork package (`@mona-health/react-input-mask`) is still installed (removed in MLID-2770) but no longer imported by any file this task touches after Step 5/7.
- **Manual / browser verification (run by the user, not the AI) — concrete route-by-route steps.** With the dev server running, this is the load-bearing acceptance check (unit tests are the regression net; real-browser caret behavior is the final word). For every masked field below, exercise the full caret set: type digits, delete mid-string, delete **across a fixed character** (`)`, `-`, `/`), paste a full value, range-select-then-type, and tab focus in/out. Confirm no React 19 console errors (no `findDOMNode`, no caret jump-to-end).

  1. **Date mask `99/99/9999` and 6-digit mask `999999` — GlobalSearch drawer (easiest; reachable from any signed-in page).**
     1. Log in; land on any page other than `/` (the header search is mounted everywhere except the base route).
     2. Click the global search input in the top header and type a nonsense name that matches no patient.
     3. In the empty-results dropdown, click **"Create Patient's Profile"** — the "New patient profile" drawer slides in from the right.
     4. **Date of Birth** field → date mask `99/99/9999` (placeholder `MM/DD/YYYY`). Run the full caret set; e.g. type `01152026` → shows `01/15/2026`; backspace across a `/`.
     5. **WeInfuse ID (IV)** field (a few fields below) → 6-digit mask `999999` (placeholder `123456`). Type `123456`; confirm digit-only, no fixed characters.

  2. **Phone mask `+1 (999) 999-9999` — `MuiPhoneField` in Patient Create Mode (`/intakes/[id]/review`).**
     1. Go to `/intakes` and open an intake of type `spruceFax` or `eOrder` (`/intakes/[id]`).
     2. Click **"Review Documents"** → routes to `/intakes/[id]/review` and opens the document-intake wizard on step 1 "Patient Lookup".
     3. In the mode selector strip, click **"Create Patient"**.
     4. Scroll to "Contact Information" → **"Mobile"** field (phone mask). Run the full caret set; e.g. type `5551234567` → `+1 (555) 123-4567`; backspace across `)` and `-`; paste `(555) 123-4567`.

  3. **Phone mask `+1 (999) 999-9999` — `MuiPhoneField` in Order Form (`/intakes/[id]/review`).**
     1. From the same wizard (Route 2 above, or reach step 3 directly), advance to step 3 **"Review Data"**.
     2. In the "Review sections" badge row, click the **"Order"** badge to mount `OrderForm`.
     3. Expand the **"Prescribing Provider"** section → "Contact Information" → **"Provider Phone"** and **"Provider Fax"** fields (both phone mask). Run the full caret set on each.

  4. **`isDisabled` → `disabled` fix — admin integration pages (`/admin/integrations/abbvie` and `/admin/integrations/kedrion`).** (Admin/Auditor role required.)
     1. As an admin, go to `/admin/integrations/abbvie` (repeat for `/kedrion`), reachable also via `/admin/integrations` → the AbbVie / Kedrion tile.
     2. Fill Host / Username / Password, then click **"Test Connection"** or **"Save Settings"**.
     3. While the request is in flight, confirm the five SFTP fields (Host, Port, Username, Password, Remote Path) are now **actually disabled** (grayed out, reject input) — before this fix they stayed typeable. Confirm the React "unrecognized `isDisabled` attribute" console warning is gone.

  5. **`UI/DatePicker/DatePicker.tsx` — no browser check possible.** It is orphaned dead code with no route (see Affected Routes). Its correctness after the swap is covered by the type-check in Step 8 only; there is nothing to click.
- **Full command reference**: `cd apps/web && npx jest MaskedInput MuiPhoneField abbvie/page kedrion/page`, then `npm run types:check`, `npm run lint`, `npx prettier --check <touched files>`.

---

## Security Considerations

- **Input validation**: not applicable — this task swaps a UI formatting engine underneath existing wrappers. No validation logic changes (`isValidMobilePhone`, `orderYupValidator`, date parsing in `DatePicker.tsx` are all untouched).
- **Authorization**: not applicable — `MaskedInput`/`MuiPhoneField`/`StyledInput` are unauthenticated, permission-free UI primitives. The two admin integration pages touched for the `isDisabled` fix already have their own role gating (`hasAdminPermissions`), which this task does not modify.
- **Feature flag gating**: none needed — this is an internal, behavior-preserving implementation swap, not a new user-facing feature or toggle.
- **PHI handling**: `MaskedInput`/`MuiPhoneField` carry phone numbers and dates of birth (PHI-adjacent) through `value`/`onChange`. No logging is introduced anywhere in this task (no `console.log`/`logger` calls added to any of the 3 wrapper files or their tests). Test fixtures use obviously-fake values (e.g. `"5551234567"`, `"01152026"`), never real-looking production data.

---

## Resolved / Open Decisions

1. **StyledInput type fix — REVISED mid-implementation (2026-07-09).** Option A was approved on the estimate of "2 files / 10 call sites." During implementation, tightening `StyledInput` to strict `React.InputHTMLAttributes` surfaced **~19 latent-bug sites across 8 files** in **two** categories: (a) `isDisabled` that never disabled — `argenx` (5), `spruce` (2), `jotform` (1), `notion` (1), on top of `abbvie`/`kedrion`; and (b) a custom `error` prop passed to `StyledInput` — `AddEditLocationModal` (9), `FormItem/variants` (1) — which needs design intent, not a mechanical fix. Given the blast radius, the user chose the **minimal** path:
   - `StyledInput` gets a **local permissive type** (`interface StyledInputProps extends InputHTMLAttributes<HTMLInputElement> { [prop: string]: unknown }`) with **no fork import** — this satisfies MLID-2769's actual requirement (drop the fork type so MLID-2770 can delete the shim) with zero collateral.
   - The `abbvie`/`kedrion` `isDisabled` fixes (Steps 1–2) were **already done and kept** — they are real fixes backed by RED→GREEN tests (using the real `StyledInput` via `jest.requireActual`, queries moved to `getByPlaceholderText`).
   - The remaining ~19-site latent-bug cleanup (argenx/jotform/spruce/notion `isDisabled`, and the `error`-prop misuse) is **out of scope** and should be filed as a **separate ticket**.
   Steps 4–5 (MuiPhoneField) were combined into a single swap (remove the dead fork mock + point `PhoneMaskAdapter` at `MaskInput`), avoiding an intermediate state that renders the real fork.
   - **Option A (recommended)**: switch `StyledInput` to the clean `React.InputHTMLAttributes<HTMLInputElement>` type (no index signature) **and** fix the 2 real call sites currently passing the invalid `isDisabled` prop (`abbvie/page.tsx`, `kedrion/page.tsx`, 5 call sites each = 10 total) to the correct `disabled` prop. This fixes a real, verified latent bug (the SFTP host/port/username/password fields were never actually disabled during an in-flight update, and React already logs an unrecognized-DOM-attribute warning for `isDisabled` at runtime today). Extra churn: 2 source files + 2 test files, all already staged as Steps 1–2 above.
   - **Option B**: give `StyledInput` a local permissive type (its own index-signature-carrying type, replacing the fork's `InputState` 1:1) to avoid touching the 2 admin pages at all. Zero extra churn, but perpetuates the latent bug and keeps a type-safety hole that was only ever an artifact of depending on the fork's shim.
   - This plan is written assuming **Option A**. If the user prefers Option B instead, skip Steps 1–2 entirely and change Step 3 to define a local permissive type in `StyledInput.tsx` (e.g. `type StyledInputProps = React.InputHTMLAttributes<HTMLInputElement> & Record<string, unknown>`) instead of switching to the strict type.

2. **Consumer-count discrepancy.** The Jira ticket's acceptance criteria states "all 9 downstream consumers work unchanged." Verified actual production JSX usages: `DatePicker.tsx` (1), `patientDynamicFormItems.tsx` (2), `OrderForm.tsx` (2), `PatientCreateMode.tsx` (1) — **4 files, 6 call sites.** Treat "9" as the ticket author's informal estimate (possibly counting test files, the two wrapper files themselves, or other near-matches), not a literal checklist. The actual scope of "consumers that must keep working unchanged" is the 4 files listed above plus the two wrapper test files (`MuiPhoneField.test.tsx` gets one intentional, harmless edit — the dead mock removal — everything else stays green).

3. **`MaskedInputExtended` confirmed out of scope** — it is a `cleave.js`-based component with no relationship to `@mona-health/react-input-mask`; it only surfaced in an initial substring grep for "MaskedInput". No action needed.

4. **Branch/merge mechanics** mirror MLID-2768 exactly: `feature/MLID-2769-swap-wrappers-to-maskinput` branches off `feature/MLID-2742-inhouse-masked-input` and merges back into it directly, no per-subtask pull request. The single pull request for the whole parent task is `feature/MLID-2742-inhouse-masked-input` → `develop`, opened after MLID-2770 also lands.

5. **Do not remove the `@mona-health/react-input-mask` npm dependency or the ambient type shim** (`apps/web/types/mona-health-react-input-mask.d.ts`) in this task, even though after Step 5/7 no file imports the fork anymore — that removal, plus any residual `package.json`/`package-lock.json` changes, is explicitly MLID-2770's scope.

6. **`UI/DatePicker/DatePicker.tsx` is orphaned dead code (open item, no action this task).** The route investigation confirmed nothing in `apps/web` renders it — the app's real date pickers come from `@repo/ui`, `@mui/x-date-pickers`, and `MuiDatePicker.tsx`. It remains a compile-time consumer of `MaskedInput`, so the reworked `MaskedInput` must keep it type-checking (covered by the `...rest` passthrough for its `ref`/`onClick`/`onChange`), but there is no browser route to verify it. Do NOT delete it here (out of scope, would need team confirmation per the "delete only confirmed-unused code, per authorized batch" convention). Worth raising with the team as a separate cleanup candidate.
