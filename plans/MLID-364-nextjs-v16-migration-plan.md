# MLID-364 — Next.js v16 Migration: Real Implementation Plan

> This is the **real, production migration plan** for moving LISA (`apps/web`) from Next.js 14 / React 18 to **Next.js 16 / React 19**. It is **not** a proof of concept. It builds directly on a completed PoC (summarized in "Where we are coming from" below) instead of restarting the investigation from scratch.
>
> **Objective:** have LISA **successfully and durably running on Next.js 16 in production** — the same green bar the team relies on every day: `npm run build` clean, `npm run dev` clean, **the full test suite green**, lint passing, and all runtime surfaces verified.
>
> **Companion document:** `MLID-364-nextjs-v16-migration-implementation-history.md` — the running log (live status, per-step notes, errors and fixes). This plan is the stable strategy; the history file is the source of truth for "where we are."

---

## Where we are coming from (pre-context)

Before this plan, a **proof of concept** proved the migration is technically reachable. The PoC took the **hardest path first** — a single **direct 14 → 16 / React 18 → 19 jump** (no staging) — run in an **isolated git worktree** so breakage was free. It reached a **clean, error-free app in both `npm run build` and `npm run dev`**, with the two hard third-party blockers resolved and browser-verified.

### What the PoC already achieved (carried into this epic)

- **Production build is GREEN** — `npm run build:web` completes with no errors on **Next.js 16.2.9 (webpack) + React 19.2.7**. All **64** React 19 / Next 16 type errors were fixed, and `next.config.mjs` was cleaned of every removed/renamed key.
- **Dev runtime is clean and browser-verified** — `npm run dev:web` boots clean; **Google login works end-to-end** and **masked/phone inputs render correctly**.
- **Both hard blockers resolved:**
  - **NextAuth v4 → Auth.js v5** (`next-auth@5.0.0-beta.31`, pinned exact) — fixed the async `cookies()`/`params` incompatibility. Real blast radius **~336 files**.
  - **`react-input-mask` → `@mona-health/react-input-mask@3.0.3`** — removed the React-19-incompatible `findDOMNode`. Not a pure drop-in (children + ref API changed). **PE follow-up (Anatoliy): production must not depend on a third-party fork → port to an in-house `@repo/ui` masked-input component (Option B, own component). Tracked in MLID-2742 (5 pts; ~3–5 days). Out of scope for this epic; may gate the release or be a fast-follow — PE/PO call.**
- **Mechanical migrations completed** — async `params` migrated across **71 source files**; `middleware` → `proxy` rename; `react-select` 5.8 → 5.10.2 (React 19 type collapse).
- **Lint moved to ESLint 9 flat config** for the LISA scope (`apps/web`, `packages/ui`, `packages/eslint-config`, `apps/kiosk`, root) — **config-only, zero source files touched**.
- **A render-loop bug was found and fixed during browser verification** — `useFetchData` returned unstable `customFetch`/`refetch` when a consumer passed an inline callback, causing "Maximum update depth exceeded" on `PatientProvider`-backed routes (e.g. `/patient/[id]/details`). Fixed by holding the callback in a ref. This fix is included in the epic branch's starting commit.

### The PoC stages, for reference (basic description of each step taken)

