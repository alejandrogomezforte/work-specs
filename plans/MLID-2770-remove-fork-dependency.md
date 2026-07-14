# MLID-2770 — Remove `@mona-health/react-input-mask` dependency + type shim

## Task Reference

- **Jira**: [MLID-2770](https://localinfusion.atlassian.net/browse/MLID-2770)
- **Story Points**: 1 (parent MLID-2742 total; MLID-2770 is subtask 3 of 3 — the final one)
- **Branch**: `feature/MLID-2770-remove-fork-dependency`
- **Base Branch**: `feature/MLID-2742-inhouse-masked-input` (the parent integration branch, off `develop`, already pushed to origin, currently checked out). Merge this subtask branch back into it — **no per-subtask pull request.** After this task lands, the parent branch opens its **single** pull request: `feature/MLID-2742-inhouse-masked-input` → `develop`.
- **Status**: To Do
- **Parent task**: MLID-2742 (In-house masked-input component in `@repo/ui`) — a Task, not a formal epic; there is no epic branch.
- **Depends on**: MLID-2768 (headless `MaskInput` + `useMaskedCaret`, done) and MLID-2769 (swap `MaskedInput`/`MuiPhoneField` wrappers to `MaskInput`, done, merged into `feature/MLID-2742-inhouse-masked-input`). After MLID-2769, no source file in the repository imports `@mona-health/react-input-mask` at runtime or as a type — this task removes the now-unused dependency and its ambient type shim.
- **Commit format**: `[MLID-2770] - type(scope): description`, no `Co-Authored-By`.

---

## Summary

`@mona-health/react-input-mask` is no longer imported anywhere after MLID-2769 swapped `MaskedInput` and `MuiPhoneField` to the in-house `@repo/ui` `MaskInput`. This task removes the now-dead npm dependency from `apps/web/package.json` and the root lockfile, deletes the ambient type shim that only ever existed to type that dependency, verifies nothing regresses, and closes out the parent task with a final real-browser acceptance pass across all 3 masks (phone, date, 6-digit code) — performed by the user, not the AI.

---

## Codebase Analysis

- **Single direct dependency, no transitive consumers.** `@mona-health/react-input-mask` appears only in `apps/web/package.json` (`"@mona-health/react-input-mask": "3.0.3"`, in `dependencies`). It is not a dependency of `apps/api`, `apps/mcp`, `packages/ui`, `packages/eslint-config`, `packages/typescript-config`, or the workspace root. The lockfile has no other package depending on it, so it removes cleanly with no orphaned sub-dependencies to reconcile.
- **Workspace mechanics.** This is an npm-workspaces + Turborepo monorepo with a single root `package-lock.json` (no per-app lockfiles). The correct removal command, run from the repository root, is:
  ```
  npm uninstall @mona-health/react-input-mask -w apps/web
  ```
  This updates `apps/web/package.json` and the root `package-lock.json` in one operation.
- **Zero remaining references after MLID-2769.** Verified: no runtime `import ... from '@mona-health/react-input-mask'` and no type-only import remain in any source file.
  - `apps/web/components/UI/StyledInput/StyledInput.tsx` now uses a local `StyledInputProps` type (no fork import) — this was the decision recorded in MLID-2769's Resolved Decisions (a permissive local type, not the strict `React.InputHTMLAttributes`, to avoid the wider `isDisabled`/`error`-prop blast radius discovered mid-implementation).
  - `apps/web/components/UI/MaskedInput/MaskedInput.tsx` renders `@repo/ui`'s `MaskInput` directly and uses `MaskInputProps`-derived types.
  - `apps/web/components/UI/MUI/MuiPhoneField.tsx` renders `@repo/ui`'s `MaskInput` in `PhoneMaskAdapter`; its own public types (`MuiPhoneFieldProps<T>`) are plain React/react-hook-form types.
  - No `jest.mock('@mona-health/react-input-mask', ...)` remains anywhere (the one instance, in `MuiPhoneField.test.tsx`, was removed in MLID-2769 Step 4).
  - No `jest.config.js` `moduleNameMapper`/`transformIgnorePatterns` entry, no `next.config.mjs` `transpilePackages` entry, and no `tsup.config` reference the fork.
  - The shim's own exports (`InputState`, `Props`) have zero consumers left in the codebase.
- **Ambient type shim mechanics.** `apps/web/types/mona-health-react-input-mask.d.ts` is pulled into the TypeScript program only via `apps/web/tsconfig.json`'s blanket `include` glob (which matches `**/*.d.ts`) plus its `typeRoots: ["./types", ...]` entry. There is no barrel/index file that references it by name — ambient module declarations are never imported directly. Deleting this single file has no effect on the other 62 files under `apps/web/types/` (e.g. `next-auth.d.ts`, `node-stream-zip.d.ts`, `fuse.d.ts` all stay untouched, unrelated shims for unrelated packages).
- **`cleave.js` is a separate library and is out of scope.** `apps/web/package.json` also lists `cleave.js` (`^1.6.0`), used by `MaskedInputExtended` for currency/percent masks. It has no relationship to `@mona-health/react-input-mask` and must not be touched by this task.
- **Three prose comments mention the fork by name and will "dangle" after removal** (harmless, no functional impact — decide whether/how to tidy each, see steps below):
  1. `apps/web/components/UI/StyledInput/StyledInput.tsx` — a historical comment that also currently references "MLID-2770 removes [the ambient type shim]"; now slightly stale phrasing once this task lands (present tense "removes" should become past tense).
  2. `apps/web/components/UI/MaskedInput/MaskedInput.test.tsx` — a historical docblock note referencing the swap away from the fork. Harmless as-is; leave untouched (matches MLID-2769's own precedent of not chasing every historical comment).
  3. `packages/ui/src/hooks/useMaskedCaret/useMaskedCaret.ts` — an MIT-license **attribution** comment crediting the fork (© 2016 Nikita Lobachev) as the source the caret algorithm was ported from. **This must be KEPT, not removed.** Retaining provenance/attribution for MIT-licensed code that was ported into `useMaskedCaret` is correct and required regardless of whether the runtime dependency still exists — removing the npm package does not remove the licensing obligation for code that was derived from it.
- **Pre-existing, unrelated type errors on this branch.** `npm run types:check` currently reports `@repo/db` "has no exported member" errors (dictionary/googleDrive repositories) inherited from `develop`. These are not caused by this task, not introduced by MLID-2768/2769, and not this task's responsibility to fix. The verification step in this plan is written as "confirm no *new* type errors are introduced" against this known baseline, not "types are fully clean."
- **TDD applicability.** This task removes a dependency and deletes a file — it introduces no new logic, no new branch of behavior, and no new component/function. Per the project's documented TDD exception for "configuration changes... only if there's zero logic," this qualifies: there is no new unit test to write. The regression net is the existing test suite (which must stay green, especially the mask-consumer tests added/touched in MLID-2769) plus type-checking and a targeted grep proving no fork references remain (aside from the attribution comment, which is intentional).

---

## Implementation Steps

Each step is a single commit. Commit format: `[MLID-2770] - type(scope): description`.

### Step 1 — `chore`: delete the type shim and uninstall the npm package

- **Files**:
  - `apps/web/types/mona-health-react-input-mask.d.ts` (delete)
  - `apps/web/package.json` (modify — dependency removed)
  - `package-lock.json` (modify — regenerated by the uninstall command)
  - `apps/web/components/UI/StyledInput/StyledInput.tsx` (modify — optional comment tidy only, no code change)
- **What**:
  1. Delete `apps/web/types/mona-health-react-input-mask.d.ts`.
  2. **Pause and get explicit user go-ahead before running `npm uninstall`** — per the project rule that npm install/uninstall is never run without explicit permission. This command mutates `apps/web/package.json` and the root `package-lock.json`, so confirm before executing (or let the user run it themselves):
     ```
     npm uninstall @mona-health/react-input-mask -w apps/web
     ```
     Run from the repository root.
  3. Optionally tidy the now-stale comment in `StyledInput.tsx` that references "MLID-2770 removes [the shim]" in present tense — update it to past tense (e.g. "removed") or otherwise reflect that the shim is gone. This is a comment-only edit, no behavior change.
  4. Do **not** touch `packages/ui/src/hooks/useMaskedCaret/useMaskedCaret.ts`'s MIT attribution comment — it stays exactly as-is (see Codebase Analysis).
  5. Do **not** touch `cleave.js` or anything related to `MaskedInputExtended`.
  - Commit type: `chore` (dependency removal, no logic change). Suggested message: `[MLID-2770] - chore(deps): remove @mona-health/react-input-mask and its ambient type shim`.

### Step 2 — Verification pass: types, full test suite, reference grep

- **Files**: none expected (fix-up only if a check unexpectedly fails)
- **What**: From `apps/web/`, run:
  - `npm run types:check` — confirm the only remaining errors are the pre-existing, unrelated `@repo/db` "has no exported member" errors already present on this branch before this task (same count/shape as the baseline). Confirm **no new** type errors were introduced by removing the shim (i.e., nothing in the codebase was silently relying on the shim's permissive index signature — MLID-2769 already confirmed and resolved the one real dependent, `StyledInput`, before this task started).
  - Full Jest suite (or at minimum the targeted slice touched across MLID-2768/2769: `npx jest MaskedInput MuiPhoneField abbvie/page kedrion/page`, plus a full `npm run test` if time allows) — must stay green with zero test modifications in this task.
  - A grep for `mona-health` (case-insensitive) across the repository, confirming the only remaining hit is the intentional MIT attribution comment in `packages/ui/src/hooks/useMaskedCaret/useMaskedCaret.ts`. Any other hit means either the uninstall left a stray reference or a comment/doc was missed.
  - `npm run lint` on touched files (the `StyledInput.tsx` comment tidy, if done).
  - If any check fails or turns up an unexpected reference, fix in a follow-up commit of the appropriate type (`fix`) rather than amending Step 1's commit. This step produces a commit only if a fix was needed; otherwise it's a verification-only step with no diff.

### Step 3 — Real-browser acceptance verification (USER-run, not the AI)

- **Files**: none (manual verification only)
- **What**: This is the final acceptance check for the entire MLID-2742 parent task, not just this subtask — confirm all 3 masks behave correctly with the dependency fully removed and only the in-house `MaskInput` engine in play. Follow the same routes and caret checklist documented in MLID-2769's Testing Strategy / Affected Routes:
  1. **Date mask `99/99/9999` and 6-digit mask `999999` — GlobalSearch drawer** (easiest, reachable from any signed-in page):
     - Log in, land on any page other than `/`, click the global header search, type a nonsense name that matches no patient, click **"Create Patient's Profile"** in the empty-state dropdown.
     - **Date of Birth** field (placeholder `MM/DD/YYYY`): type digits, delete mid-string, delete across a `/`, paste a full value, range-select-and-retype, tab focus in/out. Confirm masked output (e.g. `01152026` → `01/15/2026`).
     - **WeInfuse ID (IV)** field (placeholder `123456`): type `123456`, confirm digit-only with no fixed characters, exercise the same caret checklist.
  2. **Phone mask `+1 (999) 999-9999` — `MuiPhoneField`** at `/intakes/[id]/review`:
     - Open an intake of type `spruceFax` or `eOrder`, click **"Review Documents"**.
     - On wizard step 1, click **"Create Patient"** mode, scroll to Contact Information → **"Mobile"** field: run the full caret checklist (type, delete mid-string, delete across `)`/`-`, paste `(555) 123-4567`, range-select-retype, tab focus).
     - Advance to wizard step 3 "Review Data", click the **"Order"** badge, expand "Prescribing Provider" → Contact Information → **"Provider Phone"** and **"Provider Fax"**: same caret checklist on both.
  3. **Cross-cutting checks for every field above**: confirm no React 19 console errors or warnings (no `findDOMNode`, no caret-jump-to-end, no unrecognized-attribute warnings); test on a mobile device emulator or real device for on-screen keyboard behavior; test browser autofill on the phone/date fields if convenient.
  - This step has no commit — it is a sign-off gate. Once the user confirms all 3 masks pass, the parent task MLID-2742 is ready for its single pull request (`feature/MLID-2742-inhouse-masked-input` → `develop`).

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/types/mona-health-react-input-mask.d.ts` | Delete | Ambient type shim for the fork; no longer needed, zero consumers |
| `apps/web/package.json` | Modify | Remove `@mona-health/react-input-mask` from `dependencies` |
| `package-lock.json` | Modify | Regenerated by `npm uninstall` to drop the removed package |
| `apps/web/components/UI/StyledInput/StyledInput.tsx` | Modify (optional) | Tidy a now-stale comment referencing this ticket in present tense; no behavior change |

Not modified (verified, explicitly out of scope):
- `packages/ui/src/hooks/useMaskedCaret/useMaskedCaret.ts` — MIT attribution comment for the ported caret algorithm is **kept**, not removed.
- `apps/web/components/UI/MaskedInput/MaskedInput.test.tsx` — historical docblock note left as-is.
- `apps/web/package.json`'s `cleave.js` entry and everything under `MaskedInputExtended` — unrelated masking library, out of scope.
- Any `apps/api`, `apps/mcp`, `packages/ui`, `packages/eslint-config`, `packages/typescript-config` manifests — the fork was never a dependency of these.

---

## Testing Strategy

- **Unit tests**: none new — this task has no new logic (dependency removal + file deletion only), which qualifies for the project's documented TDD exception for zero-logic configuration changes. The regression net is the existing suite from MLID-2768/2769, which must remain green with zero test modifications in this task.
- **Integration tests**: none new, same rationale. The targeted slice to re-run is `MaskedInput`, `MuiPhoneField`, `abbvie/page`, `kedrion/page` (the files touched across the parent task), plus a full `npm run test` pass if time allows.
- **Type-check**: `npm run types:check` from `apps/web/` — must show no new errors versus the pre-existing `@repo/db` baseline already present on this branch.
- **Reference grep**: case-insensitive search for `mona-health` across the repository — must return exactly one hit, the MIT attribution comment in `useMaskedCaret.ts`. Any other hit is a signal something was missed.
- **Manual verification (real browser, run by the user)**: full caret checklist (type, delete mid-string, delete across a fixed character, paste, range-select-retype, tab focus, mobile keyboard, no React 19 console errors) across all 3 masks at their real routes — GlobalSearch drawer (date + 6-digit) and the document-intake wizard (phone, both `PatientCreateMode` and `OrderForm`). This is the final acceptance gate for the entire MLID-2742 parent task before its single pull request opens.
- **Mock boundaries**: none needed for this task — no new code, no new boundary to mock.

---

## Security Considerations

- **Input validation**: not applicable — this task removes an unused dependency and a type-only shim; no validation logic changes anywhere.
- **Authorization**: not applicable — no permission surface is touched.
- **Feature flag gating**: none needed — this is dependency cleanup, not a user-facing feature or toggle.
- **PHI handling**: no PHI-adjacent code changes. The masks that carry phone numbers and dates of birth (`MaskedInput`, `MuiPhoneField`) were already reworked in MLID-2769 with no logging introduced; this task does not touch their runtime behavior at all, only the now-unused dependency underneath them.

---

## Resolved / Open Decisions

1. **npm uninstall requires explicit user permission before running.** Per project convention, the AI does not run `npm install`/`npm uninstall` without explicit go-ahead. Step 1 pauses at that point in the workflow — either the AI runs the uninstall after the user confirms, or the user runs it directly. Either way, `apps/web/package.json` and the root `package-lock.json` are the only files that command touches.
2. **Keep the MIT attribution comment in `useMaskedCaret.ts` — do not remove it.** Removing the npm dependency does not remove the licensing obligation for the caret algorithm that was ported from it. This comment stays untouched in this task and is explicitly called out as "not modified" in Files Affected.
3. **`@repo/db` type errors are a pre-existing, unrelated baseline — not this task's responsibility.** `npm run types:check` on this branch already shows dictionary/googleDrive repository export errors inherited from `develop`. This task's verification step confirms no *new* errors are introduced, not that the type-check is fully clean.
4. **`cleave.js` / `MaskedInputExtended` are out of scope.** They are a separate masking library with no relationship to `@mona-health/react-input-mask` and are not touched anywhere in the MLID-2742 parent task.
5. **Branch/merge mechanics mirror MLID-2768 and MLID-2769.** `feature/MLID-2770-remove-fork-dependency` branches off `feature/MLID-2742-inhouse-masked-input` and merges back into it directly, no per-subtask pull request. This is the final subtask — once it lands and the user completes the real-browser acceptance pass (Step 3), the parent task's single pull request opens: `feature/MLID-2742-inhouse-masked-input` → `develop`.
