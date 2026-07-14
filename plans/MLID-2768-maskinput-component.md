# MLID-2768 — Headless MaskInput component + useMaskedCaret hook in @repo/ui

## Task Reference

- **Jira**: [MLID-2768](https://localinfusion.atlassian.net/browse/MLID-2768)
- **Story Points**: 5 (parent MLID-2742 total; MLID-2768 is subtask 1 of 3)
- **Branch**: `feature/MLID-2768-maskinput-component`
- **Base Branch**: `feature/MLID-2742-inhouse-masked-input` (the parent integration branch, itself off `develop`). Merge this subtask branch into `feature/MLID-2742-inhouse-masked-input` (no per-subtask PR). The only PR is `feature/MLID-2742-inhouse-masked-input` → `develop` at the end of all 3 subtasks. **Decided 2026-07-09.**
- **Status**: To Do
- **Parent task**: MLID-2742 (In-house masked-input component in `@repo/ui`) — a Task, not a formal epic; there is no epic branch. Overview pre-plan: `docs/agomez/preplans/MLID-2742.md`
- **Sibling subtasks (not part of this task)**:
  - MLID-2769 — swap `MaskedInput` and `MuiPhoneField` internals to use `MaskInput` (depends on this task)
  - MLID-2770 — remove `@mona-health/react-input-mask` dependency + type shim (depends on MLID-2769)

---

## Summary

Build a new headless, unstyled `MaskInput` component and a `useMaskedCaret` hook in the `@repo/ui` design system (`packages/ui`), supporting exactly three digit-only (`9` placeholder) mask patterns: phone (`+1 (999) 999-9999`), date (`99/99/9999`), and a 6-digit code (`999999`). This replaces the caret/formatting logic currently provided by the third-party `@mona-health/react-input-mask` fork, which PE does not want production depending on. This task delivers the component in isolation — with a Storybook story and unit tests — and does not touch any existing consumer.

---

## Codebase Analysis

- **Component folder convention** (mirrored from `packages/ui/src/components/TextField/`): 5 files — `index.ts` (barrel), `ComponentName.tsx`, `ComponentName.styles.ts`, `ComponentName.test.tsx`, `ComponentName.stories.tsx`.
- **Component form**: `forwardRef<HTMLInputElement, MaskInputProps>(function MaskInput(props, ref) { ... })` — named function inside `forwardRef`, no separate `displayName`. See `TextField.tsx` for the pattern (there it forwards a `HTMLDivElement` ref; `MaskInput` forwards `HTMLInputElement` since it renders a bare `<input>`).
- **No `'use client'`**: nothing in `packages/ui/src` carries this directive — it's a framework-agnostic Vite/tsup package. Must not be added here, even though the parent pre-plan mentions it; that directive belongs to `apps/web` consumers, not this package.
- **Styling**: Emotion `styled` in a `.styles.ts` file (`import styled from '@emotion/styled'`), never `sx` objects or CSS modules in this package. Theme tokens under `theme.li.*`. Because `MaskInput` is headless/unstyled by design (it must render a bare `<input>` so it can be dropped into MUI's `inputComponent` slot in `MuiPhoneField`), styling stays minimal — a lightly wrapped `styled('input')` that passes through `className`/`style` untouched, no visual opinion (no border, padding, or color decisions that would leak into either consumer's styling).
- **Barrel exports**: two-hop re-export. `packages/ui/src/components/index.ts` has one line per component: `export { X, type XProps } from './X';` (see the `TextField` line for the exact form). `packages/ui/src/index.ts` re-exports batches of components from `./components` under a comment header (e.g. `// TextField`-style groupings under `// Components`, `// Tier 2 Components`, etc.). `MaskInput` needs one line added to `components/index.ts` and one export line added to `src/index.ts` (new small group, e.g. under a `// MaskInput` comment, following the `// NumberStepper` / `// Stepper` pattern already used for single-component groups).
- **Hooks**: no `packages/ui/src/hooks/` folder exists yet. This task creates it: `packages/ui/src/hooks/useMaskedCaret/useMaskedCaret.ts` + `useMaskedCaret.test.ts`, re-exported from `packages/ui/src/index.ts` (mirror the same named-export style used for components, e.g. `export { useMaskedCaret, type UseMaskedCaretOptions, type UseMaskedCaretResult } from './hooks/useMaskedCaret/useMaskedCaret';` — confirm the exact relative path/index approach when implementing; if other hooks are added later a `hooks/index.ts` barrel may be worth introducing, but for a single hook a direct re-export is enough, matching the "don't over-engineer" principle).
- **Storybook**: CSF3 format, `import type { Meta, StoryObj } from '@storybook/react-vite'`. Auto-discovered by `apps/storybook/.storybook/main.ts` glob — no manual registration. Every story renders inside `<LiThemeProvider>` via `preview.tsx`. Controlled/stateful components use a `render: () => { const [value, setValue] = useState(...); return <Component ... /> }` story (see `TextField.stories.tsx`'s `Phone` and `PhoneFax` stories) rather than plain `args`, because typing has to flow through `onChange` to be visible.
- **Unit test runtime — Vitest, not Jest**: `import { describe, expect, it, vi } from 'vitest'`, `@testing-library/react`, `@testing-library/jest-dom/vitest`, `@testing-library/user-event`. Shared helpers already exist: `packages/ui/src/test/setup.ts` and `packages/ui/src/test/renderWithTheme.tsx` (wraps `render()` in `<LiThemeProvider>` — always use this, never raw `render`).
- **Known gap — no wired test runner in `packages/ui`**: `packages/ui/package.json` has no `test` script, and neither `vitest` nor `@vitest/*` appear in its `devDependencies`. There is no `vitest.config.ts` in `packages/ui`. 37 pre-existing `*.test.tsx` files already reference `vitest` imports, but nothing currently runs them — `turbo run test` does not include `packages/ui` in its pipeline. **Per Resolved Decisions (2026-07-09): this task writes the test files following the existing vitest pattern but does NOT wire up a runner, and does not attempt to execute them.** Verification is Storybook + real browser only.
- **Caret algorithm reference** (design/behavior reference only, not a copy source): `node_modules/@mona-health/react-input-mask/lib/react-input-mask.development.js`, MIT-licensed (`node_modules/@mona-health/react-input-mask/LICENSE.md`, © 2016 Nikita Lobachev). Primitives to reimplement in a simplified, digit-only form: mask parsing into editable/fixed positions, "snap to nearest editable position," auto-advance past fixed characters on input, delete-across-fixed-character handling, and imperative `setSelectionRange` application (typically via `useLayoutEffect`, guarded by focus state). Because only 3 fixed digit-only patterns are in scope, `useMaskedCaret` can hardcode a much smaller parser than the fork's general-purpose one — no `beforeMaskedStateChange`, no custom `maskChar`/`maskPlaceholder`, no alpha/alphanumeric/regex/optional-section support.
- **Prop surface required by this task**: `mask: string`, `alwaysShowMask?: boolean` (default `false`), `value?: string`, `onChange?: React.ChangeEventHandler<HTMLInputElement>`, `onBlur?: React.FocusEventHandler<HTMLInputElement>`, `onFocus?: React.FocusEventHandler<HTMLInputElement>`, `placeholder?: string`, `type?: string` (default `'text'`), `ref` (forwarded), `className?: string`, `style?: React.CSSProperties`, plus standard input passthroughs (e.g. `name`, `id`, `disabled`, `readOnly`, `autoFocus`, `required` — spread via a `...rest` typed as `Omit<React.InputHTMLAttributes<HTMLInputElement>, 'value' | 'onChange' | 'type'>` or similar, decided at implementation time). `maskChar`/`maskPlaceholder` are intentionally omitted (see Open Questions).
- **Build/packaging**: tsup, ESM+CJS, `dts: true`, `esbuildOptions` sets `jsx: 'automatic'` / `jsxImportSource: '@emotion/react'`. Adding a component requires only the folder + the two barrel export lines above — no `tsup.config` or `package.json` changes needed.

---

## Implementation Steps

Each step is a single commit. Commit format: `[MLID-2768] - type(scope): description` (no `Co-Authored-By`).

### Step 1 — RED: scaffold MaskInput folder with a failing render test

- **Files**:
  - `packages/ui/src/components/MaskInput/MaskInput.test.tsx` (new)
- **What**: Write the first test importing `MaskInput` from `./MaskInput` (doesn't exist yet) via `renderWithTheme`. Assert it renders an `<input>` element (via `getByRole('textbox')` or a `data-testid`), forwards `className`, and forwards a `ref` to the underlying `<input>` DOM node. This test fails because the module doesn't exist. Commit type: `test`.

### Step 2 — GREEN: minimal MaskInput rendering a bare, unmasked input

- **Files**:
  - `packages/ui/src/components/MaskInput/MaskInput.tsx` (new)
  - `packages/ui/src/components/MaskInput/MaskInput.styles.ts` (new)
  - `packages/ui/src/components/MaskInput/index.ts` (new)
  - `packages/ui/src/components/index.ts` (modify — add barrel line)
  - `packages/ui/src/index.ts` (modify — add re-export)
- **What**: Minimal `forwardRef<HTMLInputElement, MaskInputProps>` implementation that renders a lightly-styled `styled('input')` from `MaskInput.styles.ts`, forwarding `ref`, `className`, `style`, `value`, `onChange`, and standard passthroughs — no masking logic yet, just enough to pass Step 1's test. `MaskInputProps` interface defined in `MaskInput.tsx` per the prop surface in Codebase Analysis (mask-related props accepted but unused at this step). Barrel-wire it into both index files following the `TextField/NumberStepper` pattern. Commit type: `feat`.

### Step 3 — RED: useMaskedCaret value-formatting tests for the 3 masks

- **Files**:
  - `packages/ui/src/hooks/useMaskedCaret/useMaskedCaret.test.ts` (new)
- **What**: Test the pure formatting behavior of the hook (or an exported pure helper it uses internally) for all 3 masks, independent of caret/DOM concerns:
  - phone mask `+1 (999) 999-9999`: raw digits `"5551234567"` → formatted `"+1 (555) 123-4567"`; partial input `"555"` → `"+1 (555"` (or the hook's defined partial-format shape — pin down exact partial behavior in the test itself); non-digit characters in input are stripped before formatting.
  - date mask `99/99/9999`: raw digits `"01152026"` → `"01/15/2026"`.
  - 6-digit code mask `999999`: raw digits `"123456"` → `"123456"` (no fixed characters, formatting is a no-op beyond digit-only filtering).
  - Over-length input is truncated to the mask's digit capacity (no overflow past the last editable position).
  - `alwaysShowMask: true` renders remaining placeholder positions (e.g. `9` positions un-filled show as `_` or whatever placeholder character the hook defines — pin this down explicitly in the test); `alwaysShowMask: false` (default) shows only the characters typed so far plus interleaved fixed characters up to the last entered digit.
  These tests fail because `useMaskedCaret` doesn't exist yet. Commit type: `test`.

### Step 4 — GREEN: useMaskedCaret formatting implementation

- **Files**:
  - `packages/ui/src/hooks/useMaskedCaret/useMaskedCaret.ts` (new)
  - `packages/ui/src/index.ts` (modify — add hook re-export)
- **What**: Implement the mask parser (digit-only `9` placeholder, fixed characters are everything else in the mask string) and the raw-value → formatted-value function, satisfying Step 3's tests. No caret positioning logic yet — the hook returns formatted `value` and a `handleChange` that strips non-digits and reformats. `MaskInput` is not wired to the hook yet (kept as a separate step to isolate the two concerns). Commit type: `feat`.

### Step 5 — RED: MaskInput + useMaskedCaret integration test (typing produces masked output)

- **Files**:
  - `packages/ui/src/components/MaskInput/MaskInput.test.tsx` (modify)
- **What**: Using `@testing-library/user-event`, simulate typing digits into a controlled `MaskInput` (wrapped in a small test harness component holding `useState`) for the phone mask, and assert the displayed input value matches the expected masked output at each stage (e.g. after typing `"5"` the field shows `"+1 (5"`, per whatever partial-format shape Step 3 pinned down). Fails because `MaskInput` doesn't yet call `useMaskedCaret`. Commit type: `test`.

### Step 6 — GREEN: wire MaskInput to useMaskedCaret

- **Files**:
  - `packages/ui/src/components/MaskInput/MaskInput.tsx` (modify)
- **What**: `MaskInput` calls `useMaskedCaret({ mask, alwaysShowMask, value, onChange })`, renders the hook's formatted value, and calls the hook's `handleChange` on the input's native `onChange`, then calls the consumer's `onChange` prop with the formatted value (matching how the existing `MaskedInput`/`MuiPhoneField` consumers expect `onChange` to deliver the masked string — confirm against `apps/web/components/UI/MaskedInput/MaskedInput.tsx`'s current `onChange` contract if any ambiguity arises during implementation, but do not modify that file). Passes Step 5's test. Commit type: `feat`.

### Step 7 — RED: caret auto-advance past fixed characters while typing

- **Files**:
  - `packages/ui/src/components/MaskInput/MaskInput.test.tsx` (modify)
- **What**: Test that after typing a digit immediately before a fixed character (e.g. entering the 3rd digit of the phone area code, `"555"`), the caret lands after the auto-inserted `)` and space, not immediately after the digit — assert via `input.selectionStart`. Cover at least one case per mask (phone: skip over `)`/`-`; date: skip over `/`). Fails because Step 6 doesn't manage the caret yet. Commit type: `test`.

### Step 8 — GREEN: caret auto-advance implementation

- **Files**:
  - `packages/ui/src/hooks/useMaskedCaret/useMaskedCaret.ts` (modify)
  - `packages/ui/src/components/MaskInput/MaskInput.tsx` (modify, if the hook needs to expose a ref/effect the component must attach)
- **What**: Add "snap to nearest editable position" and imperative `input.setSelectionRange` application (via `useLayoutEffect`, guarded by an "is this input focused" check) so the caret always lands on the next editable digit slot after a keystroke, skipping fixed characters. Passes Step 7's test. Commit type: `feat`.

### Step 9 — RED: delete across fixed characters (Backspace)

- **Files**:
  - `packages/ui/src/components/MaskInput/MaskInput.test.tsx` (modify)
- **What**: Test that pressing Backspace with the caret immediately after a fixed character (e.g. caret right after `) ` in `"+1 (555) "`) deletes the preceding digit, not the fixed character, and the fixed character reappears once a digit exists to its left. Cover phone and date masks. Fails against current implementation. Commit type: `test`.

### Step 10 — GREEN: delete-across-fixed-character implementation

- **Files**:
  - `packages/ui/src/hooks/useMaskedCaret/useMaskedCaret.ts` (modify)
- **What**: Handle the Backspace/Delete key path (via `onKeyDown` or by diffing previous/next raw value in `handleChange`, matching whichever approach the earlier `handleChange` design already established) so deletion always removes the nearest editable digit to the left/right of the caret, re-formats, and repositions the caret past the resulting fixed characters. Passes Step 9's test. Commit type: `feat`.

### Step 11 — RED: paste and range-select-then-type

- **Files**:
  - `packages/ui/src/components/MaskInput/MaskInput.test.tsx` (modify)
- **What**: Two test cases:
  1. Pasting a full formatted phone number (e.g. `"(555) 123-4567"`, with the fork's non-digit characters included) into an empty `MaskInput` results in the correctly masked value `"+1 (555) 123-4567"` (paste strips non-digits same as typed input).
  2. Selecting a range of already-entered digits (via `input.setSelectionRange`) and typing a replacement digit overwrites only the selected digits, not the whole value.
  Use `fireEvent.paste` with a `clipboardData` object (jsdom) for case 1, and `userEvent.setup()` with keyboard input after a programmatic selection for case 2. Both fail against the current implementation. Commit type: `test`.

### Step 12 — GREEN: paste and range-select handling

- **Files**:
  - `packages/ui/src/hooks/useMaskedCaret/useMaskedCaret.ts` (modify)
- **What**: Ensure paste flows through the same `handleChange` path as typed input (matching the fork's approach of not needing a separate `onPaste` handler), and that range selection is respected when computing the new raw value (replace the selected range rather than always inserting/appending). Passes Step 11's tests. Commit type: `feat`.

### Step 13 — RED: focus/blur behavior and alwaysShowMask

- **Files**:
  - `packages/ui/src/components/MaskInput/MaskInput.test.tsx` (modify)
- **What**: Test cases:
  - On focus with `alwaysShowMask={false}` and an empty value, the field shows nothing (or the bare mask skeleton — pin down exact behavior) and caret positions at the first editable slot.
  - On blur with `alwaysShowMask={false}` and no digits entered, the value clears back to empty (matches fork behavior of clearing an unfilled mask on blur).
  - With `alwaysShowMask={true}`, the placeholder skeleton (e.g. `"+1 (___) ___-____"`) is visible on mount even before focus, and remains after blur regardless of fill state.
  - `onFocus`/`onBlur` consumer callbacks still fire.
  Fails against current implementation (no focus/blur formatting logic yet). Commit type: `test`.

### Step 14 — GREEN: focus/blur + alwaysShowMask implementation

- **Files**:
  - `packages/ui/src/hooks/useMaskedCaret/useMaskedCaret.ts` (modify)
  - `packages/ui/src/components/MaskInput/MaskInput.tsx` (modify — wire `onFocus`/`onBlur`)
- **What**: Implement the focus-in reformat (show skeleton, position caret at first editable slot) and blur-out clear-if-empty behavior, gated by `alwaysShowMask`. Passes Step 13's tests. Commit type: `feat`.

### Step 15 — REFACTOR: clean up useMaskedCaret and MaskInput

- **Files**:
  - `packages/ui/src/hooks/useMaskedCaret/useMaskedCaret.ts` (modify)
  - `packages/ui/src/components/MaskInput/MaskInput.tsx` (modify)
  - `packages/ui/src/components/MaskInput/MaskInput.styles.ts` (modify)
- **What**: With all behavior tests green (Steps 1–14), refactor for clarity: extract named helper functions for mask parsing / editable-position lookup / raw-value diffing if they've grown inline, remove duplication between the phone/date/code code paths, tighten types (no `any`, discriminated handling where useful), add doc comments explaining the caret algorithm's provenance (reference to the MIT-licensed fork, not a copy). Run the full `MaskInput.test.tsx` + `useMaskedCaret.test.ts` suite after every change to confirm nothing regresses (subject to the test-runner gap in Open Questions). Commit type: `refactor`.

### Step 16 — Storybook story for all 3 masks

- **Files**:
  - `packages/ui/src/components/MaskInput/MaskInput.stories.tsx` (new)
- **What**: CSF3 story file, `title: 'Atoms/MaskInput'`, `tags: ['autodocs']`. Three controlled stories (`Phone`, `Date`, `SixDigitCode`), each following the `TextField.stories.tsx` `Phone`/`PhoneFax` pattern — local `useState` + `<MaskInput mask="..." value={value} onChange={(e) => setValue(e.target.value)} />`. Add a fourth `AlwaysShowMask` story demonstrating the skeleton-visible-on-mount behavior. No `argTypes` complexity needed since the component takes plain string/boolean props. This is the primary venue for real-browser caret verification (see Testing Strategy). Commit type: `feat` (or `docs`, since it's Storybook-only — use `feat` since it's part of the deliverable, not documentation of existing behavior).

### Step 17 — Verification pass: types, lint, build

- **Files**: none (no code changes expected; fix-up only if any check fails)
- **What**: Run `npm run types:check` and `npm run lint` from `packages/ui/`, and `npm run build` (`tsup`) to confirm `MaskInput` and `useMaskedCaret` are emitted correctly in the built `dist/index.d.ts`/`dist/index.js`/`dist/index.cjs` output and that no `'use client'` directive leaked in. If any check fails, fix in a follow-up commit of the appropriate type (`fix`/`chore`) rather than amending prior commits. This step produces a commit only if fixes were needed.

---

## Files Affected

| File | Action | Description |
|------|--------|--------------|
| `packages/ui/src/components/MaskInput/MaskInput.tsx` | Create | Headless `forwardRef` component rendering a bare `<input>`, wired to `useMaskedCaret` |
| `packages/ui/src/components/MaskInput/MaskInput.styles.ts` | Create | Emotion `styled('input')` wrapper, minimal/unstyled |
| `packages/ui/src/components/MaskInput/index.ts` | Create | Barrel: `export { MaskInput, type MaskInputProps } from './MaskInput';` |
| `packages/ui/src/components/MaskInput/MaskInput.test.tsx` | Create | Render, ref-forwarding, typing, caret auto-advance, delete-across-fixed, paste, range-select, focus/blur, alwaysShowMask tests |
| `packages/ui/src/components/MaskInput/MaskInput.stories.tsx` | Create | Storybook stories: Phone, Date, SixDigitCode, AlwaysShowMask |
| `packages/ui/src/hooks/useMaskedCaret/useMaskedCaret.ts` | Create | Mask parsing, formatting, caret positioning, delete/paste/range-select logic |
| `packages/ui/src/hooks/useMaskedCaret/useMaskedCaret.test.ts` | Create | Pure formatting behavior tests for the 3 masks, over-length truncation, alwaysShowMask skeleton |
| `packages/ui/src/components/index.ts` | Modify | Add `export { MaskInput, type MaskInputProps } from './MaskInput';` |
| `packages/ui/src/index.ts` | Modify | Add re-export for `MaskInput`/`MaskInputProps` and `useMaskedCaret`/its option/result types |

---

## Testing Strategy

- **Unit tests (Vitest, `packages/ui/src/hooks/useMaskedCaret/useMaskedCaret.test.ts`)**: pure formatting logic for the 3 masks — digit-in/masked-out mapping, partial input, over-length truncation, `alwaysShowMask` skeleton rendering. These are the most reliable tests since they don't depend on jsdom's selection APIs.
- **Unit tests (Vitest, `packages/ui/src/components/MaskInput/MaskInput.test.tsx`)**: render + ref forwarding + `className` passthrough (Step 1); typed-input masked output (Step 5); caret auto-advance past fixed characters via `input.selectionStart` assertions (Step 7); Backspace/Delete across fixed characters (Step 9); paste via `fireEvent.paste` with `clipboardData` (Step 11); range-select-then-type via `input.setSelectionRange` + `userEvent` keyboard input (Step 11); focus/blur formatting and `alwaysShowMask` (Step 13). All rendered via `renderWithTheme`, never raw `render`.
- **jsdom limitation, called out explicitly**: jsdom supports `setSelectionRange`/`selectionStart`/`selectionEnd` natively, so the caret-position assertions above are executable, but jsdom's fidelity for real browser quirks (IME composition, autofill, mobile virtual keyboards, some paste event nuances) is limited. Per the parent pre-plan, these edge cases are expected to only surface in a real browser.
- **Manual verification (Storybook, port 6006, run by the user — not the AI)**: `npm run dev:storybook`, navigate to `Atoms/MaskInput`, and manually verify all 4 stories (Phone, Date, SixDigitCode, AlwaysShowMask) in a real browser: typing, deleting mid-string, deleting across fixed characters, pasting a full value, range-selecting and retyping, tabbing focus in/out, and — if convenient — a mobile device emulator for on-screen keyboard behavior. This is the primary confidence check for caret correctness per the parent pre-plan's stated mitigation strategy; unit tests are a regression net, not the final word on caret behavior.
- **Mock boundaries**: none needed — `MaskInput` and `useMaskedCaret` have no external dependencies (no network, no database, no other services). Tests exercise real DOM APIs via jsdom, no mocking required beyond what React Testing Library provides.
- **No integration tests**: this task does not touch any consumer (`MaskedInput`, `MuiPhoneField`) — that's MLID-2769. No API routes are involved.

---

## Security Considerations

- **Input validation**: not applicable at this layer — `MaskInput` is a client-side formatting primitive, not a data boundary. No server-side validation is introduced or bypassed by this task.
- **Authorization**: not applicable — this is a UI component with no permission surface.
- **PHI handling**: `MaskInput` will, once wired up in MLID-2769, carry phone numbers and dates of birth (PHI-adjacent) through its `value`/`onChange`. In this task, that only matters for test/story hygiene: do not log input values (no `console.log`/`logger` calls in the component, hook, tests, or stories), and use obviously-fake sample values (e.g. `"555-123-4567"`-style test fixtures) in tests and stories, never real-looking production data.

---

## Implementation Findings (2026-07-09, during build)

- **Default output pads unfilled editable positions with `_` (matches the fork).** The fork's `maskPlaceholder` defaults to `"_"` (`InputMask.defaultProps`), and consumers depend on that output shape: `MuiPhoneField.tsx` documents "an empty mask renders as `+1 (___) ___-____`" and `MuiPhoneField.test.tsx` feeds `"+1 (___) ___-____"` as a real value. **Decision (confirmed with user 2026-07-09): `MaskInput` hardcodes the `_` placeholder rendering as a fixed, non-configurable default** so its formatted output is identical to today's fork output, keeping MLID-2769 a pure internal swap. This **supersedes the Step 3 / Step 13 examples below** that assumed no placeholder padding. Corrected expectations:
  - phone, focused/empty → `+1 (___) ___-____`; partial `"555"` → `+1 (555) ___-____`; full `"5551234567"` → `+1 (555) 123-4567`
  - date, focused/empty → `__/__/____`; partial `"0115"` → `01/15/____`; full `"01152026"` → `01/15/2026`
  - code, focused/empty → `______`; partial `"123"` → `123___`; full `"123456"` → `123456`
  - `alwaysShowMask` only governs whether the empty skeleton shows when the field is **unfocused and empty** (true = skeleton always visible; false = clears to `""` on blur when no digits entered). It does NOT govern the `_` padding of the filled/focused state — that padding is always present.
- **`maskChar`/`maskPlaceholder` stay off the public prop surface** (per Resolved Decision 3) — the `_` behavior above is internal/fixed, not exposed as a prop.
- **`MaskedInputExtended` is NOT a fork consumer** — it uses `cleave.js`, so it is out of scope for this component and for MLID-2769. The relevant fork consumers are `MaskedInput` (via `StyledInput` child) and `MuiPhoneField` (via `PhoneMaskAdapter`, bare `<input>` in MUI's `inputComponent` slot).
- **Approach: faithful trimmed port of the fork's `MaskUtils` + component logic** (digit-only `9` placeholder, fixed `maskPlaceholder = '_'`, no `beforeMaskedStateChange`, no array masks). Porting the proven caret algorithm minimizes caret-edge-case risk versus reinventing it. The fork is MIT-licensed (© 2016 Nikita Lobachev) — reference/provenance noted in the hook's doc comment.

## Resolved Decisions (2026-07-09)

1. **Test runner — write vitest files, do NOT wire up a runner.** `packages/ui` has 37 existing `*.test.tsx` files written against vitest imports, but no `vitest.config.ts`, no `vitest`/`@vitest/*` in `package.json` devDependencies, no `test` script, and `packages/ui` is not in `turbo run test`. **Decision:** write `MaskInput.test.tsx` and `useMaskedCaret.test.ts` following that existing vitest convention so they are ready to run once a runner is eventually wired up, but **do not** wire up the runner as part of this task — no `npm install`, no `vitest.config.ts`, no `test` script, no `turbo.json` change. Wiring up the runner is out of scope for MLID-2768. Consequence: the RED/GREEN steps below still get their test files written, but the tests cannot be executed/confirmed-green in this task. **Verification for this task is Storybook + real browser only** (see Testing Strategy). Treat the "run tests" instructions in the steps as "write the test file"; do not attempt to execute them.
2. **Branch structure — parent integration branch.** `feature/MLID-2742-inhouse-masked-input` is branched off `develop` and acts as the integration branch for all 3 subtasks. `feature/MLID-2768-maskinput-component` is branched off `feature/MLID-2742-inhouse-masked-input` and merged back into it (no per-subtask PR). The single PR is `feature/MLID-2742-inhouse-masked-input` → `develop` after all 3 subtasks land. MLID-2769 and MLID-2770 branch off the same 2742 branch in sequence.
3. **`maskChar`/`maskPlaceholder` excluded — confirmed.** Our code does not use them, so there is nothing to port. They are omitted entirely from `MaskInputProps`. If MLID-2769 later uncovers a hidden usage, adding them then is an acceptable follow-up.