| Stage | Basic description |
|---|---|
| **S0 — Setup & isolation** | Create the sibling worktree + spike branch off develop; install a Next 14 baseline; confirm the app and the 3 target routes render on 14 (the "before" reference). |
| **S1 — Core upgrade codemod** | Run the Next upgrade codemod: bump `next`→16, `react`/`react-dom`→19, config cleanups, `middleware`→`proxy`, lint migration. |
| **S2 — React 19 dependency audit** | Inspect actual peer ranges against React 19; identify libraries needing bump/replace (`react-input-mask`, `next-auth@4`, `react-select`, `react-modal`, `react-collapsible`, …). |
| **S3 — Async params sweep** | Run the async-request-api codemod across ~147 dynamic route files; 71 source files transformed. |
| **S4 — Builder + lint config** | Add `--webpack` to dev + build (keep webpack, defer Turbopack); start the ESLint flat-config migration. |
| **S5 — Build GREEN** | Fix all React 19 / Next 16 build + type errors (64) until `npm run build` is clean; clean `next.config.mjs` keys. |
| **S6 — Dev GREEN + browser verify** | Fix runtime-only React 19 breaks; user navigates `/intakes`, `/admin`, `/orders-tracker` with no console/runtime errors. |
| **S6b — Test-suite consistency** | (Started, not completed in PoC) Update async-params page tests and route/test mocks. **This is the main carry-over into the real plan.** |
| **S6c — ESLint 8→9 flat config (LISA scope)** | Move `apps/web` + related packages to ESLint 9 flat config, config-only. |
| **S7a — NextAuth v4 → Auth.js v5** | Migrate authentication to Auth.js v5 so it works on Next 16's async request APIs. |
| **S7b — Replace react-input-mask** | Swap to `@mona-health/react-input-mask` to remove `findDOMNode`. |
| **S8 — Findings & teardown** | Record the verdict; keep the branch (teardown intentionally skipped so the change-set is preserved). |

The PoC evidence branch (`spike/MLID-364-next16-direct-poc`) is preserved at origin. This epic branch was created **from that PoC HEAD**, so all of the above is already in place — the real plan continues from here rather than repeating it.

---

## Strategy (agreed for the real migration)

1. **Build on the PoC, do not restart.** The concrete change-set (Auth.js v5, react-input-mask fork, react-select bump, async params, `next.config.mjs`, the 64 type fixes, ESLint 9 config, the `useFetchData` fix) is the deliverable. Re-deriving it would waste the PoC investment.

2. **Branch model — epic with feature branches.** The work is epic-sized (the auth migration alone touched ~336 files). It runs on the epic branch `epic/MLID-364-nextjs-v16-migration`. Individual work items are feature branches cut **off the epic** and merged **into the epic with no per-feature pull request**. The only pull request is the final **epic → develop**.

3. **Execution timeline:**
   1. **Graduate the branch** — promote the spike into the epic branch off the PoC HEAD. **(Done.)**
   2. **Address the test suite** — the long part; close the TDD exception the PoC used (see below).
   3. **Sync develop — then repeat daily.** The first sync is intentionally deferred until after the test work, because the branch is already several days behind a fast-moving develop and the first sync will be the largest. From creation of the epic onward we do **not** want to stay stale, so once we start syncing it becomes a **daily** merge.

4. **Sync discipline (every develop merge).** Merge only — **never rebase**. Always bring local develop current first, then merge local develop into the epic. Each merge is treated as a checklist, not just text-conflict resolution: when develop introduces new code, apply the same transforms the migration relies on. Watch for:
   - New dynamic-route files (`[param]/route.ts`, `[param]/page.tsx`) still using **synchronous `params`** → apply the async-params transform.
   - New `getServerSession(authOptions)` / `authOptions` imports → convert to `auth()` from `@/auth`.
   - New `react-input-mask` imports → point to `@mona-health/react-input-mask` and adjust the children/ref API.
   - New `useRef()` with no initial argument → give it an explicit initial value (React 19).
   - New global `JSX.Element` usages → import `JSX` from `react`.
   - New `NextRequest.ip` usages → header fallback.
   - New `.eslintrc`-style config in the migrated packages → convert to flat config.

5. **The beta dependency is not a blocker.** `next-auth@5.0.0-beta.31` is beta only because stable v5 has not shipped — a timeline we do not control. We pin the exact version and document the risk for the team; we do **not** wait for stable. Swap to stable later when it exists.

6. **Production bar — close the TDD exception.** The PoC used `CLAUDE.md`'s documented spike exception to skip TDD. The real migration must meet the normal bar: the test suite is written/updated so `npm test` is green. This is the immediate next phase.

