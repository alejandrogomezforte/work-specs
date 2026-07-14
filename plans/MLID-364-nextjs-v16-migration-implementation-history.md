# MLID-364 — Next.js v16 Migration: Implementation History

> The **running log** for the real MLID-364 Next.js 16 / React 19 migration: live status, every step taken, errors found and how they were solved. This is the **single source of truth for "where we are."**
>
> **Companion document:** `MLID-364-nextjs-v16-migration-plan.md` — the stable strategy (objective, pre-context, agreed strategy, continuation steps). Read that for *what each step is*; read this file for *what actually happened*.
>
> This file lives only in the main folder (`docs\agomez\plans\`) and is never committed onto any code branch. All status/history updates happen here.

---

## Resume pointer

> **Phase 1 (test suite) is the active work. Epic branch created + pushed; PoC change-set carried over.**
>
> **Epic branch:** `epic/MLID-364-nextjs-v16-migration` — created off the PoC HEAD, **pushed to `origin/epic/MLID-364-nextjs-v16-migration`**, HEAD `7eca4504e`. Worktree: `C:\Users\alejandro.gomez\Dev\li-call-processor-nextjs-v16-poc` (Next 16 / React 19 `node_modules` intact, no reinstall needed).
>
> **State:** the app builds clean (`npm run build:web` GREEN) and runs clean in dev (login + masked inputs browser-verified). Both PoC blockers (Auth.js v5, react-input-mask) are resolved and included. The `useFetchData` render-loop fix is included (`7eca4504e`).
>
> **✅ EPIC DONE (2026-07-07): MLID-364 MERGED TO `develop`.** The epic → develop PR (doc: `docs/agomez/PR/MLID-364.md`) was opened by the user and **merged by PE Anatoliy**. Final epic HEAD before merge: `d6d9e3500` (14 commits ahead of develop, 0 behind). LISA (`apps/web`) now runs on Next.js 16.2.9 / React 19.2.7 on develop. **All R-steps complete or split out; no further work on this epic.** Active follow-up now = the three split-out tickets: **MLID-2740** (Turbopack), **MLID-2741** (pages/api test-route smell), **MLID-2742** (in-house `@repo/ui` masked-input to replace the `@mona-health` fork).
>
> **(historical) Next (2026-07-07): Sync 4 DONE. Epic = `d6d9e3500`, 0 behind develop / 14 ahead, pushed (0/0 vs origin).** Phase 1 (R1–R4) done + merged (fast-forward, 2026-07-03); R6 develop syncs ongoing on the validate-then-apply approach (temp branch off epic → verify → ff epic → push).
> - Local `develop` fast-forwarded in the **main** worktree (`40aee1465 → 0824796a4`). Merged local `develop` into the temporary branch from the **epic** worktree: **ZERO conflicts** (98 commits ahead, 60 added / 90 modified). Merge is **staged, not committed**.
> - Codemod re-application: **two** casualties. (1) net-new file `weekly-provider-reports/[id]/sends/[sendId]/resolve` `route.ts` + `route.test.ts` (async params + Auth.js v5 / R1 auth-mock) — new test **11/11**. (2) **subtle clean-merge break** in `app/api/orders/maintenance/route.test.ts`: epic had R1-renamed `mockGetServerSession`→`mockAuth`, develop added a new block using the old name; auto-merged with no conflict → `mockGetServerSession is not defined`. Re-applied the R1 rename → **63/63**.
> - **Sync gotcha:** `apps/web` `tsc` first showed ~39 `@repo/db` / `@repo/ui` "missing export" errors — stale gitignored `dist/`. Rebuilt both packages → **`tsc` clean**. Playbook: rebuild `@repo/db` + `@repo/ui` after every sync, before typecheck.
> - **Sync 1 verified GREEN:** merge 0 conflicts · `build:web` exit 0 · `tsc` clean · full suite 18,961 pass. Non-owned residue: `ordersTrackerCache.test.ts` (15) — **pre-existing on develop, confirmed via Next 14 baseline** (identical failure), merge-neutral, flag to team; known V8 `jobs/…export` crash; 3 flaky suites pass on rerun. **Sync 1 landed on epic (ff `d12040c9d → a85749584`).**
> - **Sync 2 (team pushed 5 more commits during sync 1):** `0824796a4 → 12b71100b`, 16 files (weekly-provider-reports / communications). Conflict-free, **zero codemod work** (no new dynamic routes, no auth casualties, no react-input-mask). `tsc` clean, affected tests green (10 suites / 208). **Landed on epic (ff `a85749584 → c87f4084d`).**
> - **Sync 3 (2026-07-07, +8h after sync 2):** `12b71100b → 506d4053e`, **1 file** (`ordersTrackerCache.test.ts`, 15/15 lines — develop's MLID-2411 fix). Conflict-free, **zero codemod work** (no migration-sensitive patterns). No package rebuild (delta didn't touch `packages/`), no `build:web` (test-only, no signature change). **This RESOLVES the flagged 15 `ordersTrackerCache` failures** — suite now **58/58 green in the epic's Next 16 / React 19 env**, `tsc` clean. **Landed on epic (ff `c87f4084d → f0202ab8f`), pushed.** `temp/…-sync-3` kept.
> - **Gitignore fix (2026-07-07, committed directly to epic per user OK):** `apps/storybook/storybook-static/` (R8 `build:storybook` output) was untracked and NOT covered by `.gitignore`, showing in every `git status` on the epic worktree. Deleted the stray folder + added `apps/storybook/storybook-static/` to the **root** `.gitignore` (matching the repo convention of centralizing per-app build outputs there — `apps/web/e2e/*`, `apps/kiosk/public/sw.js` — rather than a new `apps/storybook/.gitignore`). Verified the rule now ignores files inside. Commit `6a0522fc6` (`[MLID-364] - chore(gitignore): …`), pushed.
> - **Sync 4 (2026-07-07):** `506d4053e → 492bf2680`, **13 commits** (MLID-2734 orders/WeInfuse series linking, MLID-2635 document-fax-categories, MLID-2366 staging SMS, MLID-2411 + MLID-683 test fixes). **1 real conflict** — `documents/manual-upload/route.ts` imports: epic had migrated `authOptions`→`auth`, develop kept `authOptions` **and** added a new `isValidDocumentCategory` import. Resolved (shown to user + approved): kept `auth`, kept `isValidDocumentCategory`, dropped dead `authOptions` (body already `await auth()`). **3 deceptive-clean-merge casualties** (same as sync 1) — develop's new blocks in `orders/maintenance`, `orders/new`, `documents/manual-upload` route tests reintroduced `mockGetServerSession`/`typeof getServerSession`; re-applied the R1 rename → `mockAuth`/`typeof auth`. Rebuilt `@repo/db` + `@repo/ui` (packages/db had new migrations), `tsc` clean, **12 affected suites / 476 tests green**. **Landed on epic (ff `6a0522fc6 → d6d9e3500`), pushed.** Post-landing full gates on request — **all GREEN**: `build:web` exit 0 (4/4, 3m34s); full suite **19,046 pass** (only the known V8 `jobs/…export` crash fails-to-run); `apps/web` lint **0 errors / 250 warnings** (= R7 baseline). `temp/…-sync-4` kept.
> - **Epic now = `d6d9e3500`: 0 behind `develop`, 14 ahead. PUSHED to `origin/epic`; local epic 0/0 vs origin.** `temp/…-sync` … `-sync-4` kept.
> - **Open:** R10 runtime verification (user, browser — checklist in `docs/agomez/testing/`) → R12 team decisions (brief in `docs/agomez/deploy/`) → R13 epic→develop PR. Hardening done: **R7 ✅** (lint Option A, 0 in-scope errors), **R8 ✅** (Storybook builds on React 19).
> - **Follow-up tickets split out of this epic:** **MLID-2740** (Turbopack / R9), **MLID-2741** (pages/api test-route smell / R11), **MLID-2742** (in-house `@repo/ui` masked-input to replace the `@mona-health` fork — PE request; may gate release or fast-follow). All *relate to* MLID-364 with tracking comments.
> - Keep syncing develop frequently. **`ordersTrackerCache.test.ts` no longer needs flagging** — develop fixed it in MLID-2411 (sync 3) and it passes in the epic env.
>
> ---
> **(earlier)** R1 ✅ committed `6f101fb56`; R2 ✅ committed `d0d9200` on `feature/MLID-364-auth-v5-test-mocks`.
> - **node project: green** (12,601 pass; only pre-existing V8 crash + flaky parity remain).
> - **jsdom project: 9 failed suites / 15 failed tests** (from 47/33), 6,076 pass. Zero `use()`, zero next-auth ESM, zero `findDOMNode` project-wide.
> - **R3 (react-input-mask) looks already done** (0 `findDOMNode`) — needs only a verification pass.
> - **Remaining Phase-1 work = triage the 9 jsdom suites:** `app/layout.test.tsx` (R4 RootLayout), the `insurance-payors/[id]/guidelines` async-server-component test, and 7 assorted component suites (AuthReviewDetail, drug-detail-view, insurance-plan-detail-view, RevealKeyModal, ReviewDataStep, PriorAuthReview, MuiPhoneField). None are `use()`/ESM/findDOMNode.
> - First develop sync (R6) still deferred until Phase 1 finishes.
>
> _Update this line at the end of each working session so the next pickup is instant._

## Status legend

⬜ Not started &nbsp; 🟡 In progress &nbsp; ✅ Done &nbsp; ⛔ Blocked

## Status table

| Step | Status |
|---|:--:|
| **R0 — Graduate the branch (spike → epic)** | ✅ |
| **R1 — Auth.js v5 test mocks (~156 files)** | ✅ |
| **R2 — Async-params page tests (~130)** | ✅ |
| **R3 — react-input-mask test mocks (~54)** | ✅ |
| **R4 — RootLayout test (`app/layout.test.tsx`)** | ✅ (folded into the jsdom-fix batch `d12040c9d`) |
| **R5 — API-route param mock type accuracy (optional)** | ⬜ |
| **R6 — Daily develop sync (first run after Phase 1)** | 🟡 (merge clean, transforms + `tsc` done; build + full tests pending) |
| **R7 — Lint (migration-scoped, Option A)** | ✅ (2026-07-06: `apps/web` lint = 0 errors / 250 warnings; pre-existing backlog + warnings deferred to a separate ticket per PO) |
| **R8 — Storybook React 19** | ✅ (2026-07-06: builds clean on React 19; only optional cosmetic `^18→^19` manifest fix left) |
| **R9 — Turbopack migration** | ➡️ split into **MLID-2740** (own ticket, out of scope for this epic) |
| **R10 — Runtime verification sweep** | ✅ (user manual browser pass; satisfactory → PR opened + merged) |
| **R11 — pages/api test-route smell** | ➡️ split into **MLID-2741** (own ticket, pre-existing, out of scope for this epic) |
| **R12 — Team decisions (beta dep, cookie invalidation, landing strategy)** | ✅ (resolved by the PE merging the single epic PR) |
| **R13 — Epic → develop pull request** | ✅ MERGED by PE Anatoliy (2026-07-07) |

(Step definitions are in the plan file.)

---

## Per-step log

### R0 — Graduate the branch (spike → epic) ✅

**2026-07-03 — spike promoted to a real epic branch.** After browser-verifying the basic routes on Next 16, the decision was made to build the real implementation on the PoC rather than restart off develop. Steps taken:

- Confirmed the PoC worktree was clean and on `spike/MLID-364-next16-direct-poc` at `7eca4504e`.
- Created `epic/MLID-364-nextjs-v16-migration` **from the PoC HEAD** (no rebase — a plain branch off the current tip, so the entire PoC history plus the `useFetchData` fix carried over intact). The worktree now sits on the epic branch; its Next 16 / React 19 `node_modules` was untouched (no reinstall).
- **`spike/MLID-364-next16-direct-poc` is preserved** at `7eca4504e` (not deleted) so the original PoC record stays as evidence.
- Pushed the epic branch with upstream tracking: `origin/epic/MLID-364-nextjs-v16-migration`.

**Already included in the epic (carried from the PoC):** the full change-set — Auth.js v5 migration, `@mona-health/react-input-mask`, `react-select` 5.10.2, async params across 71 files, `middleware`→`proxy`, the 64 React 19 / Next 16 type fixes, `next.config.mjs` cleanups, ESLint 9 flat config (LISA scope), and the `useFetchData` render-loop fix.

### R1 — Auth.js v5 test mocks 🟡 (pattern proven on representatives; ~150 mechanization pending)

**Feature branch:** `feature/MLID-364-auth-v5-test-mocks` (off the epic).

**2026-07-03 — investigation: the mock landscape.** 156 test files reference `getServerSession`; **zero** import from `@/auth` yet (none migrated). The root mismatch: after S7a, all server source calls `auth()` from `@/auth` (App Router `auth()`, Pages Router `auth(req, res)`, `middleware/authCheck.ts` `auth()`), but every test still mocks the old `getServerSession`, so the source's `auth()` runs unmocked, hits real `@/auth`, returns null → 401. Buckets:

| Bucket | Count | Current mock | In R1? |
|---|---|---|---|
| A — Server route/handler tests (App + Pages Router) | ~153 | `jest.mock('next-auth', …)` + `getServerSession` | **Yes (core)** |
| B — `middleware/authCheck.test.ts` | 1 | `jest.mock('next-auth/next')` | **Removed** — orphaned (never collected by jest); deleted per user decision |
| C — `[...nextauth]` callback tests (`signIn`, `session`) | 2 | mock `next-auth` + providers; import callbacks from old route | **Yes — manual** (callbacks moved to `auth.ts`) |
| D — `lib/apiAuth.test.ts` | 1 | `jest.mock('next-auth/jwt')` (getToken) | No — already fixed in PoC (proxy) |
| E — Client page tests | ~37 | `jest.mock('next-auth/react')` (`useSession`) | **No — out of scope** |

**Bucket E is not auth-migration work.** Auth.js v5 keeps the client API in `next-auth/react`, and source still imports `useSession` from there (verified `admin/feature-flags/page.tsx`). Those 37 may still be red from react-input-mask/async-params (R2/R3), not auth.

**The uniform transform (buckets A + B):**
- `jest.mock('next-auth'…)` / `jest.mock('next-auth/next')` → `jest.mock('@/auth', () => ({ auth: jest.fn() }))`
- `import { getServerSession } from 'next-auth'` → `import { auth } from '@/auth'` (keep any co-imported `Session` type as `import type { Session } from 'next-auth'`)
- identifier `getServerSession` → `auth`; alias `mockGetServerSession` → `mockAuth`; casts standardize to `auth as jest.Mock`
- `mockResolvedValue` works for both `auth()` and `auth(req, res)`, so App Router and Pages Router share one rule.
- Mocking `@/auth` with a factory is cleaner than before — it stops the real NextAuth + MongoDB adapter from loading. The now-dead `jest.mock('@/app/api/auth/[...nextauth]/route', () => ({ authOptions: {} }))` was **left in place** (harmless) to minimize churn; optional cleanup for the codemod.

**Per-file variance the codemod must handle:** explicit-factory vs auto-mock (`jest.mock('next-auth')`); `import` vs `const { getServerSession } = require('next-auth')` (Pages Router); a `mockGetServerSession` alias variable; `Session` co-imported as a type.

**Representative proof (2026-07-03) — 3 runnable variants GREEN.** Applied the transform by hand to one file per variant and ran them:
- `app/api/feature-flags/route.test.ts` (explicit factory + `(getServerSession as jest.Mock)`) — PASS
- `app/api/clinical-reviews/__tests__/route.test.ts` (auto-mock + `mockGetServerSession` alias, 16 usages) — PASS
- `pages/api/patients/getByIds.test.ts` (Pages Router `require('next-auth')`, 7 usages) — PASS
- Combined: **3 suites, 26 tests passed.**

**Orphan finding + resolution — `middleware/authCheck.test.ts` REMOVED (2026-07-03).** The `node` project `testMatch` covers only `constants/`, `services/`, `utils/`, `types/`, `models/`, `pages/api/`, `app/api/`, `app/checkin/`, `lib/` — **`middleware/` is in no testMatch** (pre-existing gap, present on develop), so the file was never collected or run by jest (`jest middleware/authCheck` → "0 matches"). It was the only test file under `middleware/` (siblings `authCheck.ts` and `rateLimit.ts` are source). Per user decision, a never-run test is dead weight, so the file was **deleted** (`git rm -f`, staged on the feature branch) rather than wired into `testMatch`. **Note for a future develop sync:** if the team later adds `middleware/` to `testMatch`, any `middleware/*.test.ts` arriving through develop will need the same `getServerSession`→`auth` transform, and `middleware/authCheck.ts` (source) would then be uncovered — revisit test coverage there if that happens.

#### Sweep executed (2026-07-03) — node suite now GREEN (0 test failures)

Ran a dependency-free Node codemod (whole-`jest.mock` balanced-paren replacement + import/require/identifier rewrites) over all `*.test.ts` / `*.spec.ts` under `apps/web`, excluding the `[...nextauth]` callback tests.

**Codemod result:** 152 `.test.ts` changed on the first pass (1 straggler auto-flagged and hand-fixed: `orders/details/route.test.ts` had an inline `type Session` co-import the regex missed). Extended to `.spec.ts` and re-ran → 4 more (`admin/insurance-plans`, `insurance-plans/[id]`, `users-locations`, `pages/api/users/users-auth`). With the 3 representatives + the deleted orphan, **175 test files touched (174 modified + 1 deleted)**.

**Edge-variant fixes the full suite surfaced (each hand-fixed):**
1. **Factory-referenced-a-local-mock (3 files)** — `patient/orders/route.test.ts`, `patient/orders/[weInfuseOrderId]/link/route.test.ts`, `intakes/[id]/order-recommendation/route.test.ts` pre-declared `let mockGetServerSession = jest.fn(...)` and the factory referenced it. The whole-call replacement dropped that wiring → orphaned mock → 401. Fixed by wiring the factory: `jest.mock('@/auth', () => ({ auth: (...args) => mockAuth(...args) }))`.
2. **Local identifier collision (1 file)** — `admin/api-keys/[id]/route.test.ts` had a local helper `function auth()` that shadowed `import { auth } from '@/auth'`. Renamed the helper to `signInAsAdmin()`.
3. **Obsolete v4 assertion (1 file)** — `pages/api/users/getUsers.test.ts` had a test `should use authOptions from imported module` asserting `auth` was called with `authOptions` (v4). Rewrote it to assert the v5 Pages Router signature `auth(req, res)`.

**ESM-crash discovery (important, and broader than R1).** Many suites that never mocked `getServerSession` still failed to *load* with `SyntaxError: Cannot use import statement outside a module` from `next-auth`. Cause: they auto-mock `@/middleware/authCheck` (no factory) or otherwise transitively import the **real `auth.ts`**, which does `import NextAuth from 'next-auth'` — and next-auth v5 is **ESM-only**, which ts-jest (CJS) cannot load. Fix that works: add `jest.mock('@/auth', () => ({ auth: jest.fn() }))` so the real `auth.ts` never loads. Applied to **14** app/api suites (admin/configuration, auditor/feedback×3, content-library×8, insurance-payors×3) via a second small script (insert before the first `jest.mock(` to preserve any leading `@jest-environment` docblock).

**This ESM issue is SYSTEMIC and pre-existing (not caused by the R1 codemod).** It affects any test transitively importing `auth.ts` — confirmed beyond app/api in `pages/api/patients/[id]/financialCalculation.spec.ts` (no auth refs at all) and `services/mongodb/__tests__/holiday.parity.test.ts`. A **decision is needed**: a systemic fix (either add `next-auth`/`@auth` to jest `transformIgnorePatterns`, or a global `@/auth` mock via `moduleNameMapper`/setup) versus continuing per-file `jest.mock('@/auth')`. Per-file is proven and low-risk but is whack-a-mole across the whole suite; systemic is config-only but needs validation that next-auth's ESM dep chain transforms cleanly.

**Verified status — `npx jest --selectProjects node "app/api" "pages/api"`:**
- **Tests: 12,498 passed, 0 failed, 2 skipped.**
- **Test Suites: 867 passed, 5 failed** — all 5 are "failed to run", none are assertion failures:
  - `[...nextauth]/session.test.ts`, `[...nextauth]/signIn.test.ts` — bucket C, **manual R1 work still pending** (ESM-crash + they test callbacks that moved to `auth.ts`).
  - `jobs/analytics-process-intake-overview/export/route.test.ts` — pre-existing V8 worker crash, unrelated to auth.
  - `pages/api/patients/[id]/financialCalculation.spec.ts`, `services/mongodb/__tests__/holiday.parity.test.ts` — pre-existing ESM crashes (systemic issue above), not R1 codemod.

**Working-tree state (uncommitted, on `feature/MLID-364-auth-v5-test-mocks`): 174 `M` + 1 `D` test files.** Not committed — held for review.

#### Global `@/auth` test mock adopted (2026-07-03) — ESM crashes fully resolved (config-only)

User chose the **global mock** over per-file. Implemented:
- New `apps/web/jest/authMock.ts` — exports `auth`/`signIn`/`signOut`/`handlers` as `jest.fn()` (matches `auth.ts`'s exports).
- `jest.config.js` — added `'^@/auth$': '<rootDir>/jest/authMock.ts'` **before** the generic `'^@/(.*)$'` mapper (applies to all config blocks). No test ever loads the real `auth.ts`, so next-auth ESM never reaches ts-jest. Suites can still `jest.mock('@/auth', …)` to override per file (factory beats moduleNameMapper).
- **Reverted the 14 per-file `jest.mock('@/auth')` patches** — the global mapper makes them redundant. Those 14 files are back to untouched, keeping the R1 changeset focused on the actual `getServerSession`→`auth` migration.

**Final verified status — full `npx jest --selectProjects node` (whole node project):**
- **0 ESM crashes** (was 32).
- **Tests: 12,592 passed**, 2 skipped.
- **5 suites fail**, none from R1:
  - `[...nextauth]/session.test.ts`, `[...nextauth]/signIn.test.ts` — bucket C, **the only remaining R1 work** (rework to test the callbacks now in `auth.ts`).
  - `jobs/analytics-process-intake-overview/export/route.test.ts` — pre-existing V8 worker crash, unrelated.
  - `services/lidrugs/__tests__/liDrugs.parity.test.ts`, `liDrugsMapResolution.parity.test.ts` — **pre-existing flakiness**: all 24 parity suites (215 tests) pass in isolation; they fail intermittently under full-suite concurrency (the failing set changed run-to-run — `holiday.parity` one run, `liDrugs` the next). Not auth-related.

#### Callback tests reworked (2026-07-03) — R1 COMPLETE

The 2 `[...nextauth]` callback tests imported `authOptions.callbacks` from the old v4 route, which no longer exists. Reworked by **extracting the callbacks out of the `NextAuth({...})` config**:
- New `apps/web/app/api/auth/[...nextauth]/callbacks.ts` — exports `signInCallback`, `redirectCallback`, `jwtCallback`, `sessionCallback` (logic copied verbatim; only a **type-only** `import type { NextAuthConfig }` from next-auth, which is erased, so no ESM load and the global `@/auth` mock doesn't hide them).
- `auth.ts` now imports and references those callbacks; the two env-derived constants (`authorizedEmails`, `AUDITOR_EMAILS`) and the now-unused imports moved to `callbacks.ts`.
- `signIn.test.ts` / `session.test.ts` rewritten to import the callbacks directly and mock only `@/services/mongodb/users` + `@/utils/logger` (dropped all the next-auth/adapter/provider load-time mocking). **19 tests pass.**
- Corrected one stale assertion (never validated because the suite previously ESM-crashed): the no-session case returns the falsy session it was given (`null`) and skips the DB lookup — was asserting `toBeUndefined()`, now `toBeNull()` + "lookup skipped". Source behavior unchanged.
- Test-infra: `jest/authMock.ts` added to `tsconfig.json` `exclude` (it uses the ambient `jest` global; not app code).

**R1 FINAL — full `npx jest --selectProjects node`:** **12,601 tests pass, 0 failures**, 2 skipped, **0 ESM crashes**; `tsc --noEmit` exit 0. Only 2 suites fail, both pre-existing and non-R1:
- `jobs/analytics-process-intake-overview/export/route.test.ts` — V8 worker crash.
- one **flaky** parity suite (a different file each run — `holiday.parity` → `liDrugs.parity` → `argenx/inventoryReport.parity`); all 24 parity suites / 215 tests pass in isolation.

**R1 changeset (uncommitted, `feature/MLID-364-auth-v5-test-mocks`):** ~176 test files (migrated) + 1 deleted orphan; source `auth.ts` (callbacks extracted) + new `callbacks.ts`; test infra `jest/authMock.ts` + `jest.config.js` (moduleNameMapper) + `tsconfig.json` (exclude). Optional dead-mock cleanup (`[...nextauth]/route` + `authOptions`) left in place (harmless).

**Not R1 (surfaced, pre-existing — track separately):** the `jobs/…export` V8 worker crash; parity-test flakiness under full-suite concurrency.

### R2 — Async-params page tests 🟡 (in progress)

**2026-07-03 — R1 committed as `6f101fb56` (167 files); R2 started on the same feature branch** (`feature/MLID-364-auth-v5-test-mocks` — the auth work is done and committed; continuing here, will rename/rebranch only if the user wants R2 isolated).

Scope (from the PoC S6b finding): the client `[id]/page.test.tsx` suites render a page that now does `const params = use(props.params)` (React 19). Tests pass a plain object, so `use()` throws `An unsupported type was passed to use(): [object Object]`.

**Better fix than the PoC's suggestion (no Suspense / no async restructuring):** React's `use()` reads a thenable whose `status` is already `'fulfilled'` **synchronously** (returns `.value` without suspending). New helper `apps/web/tests/react/asyncParams.ts` → `fulfilledParams(value)` builds exactly that. Wrapping the test's params in `fulfilledParams({ id })` makes `use(props.params)` resolve on the first render, so **every existing (sync + async) assertion keeps working** with a one-line-per-params change. Confirmed on `insurance-plans/[id]` (19 tests) and `drug-admin-panel/drugs/[id]` (23 tests) → **42 tests green**.

**Pages that use `use(props.params)` (7):** intakes/[id], intakes/[id]/review, insurance-plans/[id], drug-admin-panel/drugs/[id], appointments/[id], admin/locations/[id], admin/ai-chat-settings/[id]. Only 4 have `page.test.tsx` (intakes/[id], intakes/[id]/review, appointments/[id] have none — nothing to fix there).

**R2 status:**
- ✅ `insurance-plans/[id]/page.test.tsx` — 19 pass.
- ✅ `drug-admin-panel/drugs/[id]/page.test.tsx` — 23 pass (also fixed a stale assertion that read `mockParams.id`, now a thenable → used the literal id).
- 🟡 `admin/locations/[id]/__tests__/page.test.tsx` and `admin/ai-chat-settings/[id]/page.test.tsx` — `fulfilledParams` applied, but **blocked by a separate ESM crash** (below).
- `insurance-payors/[id]/guidelines` — NOT an R2 case: the page is an **async server component** (`await props.params`), so its test fails for a different reason (rendering an async component in RTL). Handle separately.

**Global `next-auth/react` mock adopted (user decision) — big win.** `next-auth/react` is ESM-only in v5, so any client test transitively importing it crashed at load. Added `apps/web/jest/nextAuthReactMock.ts` (`useSession` default-unauthenticated, `signIn`/`signOut` spies, `SessionProvider` pass-through) and mapped `^next-auth/react$` to it via jest `moduleNameMapper` (all config blocks), same playbook as `@/auth`. Suites that need specific behaviour still `jest.mock('next-auth/react', …)` to override.

**R2 DONE + jsdom project transformed.** All 4 `use()`-page suites green (insurance-plans 19, drug-admin 23, locations + ai-chat-settings 23). Full jsdom run after the mock: **9 failed suites / 15 failed tests, 6,076 passed** (was 47 failed / 33 failed). Signature counts across the whole jsdom project: **`use()` unsupported = 0, ESM import = 0, `findDOMNode` = 0.**

**R3 (react-input-mask mocks) appears ALREADY RESOLVED** — 0 `findDOMNode` errors project-wide. The S7b source swap to `@mona-health/react-input-mask` removed `findDOMNode`, so the ~54 masked-input test failures the PoC attributed to R3 are gone. R3 likely needs only a verification pass, not new work (confirm when we get there).

**Remaining 9 jsdom failed suites (15 tests) — beyond R2, assorted (candidates for R4 + cleanup):**
- `app/layout.test.tsx` — RootLayout (R4).
- `app/insurance-payors/[id]/guidelines/…/page.test.tsx` — async **server** component rendered in RTL (separate pattern).
- `app/orders-tracker/…/auth-review/_components/AuthReviewDetail.test.tsx`
- `components/Modules/admin/drug-clinical-review/drug-detail-view/…`
- `components/Modules/admin/insurance-plan-detail-view/…`
- `components/Modules/api-keys/RevealKeyModal.test.tsx`
- `components/Modules/document-intake/steps/ReviewDataStep.test.tsx`
- `components/Modules/patient/…/PriorAuthReview.test.tsx`
- `components/UI/MUI/MuiPhoneField.test.tsx`
None are `use()`/ESM/findDOMNode — they're assorted React 19 / component issues to triage next.

**R2 changeset (uncommitted, on `feature/MLID-364-auth-v5-test-mocks`):** `tests/react/asyncParams.ts` (helper); 4 page-test edits; `jest/nextAuthReactMock.ts` + `jest.config.js` (next-auth/react mapping).

### R3 — react-input-mask test mocks ✅ (done 2026-07-03)

Mostly resolved already by the S7b source swap to `@mona-health/react-input-mask` (removed `findDOMNode`) — the full jsdom run showed **0 `findDOMNode` errors**, so the ~54 render-crash failures the PoC attributed to R3 were gone. One leftover: `components/UI/MUI/MuiPhoneField.test.tsx` still did `jest.mock('react-input-mask', …)`, and that package is now uninstalled → `Cannot find module 'react-input-mask'`. Fixed by mocking `@mona-health/react-input-mask` instead (a `forwardRef` input, matching the fork's API — ref on the component, not `inputRef`). 9 tests green. Confirmed no other test file references the old module.

---

## Phase 1 jsdom triage — the 9 CI-scoped failing suites (2026-07-03)

**Why this matters (CI scope investigation).** The PR CI (`.github/workflows/unit-tests.yml`) runs `npx jest --ci --changedSince=origin/develop --coverage --shard=N/4 --passWithNoTests` on `apps/web` (sharded ×4), plus a separate whole-app `npm run types:check`. `--changedSince` runs only tests changed or depending on changed source. On a normal small PR that's a few tests (so a pre-existing broken test never in the changed graph stays green via `--passWithNoTests` — this is how latent-broken tests hide on develop). But the epic PR changes so much that the selection is **352 of 1,264 test files** — and **all 9 remaining jsdom failures are in that 352**, so they WILL block the epic→develop PR. (The `auth.ts` hub effect is neutralized because jest maps `@/auth` to the mock, which is why it's 352 and not ~all.) `types:check` is already green.

**Verdict: all 9 are migration-caused** (React 19 render-helper change, React 19 component-call signature, MUI 7 stepper rendering, removed `react-input-mask`), not unrelated pre-existing breakage — so we own fixing them. Working through them grouped by root cause; each logged below.

- ✅ **`components/UI/MUI/MuiPhoneField.test.tsx`** — see R3 above (old `react-input-mask` mock → `@mona-health` fork). 9 tests green.

**Fixes applied (2026-07-03), each verified green:**
- ✅ **`components/Modules/document-intake/steps/ReviewDataStep.test.tsx`** (56 tests) — `toHaveBeenCalledWith(objectContaining(...), expect.anything())` failed: React 18 called function components with a `{}` legacy-context 2nd arg (matched by `anything()`); **React 19 removed legacy context and passes `undefined`**, which `anything()` rejects. Changed the 2nd arg matcher to `undefined`.
- ✅ **`components/Modules/api-keys/RevealKeyModal.test.tsx`** (4 tests) — "Close not enabled after Copy": the async `clipboard.writeText` + `setState` no longer flushes by the time `findByRole` returns under React 19. Switched to `waitFor(() => expect(...).toBeEnabled())`.
- ✅ **`app/layout.test.tsx`** (22 tests) — `RangeError: Maximum call stack`. RootLayout returns `<html>/<body>`; React 19 emits a DOM-nesting error with a **changed message**, so the suite's `validateDOMNesting` filter missed it and its forward-to-original console.error spy recursed against the global jest.setup console spy. Replaced the fragile bind-and-forward spy with a plain `console.error` suppression (the suite only asserts providers render).

**All 9 now fixed:**
- ✅ `AuthReviewDetail` + `PriorAuthReview` — **not** an MUI/stepper rendering change: `usePriorAuthReviewWizard` starts `letterType=''` (→ `isMultiStep=true`, all steps visible), then a `useEffect` reads the fetched `documentType` and collapses to single-step. React 19 flushes that effect after the first paint, so the tests' immediate "step 2/3 absent" assertions (run right after `findByText('verify letter type')`, which is present in both states) raced the transient multi-step render. Fixed by moving the absence assertions into `waitFor` so they wait for the effect to settle. Also fixed a sibling test ("opens on step 3 for reviewed multi-step") the same way (step-3 content is set by an effect). AuthReviewDetail + PriorAuthReview green.
- ✅ `insurance-payors/[id]/guidelines` — async **server** component; RTL can't render an async element. Awaited the component to its JSX first (`const ui = await Page({ params: Promise.resolve({ id }) }); render(ui)`), then asserted. Green.
- ✅ `insurance-plan-detail-view` + `drug-detail-view` — **RESOLVED via the `react-simple-toasts` 6.0.0 → 6.1.0 bump** (user-approved; see finding below). 70 tests green.

**PHASE 1 TEST SUITE GREEN.** Full run after all fixes: **jsdom 392 suites / 6,100 tests pass, 0 failures; node 12,601 tests pass.** The only remaining failure is the pre-existing V8 worker crash in `jobs/analytics-process-intake-overview/export/route.test.ts` (a known jest-on-V8 issue, not migration-caused, and its assertions are not the blocker — it fails to *run*; tracked separately). `tsc --noEmit` is green.

### Feature → epic merge (2026-07-03)

Per the epic workflow (feature branches merge into the epic, no per-feature PR), merged `feature/MLID-364-auth-v5-test-mocks` into `epic/MLID-364-nextjs-v16-migration`. It was a **fast-forward** (the feature was a clean linear continuation of the epic), so the epic now points at `d12040c9d` and carries R1 + R2 + R3/R4 + the toast bump. `feature/…` branch kept (not deleted). **Not pushed** — the local epic is 3 commits ahead of `origin/epic`; push when resuming. Now sitting on the epic branch. (Why a feature branch existed at all: convention is never to commit directly to the epic — branch off it, then merge in.)

**⚠️ NEW RUNTIME FINDING (not test-only) — `react-simple-toasts@6.0.0` breaks under React 19.** The `reactRender is not a function` failures trace into `react-simple-toasts`: its `legacyRender` calls `ReactDOM.render`, removed in React 19. Stack: component `handleSave` → `showToast` (`components/UI/Toast/Toast.tsx`) → `toast()` → `react-simple-toasts`. **This means toasts crash in the real app under React 19, app-wide — not just in tests** (the PoC's S2 audit mislabeled `react-simple-toasts` as compatible because its peer range is permissive `>=16.8.0`, the same trap as react-input-mask). Verified fix: **`react-simple-toasts@6.1.0`** adds `createRoot` detection (`mainVersion >= 18 && createRoot` → modern render), so it works on React 19. **RESOLVED (2026-07-03, user-approved):** ran `npm install react-simple-toasts@6.1.0 --workspace=apps/web` (now `^6.1.0` in `apps/web/package.json` + lockfile updated). The two toast-triggered suites (insurance-plan-detail-view, drug-detail-view) went green. **This is a required production fix**, not just a test fix — it repairs toasts app-wide under React 19. Add to the real-migration change-set / announce to the team.

### R4 — RootLayout test ⬜
—

### R5 — API-route param mock type accuracy (optional) ⬜
—

### R6 — First develop sync 🟡 (in progress 2026-07-06)

**Approach (agreed):** merge on a **temporary branch off the epic** first, validate, then apply the reconciled result to the real epic. Temporary branch: `temp/MLID-364-develop-sync`, cut off epic HEAD `d12040c9d`.

**Freshness (Option A):** advanced local `develop` in the **main** worktree with `git pull --ff-only` (`40aee1465 → 0824796a4`, clean fast-forward, no working-file changes). Then merged **local** `develop` into the temporary branch from inside the **epic** worktree (`git merge develop --no-commit --no-ff`). The two-worktree split matters: `develop` stays checked out in the main worktree, the merge/build/test all run in the epic worktree against the Next 16 / React 19 `node_modules`.

**Merge result: ZERO conflicts.** `develop` was 98 commits ahead of the merge base (`ad58f5d6f`); 60 files added + 90 modified under `apps/web`. Everything auto-merged, including files both sides touched (`jest.config.js`, `app/api/chat/*`, `app/api/orders/maintenance/*`). Merge is **staged, not committed**.

**Codemod re-application — the real sync work.** A clean merge is deceptive: brand-new files from `develop` arrive in Next 14 / React 18 form and dodge the transforms (files both sides touched surface as conflicts and are forced into view; only net-new files merge silently). Scanned `merge-base..origin/develop` and classified every added/modified file. Result: only **one new feature** needed transforming — the `weekly-provider-reports/[id]/sends/[sendId]/resolve` endpoint:
- `route.ts` — async params (`params: Promise<{…}>` + `await params`) **and** Auth.js v5 (`getServerSession(authOptions)` → `auth()` from `@/auth`).
- `route.test.ts` — R1 auth-mock transform (`jest.mock('@/auth', () => ({ auth: jest.fn() }))`, `getServerSession` → `auth`) + async params (`params: Promise.resolve({…})`). Kept the now-dead `jest.mock('@/app/api/auth/[...nextauth]/route')` to match how the R1 codemod left the other ~174 files.
- Zero new dynamic-route page tests, zero new `react-input-mask` usages.
- New `route.test.ts`: **11/11 pass**.

**Note on the lost R1 codemod:** the custom auth-mock Node script from R1 was a one-shot and is not saved anywhere tracked (only its target helper `apps/web/jest/authMock.ts` survives). For 2 files, hand-applied rather than rewriting the script.

**⚠️ SYNC GOTCHA — rebuild workspace packages after a sync.** First `tsc --noEmit` in `apps/web` surfaced ~39 errors, **none migration-related** — all "missing export" against `@repo/db` and `@repo/ui`. Root cause: those packages resolve to their **built `dist/`** (`@repo/db` → `./dist/index.d.ts`; `@repo/ui` via tsup), and `dist/` is **gitignored**, so the merge updated each package's `src/` (develop's `spruceSends`, `Message`→`ChatMessage` rename, `*ForRead` repository methods, financial-calc repo; `@repo/ui`'s new `TablePagination`) but not its declarations. `apps/web` was typechecking against 2026-07-03 `dist`. Fix: rebuilt `@repo/db` then `@repo/ui` (`npm run build` in each — non-destructive, regenerates `dist` only). **Playbook: after any develop sync, rebuild `@repo/db` and `@repo/ui` before typechecking.** After rebuild, `apps/web` `tsc --noEmit` is **clean (exit 0)**.

**A second, subtler merge casualty — `app/api/orders/maintenance/route.test.ts`.** This one is NOT a new file and did NOT conflict, yet the migration broke it. The epic's R1 codemod had migrated this file (renamed the mock `getServerSession`/`mockGetServerSession` → `auth`/`mockAuth`, used by 20+ existing tests). Develop independently **added** a new `describe('… document notification backfill')` block (+182 lines, MLID-2447) still written in pre-migration form (`mockGetServerSession.mockResolvedValue(...)`). Git auto-merged both with **no conflict** — the added lines don't overlap the renamed setup — producing a file that references an undefined variable → `ReferenceError: mockGetServerSession is not defined` in 6 tests. **This is the deceptive-clean-merge in its purest form: both sides changed the file, the text merged cleanly, but the result is semantically broken on the migration's own surface.** Attribution confirmed via `git diff base..epic` (epic touched it: −64/+60 = the R1 rename) and `git diff base..origin/develop` (develop touched it: +182 = the new block). **Fix (ours):** re-applied the R1 rename to develop's new block (`mockGetServerSession` → `mockAuth`, 6 occurrences). Suite now **63/63 green**.

**Not ours — `ordersTrackerCache.test.ts` (15 failing).** Pure develop code: `git diff base..epic` for `ordersTrackerCache.ts` + `.test.ts` is **empty** (the epic never touched either), so the merge brought them in unchanged from develop. They fail with `TypeError: Cannot read properties of undefined (reading 'collection')` — develop's new `fullRebuild(db)` / `loadBaseOrderDocsById(db, …)` path receives an undefined `db` in the test's job-start harness. **Ran the develop baseline (Next 14, main worktree): fails IDENTICALLY — same 15 tests, same error.** So it is pre-existing on develop, independent of the migration environment; merge-neutral, develop-owned. Not fixed here (per the sync convention: don't fix develop-inherited errors, only confirm the merge didn't cause them). Flag to the team.

**Verification results (all migration/merge-owned surfaces GREEN):**
- Merge: **0 conflicts** (98 commits, 60 added / 90 modified under `apps/web`).
- Transforms: resolve `route.ts` + `route.test.ts` (async-params + Auth.js v5); maintenance `route.test.ts` (R1 rename). New resolve test **11/11**, maintenance **63/63**.
- `npm run build:web`: **GREEN** (Next 16 production build, 8m22s, 4/4 tasks, exit 0).
- `tsc --noEmit`: **GREEN** (after rebuilding `@repo/db` + `@repo/ui`).
- Full suite: **18,961 pass**. Non-owned residue: `ordersTrackerCache` (15, pre-existing develop, baseline-confirmed), the known V8 worker crash in `jobs/…export` (fails-to-run), and 3 flaky suites (`AddProviderOfficeDrawer`, `generateKey`, `pgVsCacheBenchmark`) that pass on rerun.

**Sync 1 landed on the epic.** Committed the rehearsal merge on `temp/MLID-364-develop-sync` (merge commit `a85749584`, 2 parents `d12040c9d` + `0824796a4`, includes the 3 reconciliation edits). Because that merge commit has the epic as its first parent, landing it was a plain **fast-forward** `d12040c9d → a85749584` — no re-doing the merge, no re-applying edits. `temp/MLID-364-develop-sync` kept.

### R6 — Second (incremental) develop sync 🟡 (2026-07-06, same session)

The team pushed **5 more commits** to develop while sync 1 was being verified (`0824796a4 → 12b71100b`), so we re-synced immediately — the argument for syncing often: small deltas, cheap reconciliation.

**Process (identical to sync 1):** ff-pulled local `develop` in the main worktree; created `temp/MLID-364-develop-sync-2` off the freshly-landed epic (`a85749584`); merged local `develop` from the epic worktree. Because the epic already contained `0824796a4`, only the 5 new commits merged.

**Result — conflict-free, and ZERO codemod work.** 16 files under `apps/web`, all in the weekly-provider-reports / communications feature area. No net-new dynamic routes (no async-params transform), no `react-input-mask`. The two migrated dynamic-route tests both sides changed (`weekly-provider-reports/[id]/entries/[entryId]` and `/refresh`) auto-merged **cleanly, keeping the epic's `auth()` form** — no `getServerSession` or synchronous `params` reintroduced. The maintenance-route casualty did NOT recur because develop's changes this round were in business logic, not the auth-mock setup. (Still verified the merged working-tree versions of all 4 files by grep before trusting it.)

**Verification:** `tsc --noEmit` clean (no package rebuild needed — delta didn't touch `packages/`); affected tests green (**10 suites / 208 tests**: both dynamic routes, the communications `ReviewResolve`/`EditNote`/`ReviewResolveTable` components, provider-message + entries services). Did not re-run full `build:web` — tiny delta, no signature changes, `tsc` + targeted tests green.

**Sync 2 landed on the epic.** Committed on `temp/MLID-364-develop-sync-2` (merge commit `c87f4084d`), fast-forwarded the epic `a85749584 → c87f4084d`. **Epic is now 0 behind `develop` / 11 ahead** (its own migration commits). `temp/…-sync-2` kept.

**Pushed to `origin/epic`.** `git push origin epic/MLID-364-nextjs-v16-migration` — fast-forward `d12040c9d..c87f4084d`, exit 0 (pre-push ENV-suffix hook passed). `origin/epic/MLID-364-nextjs-v16-migration` now at `c87f4084d`; local epic **0/0** vs origin.

**Still open:** R7–R10 hardening. Keep syncing develop frequently (both syncs this session prove the cadence pays off: sync 1 = 98 commits / 2 casualties, sync 2 hours later = 5 commits / 0).

### R6 — Third (incremental) develop sync ✅ (2026-07-07)

Roughly 8 hours after sync 2, develop advanced by **2 commits** (`12b71100b → 506d4053e`, PR #1841 / **MLID-2411**). Re-synced with the identical validate-then-apply process.

**Process:** ff-pulled local `develop` in the main worktree (`12b71100b → 506d4053e`, clean); created `temp/MLID-364-develop-sync-3` off the epic (`c87f4084d`); merged local `develop` from the epic worktree with `--no-commit --no-ff`.

**Result — conflict-free, ZERO codemod work.** The merge staged exactly **one** file: `apps/web/services/jobs/definitions/orders-tracker-cache/ordersTrackerCache.test.ts` (15 insertions / 15 deletions). It is a service-job test — no new dynamic route, no `getServerSession`/`authOptions`, no `react-input-mask`, no synchronous `params` (grep scan clean). The epic had never touched this file, so develop's version merged straight through.

**This closes the long-standing flagged failure.** MLID-2411 is exactly the develop-side fix for the 15 `ordersTrackerCache` failures we had confirmed as pre-existing on develop (baseline-identical) and flagged to the team across syncs 1–2 ("pass db to `startJobAndWaitForInitialMaterialization` callsites"). Ran the suite in the epic's Next 16 / React 19 environment: **58/58 pass** (previously 15 failing). So the fix holds in the migration environment too — no epic-side action needed, and this item drops off the "flag to team" list.

**Verification:** merge 0 conflicts · 1 file staged · codemod 0 · affected suite **58/58** · `tsc --noEmit` clean (exit 0). No package rebuild (delta didn't touch `packages/`); no `build:web` re-run (test-only, no signature changes — same rationale as sync 2).

**Full suite run post-sync-3 (2026-07-07, on request):** `npx jest --ci` (both projects) → **19,008 pass**, 2 skipped; **2 suites fail, both known non-migration issues** — the pre-existing V8 worker crash in `jobs/…/export/route.test.ts` (fails to run) and the flaky `argenx/inventoryReport.parity.test.ts` (**re-run in isolation → 2/2 pass**, concurrency flake). Zero migration-owned failures; the MLID-2411-fixed `ordersTrackerCache` suite is now among the passing.

**Sync 3 landed on the epic.** Committed the merge on `temp/MLID-364-develop-sync-3` (merge commit `f0202ab8f`, parents `c87f4084d` + `506d4053e`), fast-forwarded the epic `c87f4084d → f0202ab8f`, and pushed (`c87f4084d..f0202ab8f`, ENV-suffix hook passed). **Epic now 0 behind `develop` / 12 ahead, 0/0 vs origin.** `temp/…-sync-3` kept.

### Storybook-static gitignore fix ✅ (2026-07-07)

Unrelated to the syncs but recorded here for the branch timeline. `apps/storybook/storybook-static/` (output of the R8 `npm run build:storybook` verification) was untracked and **not** covered by `.gitignore`, so it surfaced in every `git status` on the epic worktree. Investigated the confusing signals (`git check-ignore` on the directory-with-trailing-slash spuriously "matched" a blank line 99, but a file *inside* was not ignored and porcelain status listed it `??` → genuinely **not** ignored). Deleted the stray folder and added `apps/storybook/storybook-static/` to the **root** `.gitignore` — matching the repo convention of centralizing per-app build outputs there (`apps/web/e2e/*`, `apps/kiosk/public/sw.js`) rather than introducing a per-app `apps/storybook/.gitignore`. Verified the rule now ignores files inside. Committed **directly to the epic** (user OK'd, no feature branch): `6a0522fc6` `[MLID-364] - chore(gitignore): ignore storybook-static build output`, pushed (`f0202ab8f..6a0522fc6`).

### R6 — Fourth (incremental) develop sync ✅ (2026-07-07)

develop advanced **13 commits** (`506d4053e → 492bf2680`; note develop moved during the `pull` itself — the fetch showed `2df249f5d` but the ff landed on `492bf2680`, so the delta was re-baselined before merging). Commits: **MLID-2734** (orders — SQ protocol partner links its own WeInfuse series in the dual-order/Looker flow), **MLID-2635** (document fax-category → "Other" migration + `documentCategories` validation), **MLID-2366** (allow a staging SMS number), **MLID-2411** + **MLID-683** (test fixes: db arg to materialization helper; scope `@repo/db` jest `testMatch` to `*.test.ts`).

**Merge — 1 real conflict.** `apps/web/app/api/documents/manual-upload/route.ts`, import block only: the epic's auth codemod had replaced `import { authOptions } from '@/app/api/auth/[...nextauth]/route'` with `import { auth } from '@/auth'`; develop kept the old `authOptions` import **and** added a new feature import `isValidDocumentCategory` from `@/constants/documentCategories` on the adjacent line, so the two edits overlapped. **Shown to the user and approved before resolving.** Resolution = the migration codemod: keep `auth` (the body already reads `const session = await auth()`, auto-merged), keep develop's new `isValidDocumentCategory` (used at line 84), drop the now-dead `authOptions`.

**3 deceptive-clean-merge casualties (the sync-1 pattern, recurred).** develop added new `it(...)` blocks to three route tests still written in pre-migration form; they auto-merged with no conflict but referenced the R1-renamed identifiers:
- `orders/maintenance/route.test.ts` (MLID-2734 block) — 1× `mockGetServerSession` → `mockAuth`.
- `orders/new/route.test.ts` (MLID-2734 block) — 1× `mockGetServerSession` → `mockAuth`.
- `documents/manual-upload/route.test.ts` (MLID-2635 block) — 1× `mockGetServerSession` → `mockAuth` and `typeof getServerSession` → `typeof auth` (matched the file's existing migrated cast at line 107).
Each file already had the epic's `jest.mock('@/auth', …)` + `mockAuth` (64/64/17 uses); develop only reintroduced one each. A full sweep of every merge-touched file (`506d4053e..develop`) confirmed **no other** reintroduced `getServerSession`/`next-auth` mock patterns, and no new dynamic routes (no async-params work).

**Verification:** rebuilt `@repo/db` + `@repo/ui` (packages/db shipped new migrations `0070_*` + `_journal`/snapshot and a `jest.config.ts` change) → `tsc --noEmit` clean (exit 0). Ran all affected suites: **12 suites / 476 tests green** (the 3 casualty-fixed route tests, `_linkCreatedOrdersToWiSeries`, `ordersTracker`, `documentCategories`, `documentNotification`, `ocrDataExtractor`, `manualDocumentUploadService`, `genericDocumentSchemaValidator`, the `041` migration test, and the `[category]/[orderId]/documents` page test).

**Full `build:web` run post-landing (2026-07-07, on request) — GREEN.** `npm run build:web` on epic `d6d9e3500`: **exit 0, 4/4 tasks successful, 3m34s**, no error lines. (The `*.spec`/`*.test` routes still appear in the route listing — the known pre-existing R11 / MLID-2741 pages/api test-route smell, not a regression.) Confirms the sync-4 change-set builds clean, not just typechecks.

**Full test suite + lint run post-landing (2026-07-07, on request) — GREEN.** To close the gap left by running only the *affected* suites during the sync itself, ran the whole suite and lint on epic `d6d9e3500`:
- **Full suite (`npx jest --ci`, both projects): 19,046 pass, 2 skipped; 1 failed suite = the pre-existing V8 worker crash** in `jobs/analytics-process-intake-overview/export/route.test.ts` (fails to *run*, not migration-caused). No flaky parity suite this run. Zero migration-owned failures — develop's 13 new commits introduce no cross-suite breakage on the epic.
- **`apps/web` lint (`npm run lint`): 250 problems, 0 errors / 250 warnings, exit 0** — identical to the R7 Option A baseline (`c87f4084d`). The merged develop code + the sync-4 reconciliation edits added **zero** new lint errors; R7 still holds.

**Sync-4 automated green bar fully re-established on `d6d9e3500`:** merge clean · `tsc` clean · full suite green (only the known V8 crash) · `build:web` GREEN · lint 0 errors. Only R10 (user browser/runtime pass) remains outstanding for the epic.

**Sync 4 landed on the epic.** Merge commit `d6d9e3500` on `temp/MLID-364-develop-sync-4` (parents `6a0522fc6` + `492bf2680`), fast-forwarded the epic `6a0522fc6 → d6d9e3500`, pushed (`6a0522fc6..d6d9e3500`, ENV hook passed). **Epic now 0 behind `develop` / 14 ahead, 0/0 vs origin.** `temp/…-sync-4` kept.

### R7 — Lint backlog ⬜
—

### R8 — Storybook React 19 ✅ (verified 2026-07-06)

Questioned whether Storybook needs migrating at all, given it is a separate, non-shipping app. Answer: it is separate as an *app* but not in its *dependencies* — it renders `@repo/ui`, and npm-workspace hoisting already resolves **React `19.2.7`** for `apps/storybook` (verified via `npm ls react --workspace=apps/storybook`) even though its `package.json` still declares `react: ^18`. So it is already running on React 19 whether or not we act.

Tested it directly: `npm run build:storybook` → **completed successfully** (Storybook 10.2.8 + Vite, built in ~39s, exit 0). Only messages were a Vite >500 kB chunk-size hint and a Turbo `no output files found … outputs key` note — both cosmetic, neither React-19-related. Low runtime risk because the React-19-breaking libraries (`react-input-mask`, `react-simple-toasts`) live in `apps/web`, not in the MUI 7 + Emotion design system that Storybook renders.

**Verdict: R8 needs no migration work.** Two optional, low-value follow-ups (not done): (1) bump `apps/storybook` `react`/`react-dom` `^18 → ^19` so the manifest matches the resolved runtime; (2) a browser pass of a story or two to close the build-cannot-render-in-browser caveat.

### R9 — Turbopack migration ➡️ split into MLID-2740 (2026-07-06)

Next.js 16 defaults to Turbopack; the PoC kept webpack via `--webpack` (dev + build) to stay scoped to the framework / React 19 changes. The bundler switch is real work — `apps/web/next.config.mjs` has ~80 lines of custom `webpack()` config (IgnorePlugin for `sshcrypto.node` + `li-public-api`, IgnorePlugin/externals for Application Insights, NormalModuleReplacementPlugin for `node:`, client/server externals + `resolve.alias`/`resolve.fallback` for `ssh2`/`ssh2-sftp-client`/Node built-ins) and Turbopack does not run `webpack()` or support plugins, so all of it must be re-expressed in Turbopack's model. Not a blocker (webpack build is green), medium risk (an incomplete translation can leak server-only modules into the client bundle).

**Filed as its own ticket: MLID-2740** (Task, Sprint 31, 5 pts) — https://localinfusion.atlassian.net/browse/MLID-2740 — linked *relates to* MLID-364, and a tracking comment added on MLID-364. **Out of scope for this epic.**

### R10 — Runtime verification sweep ⬜
—

### R11 — pages/api test-route smell ➡️ split into MLID-2741 (2026-07-06)

`apps/web/next.config.mjs` sets `pageExtensions: ['ts', 'tsx']`, so Next treats every `.ts` under `pages/` as a route — including the **29** `.test.ts` / `.spec.ts` files under `pages/api/`, which get compiled and built as if they were live endpoints (e.g. `pages/api/patients/createPatient.test.ts` → `/api/patients/createPatient.test`). **Pre-existing** — predates the migration; the stricter Next 16 / Auth.js v5 ESM build only surfaced it (this is the root cause behind the `pages/api/.../financialCalculation.spec.ts` "fails to run" case). Fix = move the test files out of `pages/` (Jest finds them via its own config) or narrow `pageExtensions` so `.test.`/`.spec.` no longer match.

**Filed as its own ticket: MLID-2741** (Task, Sprint 31, 3 pts) — https://localinfusion.atlassian.net/browse/MLID-2741 — linked *relates to* MLID-364, tracking comment added on MLID-364. **Out of scope for this epic** (build is green; not a blocker).

### R12 — Team decisions ✅ (2026-07-07)

Resolved implicitly by the team accepting the single large epic PR: PE Anatoliy merged it, which settles the landing-strategy question (one epic PR, not staged) and constitutes acceptance of the beta `next-auth@5.0.0-beta.31` pin. The one-time forced re-login (session cookie `next-auth.*` → `authjs.*`) was documented in the PR body as a deploy heads-up.

### R13 — Epic → develop pull request ✅ MERGED (2026-07-07)

PR document written to `docs/agomez/PR/MLID-364.md` and pushed (`origin/epic/MLID-364-nextjs-v16-migration` @ `d6d9e3500`). The user opened the PR (base `develop` ← compare the epic branch); **PE Anatoliy merged it to `develop`.** MLID-364 is complete. Follow-up work continues under MLID-2740 / MLID-2741 / MLID-2742.

---

## Known carry-forward bugs (found during PoC / verification)

- **`useFetchData` render loop — FIXED (`7eca4504e`, in the epic's starting commit).** Consumers passing an inline `callback` got an unstable `customFetch`/`refetch`, which — listed in an effect dependency array in `PatientProvider` — re-fired the effect every render and looped ("Maximum update depth exceeded", first seen on `/patient/[id]/details`). Fixed by holding the callback in a ref so the memoized functions keep stable identities while still calling the latest callback. Latent bug shared with develop; resolved here. A dedicated test (`useFetchData.test.tsx`) covers the stable-identity guarantee.
- **Feature-flags 401 in the PHI-masking path — pre-existing, deferred.** On `/patient/[id]/document-records` (and any route running server-side PHI masking), `getAllFeatureFlags` makes a cookie-less internal `fetch` to the session-gated `/api/feature-flags`, which returns 401. The error is caught and masking falls back to code-defined defaults; the request still returns 200 (non-fatal, noisy). Confirmed **not** caused by the upgrade: `featureFlags.ts` and `phiMaskingServer.ts` are byte-identical to develop, and the only route changes are the mechanical `getServerSession`→`auth()` codemod with identical 401 semantics. Fix direction: call `FeatureFlagService` directly server-side instead of the HTTP round-trip. Candidate **separate bug ticket**, not migration work.