7. **Open team decision — landing strategy.** The PoC proves Next 16 is **reachable**; it does not settle **how to land it without freezing the team**: one large epic→develop pull request versus a staged rollout. This is a team-velocity / merge-conflict decision, not a technical one, and should be confirmed with the team before the epic grows large. Building on this branch is compatible with either.

---

## Continuation steps of the real plan

Grouped into phases. Each R-item is a candidate feature branch off the epic. Live status, ordering changes, and per-step notes live in the implementation-history file.

### Progress (as of 2026-07-03)

**Phase 1 (test suite) is COMPLETE — the web test project is green.** jsdom **6,100** tests pass, node **12,601** pass, `tsc --noEmit` clean. Only leftover is the pre-existing V8 worker crash in `jobs/analytics-process-intake-overview/export` (fails to *run*, not migration-caused).

| Step | Status |
|---|---|
| R1 — Auth.js v5 test mocks | ✅ `6f101fb56` — ~156 files `getServerSession`→`auth`; global `@/auth` jest mock (moduleNameMapper); callbacks extracted to `callbacks.ts` for direct unit testing |
| R2 — Async-params page tests | ✅ `d0d92007b` — `tests/react/asyncParams.ts` `fulfilledParams()` (React 19 `use()` reads a fulfilled thenable synchronously → no Suspense/async rewrites); global `next-auth/react` jest mock |
| R3 — react-input-mask test mocks | ✅ mostly resolved by the S7b source swap (0 `findDOMNode` project-wide); one leftover mock (`MuiPhoneField.test`) fixed in `d12040c9d` |
| R4 — RootLayout test | ✅ folded into the jsdom-fix batch (`d12040c9d`) — React 19 changed the DOM-nesting message → console.error spy recursion; suppressed cleanly |
| R5 — API-route param mock type accuracy | ⬜ optional, not done (harmless under `isolatedModules`) |

**`d12040c9d` also closed the 9 CI-scoped jsdom failures** (React 19 component-call arg, async-flush timing, MUI-wizard effect timing, async server component) **and shipped a real runtime fix** — `react-simple-toasts` 6.0.0→6.1.0 (see carry-forward bugs).

**CI-scope finding (matters for the epic PR).** The PR workflow runs `jest --ci --changedSince=origin/develop --shard=N/4` (NOT the whole suite) plus a whole-app `types:check`. On the epic PR the changed graph selects **~352 of 1,264** test files; every suite we fixed was in that set and would have blocked the merge. (The `@/auth` hub effect is neutralized because jest maps `@/auth` to the mock, which is why it's 352 and not ~all.)

**Branch state.** All Phase-1 work is merged into `epic/MLID-364-nextjs-v16-migration` (fast-forward; epic HEAD `d12040c9d`, **3 commits ahead of `origin/epic`, not yet pushed**). `feature/MLID-364-auth-v5-test-mocks` is preserved. **Next: R6 first develop sync** — plan is a throwaway branch off the epic to attempt the develop merge and resolve conflicts, then apply to the epic.

### Phase 1 — Test suite (close the TDD exception)

| Step | Description |
|---|---|
| **R1 — Auth.js v5 test mocks** | Migrate the ~156 test files still mocking NextAuth v4 (`getServerSession`) to the Auth.js v5 `auth()` pattern. Start here: getting the shared mock helper right unblocks the most suites at once. |
| **R2 — Async-params page tests** | Restructure the ~130 `[id]/page.test.tsx` tests for React 19 async `use()`: pass `params={Promise.resolve({...})}` **and** wrap the async render in `act()` so content resolves. Per-suite work (ai-chat-settings, locations, drug-admin-panel drugs, insurance-payors guidelines, insurance-plans, …). |
| **R3 — react-input-mask test mocks** | Update the ~54 component/page tests that mock the old `react-input-mask` to the `@mona-health/react-input-mask` fork (children as element, `ref` instead of `inputRef`). |
| **R4 — RootLayout test** | Investigate and fix `app/layout.test.tsx`: `reactRender is not a function` + `RangeError: Maximum call stack exceeded` under React 19. |
| **R5 — API-route param mock type accuracy (optional)** | The ~69 API-route tests pass synchronous `params`; harmless today (`isolatedModules` transpiles without type-checking). Update only for type accuracy if we want the mocks to match the async handler signatures. |

**Phase 1 exit:** `npm test` green across `apps/web`.

### Phase 2 — Continuous develop sync

| Step | Description |
|---|---|
| **R6 — Daily develop sync** | Recurring. First run after Phase 1 (largest merge); daily thereafter. Apply the sync-discipline checklist above on every merge. Never rebase. |

### Phase 3 — Everything current (hardening)

| Step | Description |
|---|---|
| **R7 — Lint: migration-scoped only** | **Scope decision (Option A, 2026-07-06, PO): MLID-364 is a Next 16 upgrade to production standard, not a lint cleanup.** In scope: only **real ESLint errors in folders that were already linted before the migration** (mainly `apps/web`) — i.e. anything the migration itself broke. **Out of scope (deferred, separate tech-debt ticket if the business wants it):** the ~800 pre-existing issues in the never-linted folders (keep ignoring them) AND the ~248 `apps/web` warnings — do **not** re-promote the warn-downgraded rules or chase warnings; leave rule strength as landed. `apps/api` / `apps/mcp` flat-config migration = also a separate-ticket decision, not required for this epic. Net effect: if `apps/web` lint reports 0 errors (only warnings), R7 is effectively done. **Measured 2026-07-06 on epic `c87f4084d`: `apps/web` `npm run lint` = 250 problems, 0 errors / 250 warnings, exit 0. Zero in-scope errors → R7 is complete under Option A; no code changes.** |
| **R8 — Storybook React 19** | **Verified working 2026-07-06 on epic `c87f4084d`.** `apps/storybook` is Storybook 10.2.8 and already resolves **React `19.2.7`** at runtime (workspace-hoisted), despite its `package.json` still declaring `react: ^18`. `npm run build:storybook` completes successfully (built in ~39s, exit 0; only a Vite chunk-size hint + a Turbo `outputs`-key note, both cosmetic). No React-19 breakage — the design system is MUI 7 + Emotion; the libraries that broke under React 19 (`react-input-mask`, `react-simple-toasts`) live in `apps/web`, not `@repo/ui`. **Only residue = cosmetic:** bump the stale `react`/`react-dom` `^18 → ^19` declaration in `apps/storybook/package.json` (optional honesty fix, not yet done). Build compiles stories but does not render them in a browser, so a runtime-only render caveat remains (low risk); a quick browser pass of a story or two closes it. |
| **R9 — Turbopack migration** | The PoC kept webpack (`--webpack`). Moving webpack → Turbopack was split out into its own ticket: **MLID-2740** (Task, Sprint 31, 5 pts, *relates to* MLID-364; MLID-364 has a tracking comment). **Out of scope for this epic** — not a blocker (webpack build is green); it re-expresses the custom `webpack()` config (IgnorePlugin / NormalModuleReplacementPlugin / externals for `ssh2`-SFTP, Application Insights, Node built-ins) in Turbopack's model. |
| **R10 — Runtime verification sweep** | Broaden beyond the PoC's spot checks: confirm `react-pdf` renders under Next 16's forced SWC minify; verify `react-modal@3.16.1` and `react-collapsible@2.10.0` (React ≤18 peers) at runtime; do a route-by-route pass. |
| **R11 — pages/api test-route smell** | `pages/api/**/*.test.ts` (29 files) are built as live API routes because `pageExtensions: ['ts','tsx']`. **Pre-existing** (predates the migration; the v5 ESM build only surfaced it). Split out into its own ticket: **MLID-2741** (Task, Sprint 31, 3 pts, *relates to* MLID-364; MLID-364 has a tracking comment). **Out of scope for this epic** — not a blocker (build is green). Fix = move test files out of `pages/` or narrow `pageExtensions` so `.test.`/`.spec.` no longer match. |

### Phase 4 — Land

| Step | Description |
|---|---|
| **R12 — Team decisions** | Confirm: beta-dependency risk acceptance; the one-time session-cookie invalidation (`next-auth.*` → `authjs.*`, forced re-login on deploy) communication; and the landing strategy (single epic PR vs staged). |
| **R13 — Epic → develop pull request** | Write the PR document; land the epic. |

### Known carry-forward bugs (found during PoC / verification)

- **`useFetchData` render loop — FIXED** (in the epic's starting commit). Latent bug shared with develop; already resolved here.
- **`react-simple-toasts` 6.0.0 broke under React 19 — FIXED (bumped to 6.1.0, `d12040c9d`, user-approved).** 6.0.0 calls the removed `ReactDOM.render`, so **every toast crashed app-wide under React 19** (surfaced as `reactRender is not a function`). 6.1.0 detects React 18+ and uses `createRoot`. **Required PRODUCTION fix, not test-only** — belongs in the migration change-set; announce to the team. (The PoC's S2 audit mislabeled it compatible because its peer range is permissive `>=16.8.0` — same trap as react-input-mask.)
- **Feature-flags 401 in the PHI-masking path — pre-existing, deferred.** `getAllFeatureFlags` does a cookie-less internal `fetch` to the session-gated `/api/feature-flags`, which returns 401; the error is caught and masking falls back to defaults (non-fatal, request still 200). Not caused by the upgrade (the feature-flag code is byte-identical to develop). Fix direction: call `FeatureFlagService` directly server-side instead of the HTTP round-trip. Candidate **separate bug ticket**, not migration work.

---

## Authorization policy

- Dependency changes (npm install, codemods, library swaps) remain **gated**: nothing is installed or a library swapped without explicit approval. New environment variables introduced by any change must be announced so they can be wired into Azure manually.

---

## Team steps to upgrade their environment after merge

Once MLID-364 is merged, `develop` runs **Next.js 16 / React 19 / Auth.js v5**. Four major dependencies changed (`next`, `react`, `react-dom`, `next-auth`) and `react-input-mask` was replaced, so a plain `git pull` is **not** enough — stale `node_modules` and build caches will throw confusing errors. Every developer must reinstall and clear caches once.

**What changed:** the dependency bumps live in `apps/web`, but React 19 is hoisted to the repo root, so **all apps** (`web`, `storybook`, `kiosk`) resolve React 19. Reinstall once at the **repo root**, not per app.

### Steps

1. **Stop any running dev server**, then update develop:
   ```
   git checkout develop
   git pull
   ```

2. **Check the Node version** — Next 16 needs Node 20+:
   ```
   node -v
   ```
   If it is below 20, upgrade (for example `nvm install 20 && nvm use 20`).

3. **Clean reinstall from the repo root** (workspaces install together):
   ```
   npm ci
   ```

4. **Clear stale build caches:**
   ```
   rm -rf apps/web/.next .turbo
   ```
   Windows PowerShell equivalent:
   ```
   Remove-Item -Recurse -Force apps/web/.next, .turbo
   ```

5. **Rebuild the shared packages** (their `dist/` is gitignored, so it can be stale after a pull):
   ```
   npm run build --workspace=@repo/db
   npm run build --workspace=@repo/ui
   ```

6. **Start dev:**
   ```
   npm run dev:web
   ```

7. **Sign in again** — Auth.js v5 renamed the session cookie, so you are logged out once. Just log back in.

### One copy-paste block

```
git checkout develop && git pull
npm ci
rm -rf apps/web/.next .turbo
npm run build --workspace=@repo/db
npm run build --workspace=@repo/ui
npm run dev:web
```

### Notes

- **No new environment variables** — keep your existing `apps/web/.env.local`.
- Reinstall at the **repo root**, not per app (React 19 is hoisted; web / storybook / kiosk pick it up together).
- If you still hit `@repo/db` / `@repo/ui` "missing export" type errors after this, re-run step 5 — that is the usual culprit.
