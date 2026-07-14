# MLID-364 — Next.js v16 PoC: Implementation History

> The **running log** for the MLID-364 Next.js 16 / React 19 proof of concept: live status, every step taken, errors found and how they were solved, and the final results/verdict. This is the **single source of truth for "where we are."**
>
> **Companion document:** [`MLID-364-nextjs-poc-plan.md`](./MLID-364-nextjs-poc-plan.md) — the stable PoC plan (goal, rationale, isolation strategy, sequence, decision, stage definitions). Read that for *what each stage is*; read this file for *what actually happened*.
>
> This file lives only in the main folder (`docs\agomez\preplans\`) and is never committed onto the spike branch. All status/history updates happen here.

---

## Resume pointer

> **✅ PoC COMPLETE — Next 16 + React 19 migration proven, both blockers resolved. Stopped here (2026-07-02) at user request; branch PUSHED (not torn down).** &nbsp;|&nbsp; **Spike branch HEAD: `d2cff3bf4` — pushed to `origin/spike/MLID-364-next16-direct-poc`.**
>
> **Commit chain (all pushed):** `fc256252b` (S1–S3 core upgrade) → `5a3b424e4` (S4+S5+S6b, build GREEN) → `cd5700508` (S6c ESLint 9 flat config, LISA scope) → `41e54b08b` (S7a Auth.js v5) → `d2cff3bf4` (S7b react-input-mask replacement).
>
> **Verdict: the migration is real and viable.** `apps/web` builds clean (`npm run build:web` GREEN, Next 16.2.9 webpack + React 19) AND runs clean in dev — **Google login works end-to-end (S7a) and masked/phone inputs render (S7b), both browser-verified by the user**. The two hard blockers are resolved. See the PoC Results section at the bottom for the full verdict.
>
> **Not done (intentional):** S8 teardown — the worktree is kept and the branch was pushed to preserve/share the PoC (the original "throwaway, never pushed" framing was overridden by the user). Also deferred (out of PoC scope): the ~156 test files still on v4 `getServerSession` mocks + old react-input-mask mocks (not in the build/typecheck graph), and the web lint backlog left as warnings (S6c).
>
> **S6c DONE (2026-07-02) — scope narrowed to LISA, config-only.** After starting a repo-wide migration, the user narrowed scope to **LISA (apps/web) + its related packages only**: `apps/web`, `packages/ui`, `packages/eslint-config` (shared), `apps/kiosk` (rides the shared config), and root `package.json` (eslint 9). **`apps/api` and `apps/mcp` were reverted to develop** (standalone NestJS backends, unrelated to LISA's lint). The migration is **config-only — zero source `.ts/.tsx` files touched** — so it can land against an active team without merge conflicts. `apps/web`, `apps/kiosk`, `packages/ui` all lint **exit 0** on ESLint 9 flat config. Details + key findings in the **S6c log** below.
>
> **Reconciled 2026-07-02 (git = source of truth):** the S4/S5/S6b apiAuth work previously logged as "uncommitted in the worktree" was in fact committed as `5a3b424e4` ("PoC S4-S6b checkpoint — Next 16 build GREEN"). Doc was behind git; corrected.
>
> Two runtime blockers (**S7a** NextAuth v4→Auth.js v5; **S7b** react-input-mask) remain user-owned and are the only work left before the app is usable in dev. `npm run build` is GREEN; `npm test` state unchanged (apiAuth fixed; page-tests/blocker-gated suites remain).
>
> **Overall state:** S0–S5 done, `npm run build` GREEN. **Runtime (S6) is blocked by TWO user-owned library upgrades:**
> - **⛔ BLOCKER #2 (larger) — NextAuth v4 vs Next 16:** async `cookies()`/`params` break **all authentication**; app unusable past login. Fix = Auth.js v5 migration (~166 files). TBD, user decision.
> - **⛔ BLOCKER #1 — react-input-mask vs React 19:** `findDOMNode` removed; crashes masked/phone-input screens. Candidates researched. TBD, user decision.
>
> **Docs synced 2026-07-01:** Jira comment `37197` updated in place (both blockers, "one blocker" framing corrected); plan file "Known blockers to success" section added; S2 next-auth mis-assessment corrected below.
>
> **`npm test`:** 13 jsdom suites / 107 tests fail — 54 are BLOCKER #1 (react-input-mask), ~130 are async-params PAGE tests (need `Promise.resolve` + `act()` restructuring), plus `app/layout.test.tsx`. The ~69 API-route param mocks do NOT fail (isolatedModules). Green is impossible until BLOCKER #1 is decided. **`apiAuth.test.ts` (middleware→proxy) already fixed.**
>
> **Unblocked work available now:** finish the **ESLint flat-config + eslint 9** (S6b) — independent of both blockers.
>
> **Pick up here:** S4 ✅, **S5 ✅ — `npm run build` is fully GREEN and clean** (prod proven). S6 🟡 with an open ⛔ **blocker: `react-input-mask` is React-19-incompatible (`findDOMNode`)** — **user-owned, do NOT replace without approval**; candidates researched + documented in the S6 log; user will investigate usage and decide later. What CAN move forward now: user runs `npm run dev` and browser-verifies `/intakes`, `/admin`, `/orders-tracker` (screens with masked/phone inputs will crash until the blocker is resolved); also confirm `react-pdf` renders under forced SWC minify. Then **S6b** (lint flat-config + eslint 9; async-params test mocks). All S4+S5+S6b(apiAuth) fixes are committed in `5a3b424e4` (working tree clean). `apps/web` on **next 16.2.9 / react 19.2.7**; async params migrated in 71 source files; the 3 `assert`→`with` JSON imports fixed. Next is **S4**: add `--webpack` to the build script (keep webpack, defer Turbopack; note the lint codemod already set `build` to `next build --no-lint`), and finish the ESLint flat-config migration — which requires a **repo-wide** eslint 9 move (root `package.json` pins eslint 8.57.1). **Known carry-forwards to S5:** ~69 test files still pass sync `params` (typecheck-only failures; won't block `next build` or runtime); `react-input-mask` React 19 runtime break (deferred, decide at S6). See S1/S2/S3 log entries for detail.
>
> _Update this line at the end of each working session so the next pickup is instant._

## Status legend

⬜ Not started &nbsp; 🟡 In progress &nbsp; ✅ Done &nbsp; ⛔ Blocked

## Status table

| Stage | Status |
|---|:--:|
| **S0 — Setup & isolation** | ✅ |
| **S1 — Core upgrade codemod** | 🟡 |
| **S2 — React 19 dependency audit** | ✅ |
| **S3 — Async params sweep** | ✅ |
| **S4 — Builder + lint config** | ✅ |
| **S5 — Build GREEN (`npm run build` clean)** | ✅ |
| **S6 — Dev GREEN + browser verify (`npm run dev` clean)** | 🟡 ⛔ |
| **S6b — Test-suite consistency (async-params page/test mocks)** | 🟡 |
| **S6c — ESLint 8→9 flat-config migration (LISA scope: web + ui + shared + kiosk)** | ✅ |
| **S7a — Resolve Blocker #2: NextAuth v4 → Auth.js v5** | ✅ |
| **S7b — Resolve Blocker #1: replace react-input-mask** | ✅ |
| **S8 — Findings & teardown** | ✅ findings / ⏸️ teardown skipped (branch pushed) |

(Stage definitions and resumable checkpoints are in the plan file.)

> **Process note (updated 2026-07-02):** the earlier "no per-stage checkpoint commits" override was superseded — the work has been landed in **coarse checkpoint commits** rather than one-per-stage: `fc256252b` (S1–S3) and `5a3b424e4` (S4 + S5 + S6b apiAuth). Resumability relies on the persistent worktree + these commits + this log. Git is the source of truth for HEAD; keep the Resume pointer's HEAD in sync with `git worktree list`.

> **Objective change (2026-07-01, user) — the important one.** The PoC is **not** "prove reachability + catalogue what breaks." It is a **complete, clean migration**: the app must run **error-free in BOTH `npm run dev` and `npm run build`**. Every error found is to be **fixed**, not deferred. This chose **Option A** at the S5 decision (fix everything). Consequences: (1) S5 exit is now `npm run build` fully green — no catalogue-and-move-on; (2) previously **deferred items are now in scope** — `react-input-mask` replacement (S6 runtime), `next.config.mjs` key cleanups (S5), and the ESLint flat-config / eslint 9 finish + async-params test mocks (new **S6b**). The plan file's Goal + Decision + stage table were updated to match.

---

## Per-stage log

### S0 — Setup & isolation ✅

**2026-06-30 — worktree + Next 14 baseline install.** Main repo confirmed clean and up to date with `origin/develop` (0/0). Worktree created at `../li-call-processor-nextjs-v16-poc` on branch `spike/MLID-364-next16-direct-poc` (HEAD `ad58f5d6f`). Baseline `npm install` completed cleanly (exit 0); installed versions confirmed: **next 14.2.33, react 18.3.1, react-dom 18.3.1** (the expected Next 14 baseline). Stopped for the day here.

**Error #1 — missing env files (blocker for the baseline run).** The worktree is a fresh checkout and does **not** carry the gitignored `.env.local` files, so `npm run dev` would not work. Two files exist in the main folder: `./.env.local` (repo root) and `./apps/web/.env.local`.
- **Fix (2026-07-01):** copied both from the main folder into the worktree:
  ```bash
  cp ./.env.local            ../li-call-processor-nextjs-v16-poc/.env.local
  cp ./apps/web/.env.local   ../li-call-processor-nextjs-v16-poc/apps/web/.env.local
  ```
  Verified byte sizes matched (1875 and 3009). Blocker cleared.

**Error #2 — workspace-package build race (`Module not found: Can't resolve '@repo/db'`).** On the first `npm run dev`, the web dev server crashed resolving `@repo/db` (import trace through `services/mongodb/index.ts` → `instrumentation.ts`, which also failed the instrumentation hook).
- **Cause:** turbo's `dev` task has no `dependsOn: ["^build"]`, so `turbo dev` does not pre-build the workspace library packages. It starts every workspace's `dev` script in parallel — including `@repo/db`'s `tsc --watch` (which emits `dist/`) — while the Next.js web compile runs at the same time. On a **fresh** worktree `packages/db/dist` does not exist yet (`dist/` is gitignored), so the web compile reached `chatMessageSyncService.ts` before `tsc --watch` had emitted `dist/index.js`. The main folder never shows this because its `dist/` already exists from earlier builds.
- **Fix:** build web's workspace deps **once** before `npm run dev` (from the worktree root); `dist/` then persists on disk, so this is only needed after a fresh `npm install`:
  ```bash
  npx turbo run build --filter=@repo/web^...   # builds @repo/db, @repo/ui, @repo/shared (deps of web, NOT web itself)
  ```
  Ran successfully on 2026-07-01 — `3 successful, 3 total`.

**S0 complete (2026-07-01) ✅.** env files copied; workspace deps built; `npm run dev` boots the Next 14 baseline; login works and the app renders (home + `/intakes`, `/admin`, `/orders-tracker`) — the "before" reference is captured.

> Note (not a PoC finding): the first login attempt failed with a Mongo `ETIMEDOUT` to the Atlas private endpoint — the local VPN simply wasn't connected at the time. Reconnecting it resolved it immediately. Environmental, unrelated to the upgrade.

### S1 — Core upgrade codemod 🟡 (in progress)

**2026-07-01 — ran the upgrade codemod.**
```bash
cd ../li-call-processor-nextjs-v16-poc
npx --yes @next/codemod@canary upgrade latest
```

**What worked:**
- Resolved and installed **Next.js 16.2.9 + React 19.2.7 + react-dom 19.2.7** into `node_modules` (verified via each package's `node_modules/.../package.json`).
- Selected the recommended codemod set (`next-request-geo-ip`, `next-async-request-api`, `app-dir-runtime-config-experimental-edge`, `next-experimental-turbo-to-turbopack`, `next-lint-to-eslint-cli`, `middleware-to-proxy`, `remove-unstable-prefix`, `remove-experimental-ppr`).
- No unwanted `@vercel/functions` import was introduced.

**Finding — the codemod mis-handles this npm-workspaces monorepo (key PoC learning):**
1. **It bumped the wrong `package.json`.** It appended a brand-new `dependencies` block to the **repo-root** `package.json` pinning `next@16.2.9`/`react@19.2.7`/`react-dom@19.2.7`. The root is a workspaces root and had no such block. The real Next consumer, `apps/web/package.json`, was **left untouched** (still `next 14.2.33`, `react 18`, `react-dom 18`, `@types/react 18.3.3`, `eslint-config-next ^14.2.5`). The 16/19 versions only "took" because npm hoisting resolved the root pin.
2. **It halted on an interactive prompt.** Despite announcing "Running in non-interactive mode," it blocked on `Is your app deployed to Vercel? (Y/n)` from the `next-request-geo-ip` codemod. Because the process stopped there, the **source-code transforms never ran** — `git status` showed only root `package.json` + `package-lock.json` changed; no `middleware→proxy`, no async-params, no app files touched.
3. **EPERM cleanup warning** on `apps/kiosk/node_modules/@next/.swc-...node` (a locked SWC binary) — non-fatal, just a Windows file-lock during npm cleanup.

**Workspace version survey (de-risks the React 19 audit ahead):**
- `apps/kiosk` is **already on Next `^16.2.9` / React `^19`** (`@types/react ^19`, `eslint-config-next ^16.2.9`) — the repo is partially migrated already.
- `packages/ui` already declares a dual `^18.0.0 || ^19.0.0` React peer — compatible with 19 out of the box.
- Still lagging: `apps/web` (14/18) and `apps/storybook` (react `^18`).

**Current worktree state:** `node_modules` on 16/19; root `package.json` polluted with an app-level `dependencies` block; `apps/web/package.json` still on 14/18 (inconsistent); no source transforms applied; nothing committed.

**Correction applied (2026-07-01) — version bump now clean.**
1. ✅ Reverted the erroneous `dependencies` block from the **root** `package.json`. Confirmed via `git diff package.json` — the only remaining diff is a harmless LF→CRLF line-ending warning; no content change (root fully restored).
2. ✅ Bumped `apps/web/package.json` to the real targets: `next` → `16.2.9`; `react`/`react-dom` → `19.2.7`; `@types/react` → `^19`; `eslint-config-next` → `^16.2.9`. (Note: `@types/react-dom` is **not** declared in the web app — only `@types/react` — so it resolves transitively; will add `^19` explicitly only if S5 typecheck complains.)
3. ✅ Reinstalled (`npm install`). Result: `added 3, removed 33, changed 4`, audited 2267 packages. Resolved to a single **hoisted** react 19.2.7 / react-dom 19.2.7 / next 16.2.9, with **no nested react 18 in `apps/web`** (no dual-version conflict).
4. `apps/storybook` (react 18) left for later — outside the web run path.

**Finding — install does not fail on React 19 peer conflicts (matters for the S2 audit).** The worktree's `.npmrc` sets `legacy-peer-deps=true`, so `npm install` completed with **no `ERESOLVE` errors** despite packages like `react-input-mask` / `react-modal` declaring React ≤18 peers. Consequence: **install cleanliness is not a valid signal for the React 19 audit** — peer mismatches will surface at **build / typecheck / runtime** (S5/S6) instead of at install. S2 must inspect peers explicitly rather than trusting a green install.

**Source codemods run individually (2026-07-01).** Each was invoked as its own transform with `--force` (the tree was dirty from the version bump; `--force` lets it run without committing, and every change is still reviewable via `git diff`). Skipped `next-request-geo-ip` (not on Vercel). `next-async-request-api` deferred to **S3**.

- **`middleware-to-proxy` ✅ (applied).** Renamed `apps/web/middleware.ts` → `apps/web/proxy.ts` and the exported `middleware` function → `proxy`. Verified: `export async function proxy(request: NextRequest)`, `export const config` preserved, no lingering `middleware` references in the file. No other file imports the Next root middleware (the "middleware" hits elsewhere are app-level `withApiMiddleware`-style helpers, unrelated).
- **Config cleanups — all no-ops (0 files modified):** `app-dir-runtime-config-experimental-edge` (3501 unmodified), `remove-experimental-ppr` (3501 unmodified), `remove-unstable-prefix` and `next-experimental-turbo-to-turbopack` (0 modified). This codebase does not use experimental-edge runtime, PPR, `unstable_` APIs, or `experimental.turbo`.
- **`next-lint-to-eslint-cli` ⚠️ (partial — carries into S4).** Updated scripts (`lint`: `next lint` → `eslint .`; `lint:fix` → `eslint --fix .`; `build` → `next build --no-lint`), bumped `eslint` to `^9`, removed `@eslint/eslintrc`, and generated `apps/web/eslint.config.mjs`. But it **could not finish the flat-config migration** — it warned `Config does not export an array or supported pattern. Manual migration required.` So `.eslintrc.json` and the new `eslint.config.mjs` **coexist**, and the Next.js configs are not wired into the flat config yet. **→ Finish in S4.**

**Finding — deprecated `assert { type: 'json' }` import attribute in 3 files.** `remove-unstable-prefix` and `next-experimental-turbo-to-turbopack` each threw `SyntaxError: The 'assert' keyword in import attributes is deprecated ... replaced by the 'with' keyword` on 3 files (jscodeshift's Babel parser rejects the old syntax):
- `apps/web/app/orders-tracker/Orders/LeadOrder.tsx`
- `apps/web/app/orders-tracker/Orders/MaintenanceOrders.tsx`
- `apps/web/app/orders-tracker/Orders/NewOrders.tsx`

All three do `import COLUMN_WIDTHS from './orders-tracker-column-widths.json' assert { type: 'json' }`. These sit in the **`/orders-tracker`** route (one of the Option B verification routes), so this likely needs a `assert`→`with` fix for the Next 16 build — **flag for S5** — and it will also block the jscodeshift-based `next-async-request-api` sweep on these files in **S3**.

**S1 change surface (uncommitted on spike branch):**
- `D apps/web/middleware.ts` + `?? apps/web/proxy.ts` (rename)
- `M apps/web/package.json` (version bumps + lint-migration script/dep changes)
- `?? apps/web/eslint.config.mjs` (new, incomplete) — `.eslintrc.json` still present (dual config)
- `M package-lock.json` (reconciled for react/next 16/19)
- `M package.json` (root) — content-clean; the diff is only a LF→CRLF line-ending warning

**Known drift to reconcile:** the lint codemod bumped `eslint` → `^9` in `package.json` **after** the reinstall, so `node_modules` still has `eslint 8.57.1`. A reinstall is needed to reconcile (fold into the **S2** dependency-audit install).

**S1 status:** all S1 codemods applied. Two items intentionally routed forward — the `eslint ^9` reinstall to **S2**, and finishing the ESLint flat config to **S4**. Pending: the plan's S1 checkpoint commit ("S1 core upgrade") — not committed yet (no auto-commit; awaiting explicit go-ahead).

### S2 — React 19 dependency audit ✅ (audit complete; one break carried to S6)

**2026-07-01 — audit method.** Because `.npmrc` has `legacy-peer-deps=true`, `npm install` never fails on peer conflicts, so the audit inspected the **actual peer ranges** of every installed package against `react@19.2.7` (script over `node_modules/*/package.json` `peerDependencies` using `semver.satisfies`), plus a targeted check of the plan's named suspects.

**Hard peer conflicts — 5 packages declare a React peer that excludes 19:**

| Package | Declared react peer | Assessment |
|---|---|---|
| `next-auth@4.24.7` | `^17.0.2 \|\| ^18` | ⚠️ **This S2 assessment was WRONG — corrected at S6.** I judged v4 fine for the PoC because the *React 19 peer* is only conservative. But the real incompatibility is with *Next 16* (async `cookies()`/`params`), which the peer range does not reveal. At S6 runtime, v4 broke **all authentication** — see **BLOCKER #2** in the S6 log. Requires the Auth.js v5 migration. |
| `react-select@5.8.0` | `^16.8 \|\| ^17 \|\| ^18` | v5 works under React 19; newer 5.x updated peers. **Keep / optional minor bump.** |
| `react-modal@3.16.1` | `…^18` | Risk of legacy DOM API use; **verify at S6**, bump if it breaks. |
| `react-collapsible@2.10.0` | `~15 … ~18` | Old; **verify at S6.** |
| `react-spinners@0.14.1` | `^16 \|\| ^17 \|\| ^18` | Pure CSS spinners, no DOM internals; low risk despite conservative peer. **Keep.** |

**Confirmed runtime break (the important one) — `react-input-mask@2.0.4`.** Its peer is a permissive `>=0.14.0`, so the peer audit does **not** flag it — but its `dist` uses **`findDOMNode`, which React 19 removed entirely**. It will throw at runtime whenever a masked input renders. Rendering usage:
- `apps/web/components/UI/MaskedInput/MaskedInput.tsx`
- `apps/web/components/UI/MUI/MuiPhoneField.tsx` (phone field used in forms)
- (`apps/web/components/UI/StyledInput/StyledInput.tsx` imports **types only** — no runtime dependency)

Real fix = replace with a maintained mask library (e.g. `@react-input/mask` or `react-imask`) or MUI-native masking — that is migration work for the real ticket. **Decision for the PoC: document as a confirmed React 19 blocker and let S6 decide urgency** — if none of the three target routes (`/intakes`, `/admin`, `/orders-tracker`) render a masked/phone input, the PoC's Option B bar can still pass and the replacement is logged as required follow-up; if they do, we swap it before S6 can pass.

**Compatible suspects (peer allows 19, no action):** `react-pdf@10.1.0` (`…|| ^19`), `react-day-picker@9.0.9` (`>=16.8.0`), `react-hook-form@7.75.0` (`… || ^19`), `react-markdown@9` (`>=18`), `@mui/material@7.3.10` (`… || ^19`), `@testing-library/react@16` (`^18 || ^19`), `react-icons` (`*`), `react-apexcharts`, `react-modern-drawer`, `react-simple-toasts`.

**Install-clean checkpoint met trivially:** `legacy-peer-deps=true` means the S2 exit criterion ("`npm install` runs with no blocking peer errors") is always satisfied. The real S2 value is this documented audit, not a green install.

**eslint drift — cannot reconcile from `apps/web`; deferred to S4.** A reinstall left `node_modules` on **eslint 8.57.1** despite `apps/web` asking for `^9`, because the repo-**root** `package.json` pins `eslint: 8.57.1` exactly (plus `@eslint/eslintrc`, `@eslint/js` at 8.x) and workspace hoisting makes the root pin win. Moving to eslint 9 is a **repo-wide** change (root + flat config) that belongs to **S4**. Not a PoC-run blocker (dev/build don't lint; `build` is now `--no-lint`).

### S3 — Async params sweep ✅ (source migrated; test mocks are an S5 carry-forward)

**2026-07-01 — pre-step: fixed the deprecated JSON imports.** Before running the sweep, changed `assert { type: 'json' }` → `with { type: 'json' }` in the 3 `/orders-tracker` files (`LeadOrder.tsx`, `MaintenanceOrders.tsx`, `NewOrders.tsx`) so jscodeshift could parse them. Verified 0 `assert { type: 'json' }` occurrences remain. (react-input-mask replacement intentionally **deferred** per user — documented under S2.)

**Sweep result.** Ran `npx @next/codemod@canary next-async-request-api . --force` in `apps/web`:
`0 errors, 3430 unmodified, 1 skipped, 71 ok` — **71 files transformed, no parser errors** (the assert→with fix cleared the earlier blocker).

**Transforms are correct — verified both shapes:**
- **Server components / API routes** → `{ params }: { params: { id } }` becomes `props: { params: Promise<{ id }> }` + `const params = await props.params;`. Example: `apps/web/app/api/admin/api-keys/[id]/route.ts`.
- **Client components** → the codemod imports the React `use` hook and unwraps the promise (`import { …, use }`; `params: Promise<{ id }>`). Example: `apps/web/app/admin/locations/[id]/page.tsx`.

**Finding — test mocks NOT updated (S5 carry-forward).** The codemod touched **0 test files**. **69 test files** under `apps/web/app/**` still pass a **synchronous** `params: { … }` object to the route/page handlers, which now expect `Promise<…>`. Consequence:
- `npm run types:check` (`tsc --noEmit`, includes tests) will report ~69 type mismatches in **S5**.
- `next build` should be unaffected — test files are not in the app build graph — so this does **not** block S5 build or S6 runtime.
- For the PoC we **catalog** these rather than migrate all 69 (that is real work for the production ticket). Sample: `apps/web/app/api/admin/api-keys/[id]/route.test.ts`.

### S4 — Builder + lint config ✅ (builder done; ESLint flat-config finish deferred by decision)

**2026-07-01 — builder: force webpack on both dev and build.** Updated `apps/web/package.json` scripts:
- `dev`: `next dev -p 8080` → `next dev --webpack -p 8080`
- `build`: `next build --no-lint` → `next build --webpack`
- `start`: unchanged (serves build output; no builder flag needed).

**Why `--webpack` on dev too (not just build):** `next.config.mjs` has a custom `webpack()` config (line 45). Next 16 defaults **both** `next dev` and `next build` to **Turbopack**, which **ignores** `webpack()` config. Running dev on Turbopack would silently drop those customizations, so S6 would be verifying a different bundler than the one we intend to keep. Forcing `--webpack` everywhere keeps the whole PoC on webpack and defers the Turbopack migration to a separate future ticket (per plan). Confirmed via `next dev --help` / `next build --help` that `--webpack` is a valid flag in this version.

**Finding — `--no-lint` is not a valid Next 16 build flag.** The S1 lint codemod had set `build` to `next build --no-lint`, but Next 16 removed lint-during-build entirely and `next build --help` lists **no `lint` flag**. So `--no-lint` is redundant (nothing to disable) and a likely unknown-option error. Removed it. (`next build` no longer lints on its own in Next 16.)

**ESLint flat-config finish — deferred (user decision).** Lint is not on the Option B success bar, and finishing the flat config requires a repo-wide eslint 9 move (root `package.json` pins eslint 8.57.1). Left as-is: `eslint.config.mjs` (incomplete) and `.eslintrc.json` coexist; `npm run lint` (`eslint .`) will not work correctly until this is finished. Not a blocker for build (S5) or runtime (S6). Tracked as remaining production-ticket work.

### S5 — Build & typecheck 🟡 (catalogued; fix-vs-bypass decision pending)

**Two environmental / codemod blockers cleared first:**
1. **Stale `.next` lock** — first `npm run build:web` died on `EPERM: open '.next/trace'` (Windows lock left by the earlier stopped dev server; no process was actually running). Fixed by `rm -rf apps/web/.next`. Environmental, not an upgrade issue.
2. **`@next-codemod-error` false positive** — build then failed on unresolved codemod markers in `app/api/orders/new/[orderId]/auth-review/[letterId]/route.ts` (GET + PATCH). The S3 codemod inserted them because `context` is passed into a helper (`resolveOrderPatient(context)`) it couldn't statically verify — but the helper **already** does `await context.params` and `RouteContext.params` is already `Promise<…>`. Code was correct; removed the 2 marker comments (Next 16 treats unresolved markers as hard build errors).

**Build then compiled under webpack** (Next.js 16.2.9, webpack). The `Module not found: 'cpu-features'` line is a harmless optional native dep of `ssh2`, not a real error. The build then failed at the **type-check phase**.

**Typecheck catalog — `tsc --noEmit` (apps/web): 64 errors, all in app source, 0 in tests.** (Note: this **corrects** the S3 expectation — `apps/web/tsconfig` excludes test files, so the ~69 sync-`params` test mocks do **not** fail `types:check`; they will only surface under `npm test`/jest.) Errors by TS code: 41× TS2322 (not assignable), 11× TS2339 (property missing), 4× TS2554 (arg count), 3× TS2503 (namespace), 2× TS2345, 1× each TS2739 / TS2307 / TS18046.

**Root-cause breakdown (a few causes drive most of the 64):**
- **Custom `components/UI/Select/Select.tsx` — 10 errors, and it cascades.** Its props typing resolves to `SelectProps & RefAttributes<any>` under React 19, dropping `options`/`value`/`onChange`, so every consumer (`app/admin/locations/page.tsx`, `AuditLogFilters`, `UsersFilters`, `AcceptedPairsTable`, …) throws TS2322/TS2339. Fixing this one component likely clears a large share of the 41+11 bucket. **Highest-leverage fix.**
- **React 19 removed the global `JSX` namespace — 3× TS2503** (`LocationActionsMenu.tsx`, `FormItem/types.ts`, `DynamicForm.tsx`). Mechanical: `JSX.Element` → `React.JSX.Element`.
- **React 19 `cloneElement`/children typing — TS18046** `child.props is of type 'unknown'` (`ActionsMenuBase.tsx`).
- **Async-params leftover — TS2739** in `app/api/intakes/[id]/submission-state/route.ts:110` (`{ id }` used where `Promise<{ id }>` expected) — a spot S3 didn't cover.
- **MUI dependency path — TS2307** `@mui/x-data-grid-pro/models/gridApiPro` not found (`IntakesFilterBar.tsx`) — internal import path changed in the installed x-data-grid-pro; not React 19.

**`next.config.mjs` cleanups flagged by Next 16 (warnings, non-fatal):** `experimental.instrumentationHook` no longer needed (invalid key now); `swcMinify` removed (invalid key); `experimental.typedRoutes` → top-level `typedRoutes`; `images.domains` → `images.remotePatterns`.

**Key PoC insight:** none of the 64 type errors block **`next dev`** — dev does not type-check. They only block the **production `next build`**. So S6 (Option B: `npm run dev` + routes render) can proceed regardless; the type-error fixes are bounded, well-understood React 19 migration work for the production ticket.

**Decision — user chose Option A (fix everything).** Per the objective change (top of file), the PoC must be a clean migration: fix all 64, get `npm run build` green. Below is the fix work.

#### Option A fixes (2026-07-01) — typecheck 64 → 15 → 2 → (expected 0)

**Root-cause dependency bump cleared 49 of 64:** `react-select` **5.8.0 → 5.10.2** (latest; peer now includes `^19`) in `apps/web/package.json` + reinstall. react-select 5.8's `Props` type collapsed to `{}` under React 19 types, which is why every `<Select>` consumer failed. Bump fixed `Select.tsx` (10) and its whole cascade → **64 → 15**.

**The remaining 15, fixed by category (all classic React 19 / Next 16 changes):**
- **`useRef()` needs an initial arg (React 19)** — 4 files: `utils/hooks/hook-helpers/useThrottle.ts`, `app/intakes/[id]/useActionResult.ts`, `utils/hooks/useFetchData.ts`, `components/Modules/document-intake/components/useSubmissionStatePersisted.ts`. Changed `useRef<T>()` → `useRef<T | undefined>(undefined)` (matches the codebase's own existing style). This also fixed the paired TS2322 in `useThrottle`.
- **Removed global `JSX` namespace (React 19)** — 3 files: `components/UI/FormItem/types.ts`, `components/UICompositions/DynamicForm/DynamicForm.tsx`, `components/Modules/users-locations/components/LocationActionsMenu/LocationActionsMenu.tsx`. Added `import type { JSX } from 'react';` (React 19 re-exposes `JSX` as a named export); kept the `JSX.Element` usages.
- **`NextRequest.ip` removed (Next 16)** — `app/api/files/stream/route.ts`: `req.ip` → `req.headers.get('x-real-ip')` fallback (after `x-forwarded-for`).
- **Async-params leftover** — `app/api/intakes/[id]/submission-state/route.ts`: `PATCH` was awaiting params then calling `POST(request, { params })` (resolved obj) — changed to forward the promise-bearing `props`: `return POST(request, props);`.
- **MUI import path change** — `app/intakes/components/IntakesFilterBar.tsx`: `GridApiPro` no longer at `@mui/x-data-grid-pro/models/gridApiPro`; imported from the package root `@mui/x-data-grid-pro`.
- **`cloneElement` children typing → `unknown` (React 19)** — `components/UI/ActionsMenuBase/ActionsMenuBase.tsx`: cast `(child.props as { onClick?: () => void }).onClick`.
- **`RefObject<T>` → `RefObject<T | null>` (React 19 `useRef`)** — `AudioPlayer.tsx` + `PlayControls.tsx` prop types changed to `RefObject<HTMLAudioElement | null>`; then `PlayControls.tsx` `scrollRecord` uses the already-narrowed `audio` const instead of re-reading `audioRef.current` (the final 2 errors).
- **DataGrid `getActions` return type (React 19 `ReactElement` default `unknown`)** — `BiosimilarsTable.tsx` + `TreatmentsTable.tsx`: typed the actions array as `React.ReactElement<GridActionsCellItemProps>[]` and imported `GridActionsCellItemProps` from `@mui/x-data-grid`.

**State at pause:** last confirming `tsc --noEmit` run was interrupted; expected 0. Then `npm run build:web` for the green build, plus the pending `next.config.mjs` key cleanups. Files touched are uncommitted in the worktree.

#### S5 GREEN — resolved 2026-07-01 ✅

- **Typecheck: 0 errors** (`tsc --noEmit`, exit 0) — all 64 fixed.
- **`npm run build:web`: GREEN** (exit 0, `4 successful, 4 total`). Full production build on **Next.js 16.2.9 (webpack) + React 19**: compiled, type-checked, **static pages 100/100**, page optimization finalized, route table printed (incl. `ƒ Proxy (Middleware)` — confirms the `middleware`→`proxy` rename works at build time).
- **`next.config.mjs` cleaned** (Next 16): removed `swcMinify: false` (removed key) and `experimental.instrumentationHook` (default now); moved `typedRoutes` to top-level; dropped the redundant `images.domains` (the existing `remotePatterns: [{ hostname: '**' }]` already covers `lh3.googleusercontent.com`). Re-ran build → **no config warnings remain**.
- **Remaining non-fatal warnings are pre-existing, not migration issues:** the `cpu-features` webpack "module not found" (optional native dep of `ssh2`, present on Next 14 too) and the Mongoose `errors` reserved-pathname warnings. Left as-is (out of Next 16 scope; silencing `cpu-features` would be a separate `IgnorePlugin` tweak if desired).
- **Watch item for S6:** `swcMinify: false` had a comment "required for react-pdf v10". Next 16 always applies SWC minification (no opt-out) — the **build** succeeded with it, but confirm `react-pdf` renders correctly at **runtime** during S6 browser verification.

**S5 exit bar met: `npm run build` is clean and green.** Dependency added by S5: `react-select` bump (announce for Azure? no — it's a normal npm dep, no env var). Files touched remain uncommitted in the worktree.

### S6 — Dev GREEN + browser verify (`npm run dev` clean) 🟡 (blocked on the react-input-mask decision)

#### ⛔ BLOCKER (open, TBD — user-owned) — `react-input-mask` incompatible with React 19

**Problem.** `react-input-mask@2.0.4` uses `ReactDOM.findDOMNode`, which **React 19 removed**. It does not crash the build (types resolve; peer is a permissive `>=0.14.0`) but it **crashes at runtime** wherever a masked input renders. This is a **blocker to declaring the migration successful** — `npm run dev` cannot be certified clean on any screen that renders these inputs.

**Known render sites** (full usage-tree investigation is the **user's** TODO, not done yet):
- `apps/web/components/UI/MaskedInput/MaskedInput.tsx`
- `apps/web/components/UI/MUI/MuiPhoneField.tsx` (phone field; MUI `TextField` via `inputComponent` adapter)
- `apps/web/components/UI/StyledInput/StyledInput.tsx` — **types only** (`Props`, `InputState`); no runtime dependency
- (whether these appear on the 3 target routes `/intakes`, `/admin`, `/orders-tracker` is still TBD)

**Ownership / process.** Replacing this component is **the user's decision, requiring explicit approval** — not something the assistant does unilaterally. Left as **TBD**. The assistant's role here is limited to **researching candidates** (below); the user will investigate usage and choose the path later.

**Candidate replacements (research only — 2026-07-01, NOT a recommendation/decision):**

| Candidate | Version | React 19 | Migration effort | Notes |
|---|---|---|---|---|
| **`@mona-health/react-input-mask`** | 3.0.3 | ✅ (created specifically to remove `findDOMNode`) | **Lowest — near drop-in** (same API as the original) | Maintained fork of `sanniassin/react-input-mask`. Child must be a function component with `forwardRef` (MUI's `inputComponent` adapter already qualifies). Smallest change; keeps existing mask logic. |
| **`react-imask` (IMask)** | current | ✅ | Medium (API rewrite) | **MUI's own docs use `react-imask`** for masked `TextField` via `inputComponent` — aligns with `MuiPhoneField.tsx`. Powerful, well-maintained. |
| **`@react-input/mask`** | 2.x | ✅ | Medium (API rewrite) | Modern, TS-first, hooks (`useMask`) + component; no deprecated APIs. "Most current" option. |
| **`cleave.js`** | 1.6.0 | ⚠️ | Medium | **Already a dependency** in this project — but the library is essentially unmaintained/archived. Present-but-stale; not ideal for "current". |
| **`maska`** | current | ✅ | Medium | Lightweight, simple dynamic masking. |

Sources: [@mona-health/react-input-mask (npm)](https://www.npmjs.com/package/@mona-health/react-input-mask), [@react-input/mask (npm)](https://www.npmjs.com/package/@react-input/mask), [React 19 findDOMNode removal (facebook/react #28926)](https://github.com/facebook/react/issues/28926), [MUI Text Field — masked input](https://mui.com/material-ui/react-text-field/), [npm-compare: masking libraries](https://npm-compare.com/maska,react-input-mask,react-maskedinput).

**Status:** blocker parked. S6 browser verification of routes that do **not** render masked inputs can still proceed; any route that crashes on a masked/phone input is expected until this is resolved.

#### ⛔ BLOCKER #2 (open, TBD — user-owned, LARGER) — NextAuth v4 incompatible with Next 16 (surfaced 2026-07-01 in the dev runtime pass)

**Problem.** On the first real login attempt in dev, `/api/auth/[...nextauth]` returns 500 and the app redirects to `/api/auth/error`. Server logs:
- `Route "/api/auth/[...nextauth]" used 'params.nextauth'. params is a Promise and must be unwrapped...`
- `cookies() returns a Promise... TypeError: cookies(...).getAll is not a function`

**Cause.** `next-auth@4.24.7`'s own Next.js integration (`next-auth/next`) reads Next 16's now-**async** `cookies()` and route `params` **synchronously**. Confirmed the call originates **inside next-auth**, not our code (our `[...nextauth]/route.ts` never calls `cookies()`). NextAuth v4 is fundamentally incompatible with Next 16's async request APIs. This **breaks all authentication** — the app cannot be used past the login page, so the S6 "dev runs clean" bar cannot be met until this is resolved.

**Severity vs blocker #1.** Bigger than `react-input-mask`: that only crashes screens with masked inputs; this blocks **login itself**, i.e. the entire app. It also means the green S5 `npm run build` proves the app **compiles/bundles**, not that it is **usable** at runtime.

**Correction to the record.** This is the second blocker. My earlier "only one blocker" (react-input-mask) was stated with the caveat that in-browser runtime verification was still pending — that pending step is exactly what surfaced this.

**Fix path = Auth.js v5 (NextAuth v5) migration — user-owned, major.** No v4 release handles async `cookies()`; v5/Auth.js is the Next 15/16-compatible line. It is a **repo-wide** migration: root-level `auth.ts` config, `auth()` replacing `getServerSession`, restructured `[...nextauth]` handler, and a changed session cookie name (existing sessions are invalidated). **Blast radius: ~166 non-test files import `authOptions`/`getServerSession`.** Per the no-library-swap-without-approval rule, this is **TBD pending user decision** — assistant does not perform it; research only.

Sources: [Auth.js — Migrating to v5](https://authjs.dev/getting-started/migrating-to-v5), [Auth.js reference (Next.js)](https://authjs.dev/reference/nextjs), [NextAuth v4 upgrade guide](https://next-auth.js.org/getting-started/upgrade-v4).

**Jira:** documented on MLID-364 as a comment (id `37197`, 2026-07-01) — originally covered the react-input-mask blocker only. **Needs updating** to add this NextAuth v4 blocker and correct the "one blocker" framing (update in place with `commentId: 37197`; pending user go-ahead).

### S6b — Everything current (lint flat-config, test mocks) 🟡 (test-suite green is BLOCKED by react-input-mask)

**2026-07-01 — started with the test mocks; investigation reshaped the task.**

**Real test failure fixed:** `lib/apiAuth.test.ts` referenced the middleware module by its old name — dynamic `import('../middleware')` + `mod.middleware`. Updated to `import('../proxy')` + `mod.proxy` (the S1 rename). This suite now passes.

**Finding — the ~69 API-route `params` mocks do NOT fail.** `apps/web/tsconfig.json` has `isolatedModules: true`, so ts-jest **transpiles without type-checking**; and route tests call handlers with a plain `{ params: {...} }` while the handler does `await props.params` — `await` on a plain object resolves fine. So these mocks are type-inaccurate but harmless; **no runtime failure, nothing enforces their types**. Updating them is optional cosmetics, not required for green.

**Full-suite result (`npx jest`, both projects): 13 suites / 107 tests fail — all jsdom.** Breakdown by cause:
- **54× `TypeError: reactDom.findDOMNode is not a function` → the react-input-mask ⛔ BLOCKER.** Component/page tests that render masked or phone inputs crash. **These cannot pass until the react-input-mask decision is made.** → **This gates a green `npm test`: S6b cannot fully complete while that blocker is open.**
- **~130× `An unsupported type was passed to use(): [object Object]` → async-params PAGE tests** (5–6 `[id]/page.test.tsx` suites: ai-chat-settings, locations, drug-admin-panel drugs, insurance-payors guidelines, insurance-plans). Pages now do `const params = use(props.params)`; tests pass a plain object. Swapping to `params={Promise.resolve({...})}` clears the `use()` crash **but is not sufficient** — the async `use()` resolution then triggers `An update ... was not wrapped in act(...)` and content never appears, so these need the render restructured for the async page (proper `act()`/async handling), per suite. Attempted on the locations suite, then **reverted** (incomplete) to keep the tree coherent.
- **~10× `reactRender is not a function`** and **~8× `RangeError: Maximum call stack exceeded`** in `app/layout.test.tsx` (RootLayout) — React 19 render-helper / layout issues, not yet investigated.

**Conclusion / status:** the only clean, verified S6b test fix so far is `apiAuth.test.ts`. **A green `npm test` is blocked by react-input-mask** (54 failures), so S6b test work cannot reach green until that user-owned decision lands. The page-test async-`use()` migration is real per-suite work (Promise.resolve for params **plus** act()-wrapped async render) and is documented here as the approach. The **ESLint flat-config / eslint 9** half of S6b is independent of this blocker and remains completable.

### S6c — ESLint 8→9 flat-config migration (LISA scope) ✅ (done 2026-07-02, config-only)

#### RESULT (2026-07-02) — done, scope narrowed to LISA, zero source files touched

**Scope decision (user, 2026-07-02).** The repo-wide framing below was **narrowed**: migrate only **LISA (apps/web) and its related packages**, revert everything else. Rationale from the user: *the more source files we touch, the harder the real migration is to land against a team pushing PRs hourly.* Final kept vs reverted:

| Kept (migrated to ESLint 9 flat config) | Reverted to develop |
|---|---|
| `apps/web` — new `eslint.config.mjs`, `.eslintrc.json` deleted | `apps/api` (standalone NestJS; its own flat config, unrelated to web) |
| `packages/eslint-config` (shared) — `base.js`/`nextjs.js` → flat | `apps/mcp` (standalone NestJS) |
| `packages/ui` — new flat config (was config-less) | |
| `apps/kiosk` — new flat config (rides the shared config) | |
| root `package.json` — `eslint`/`@eslint/js` → 9, `@eslint/eslintrc` → 3 | |

**Config-only — no source churn (the key strategic finding).** The entire pre-existing lint **backlog was handled with configuration, not source edits**, so the migration PR touches only ~12 config files (which feature PRs rarely edit) and will not conflict with an active team:
1. **Ignore the folders `next lint` never linted** — `services/`, `utils/`, `db-update/`, `scripts/`, `buildserver/`, `e2e/`, `tests/`, `__mocks__/`, and root tooling files. This reproduces the *exact* pre-Next-16 lint scope (`next lint` only covered `app/pages/components/lib/src`).
2. **Downgrade the pre-existing noisy rules to `warn`** in web's config (`no-console`, `prettier/prettier`, `@typescript-eslint/no-unused-vars`, `no-constant-binary-expression`). Formatting is still enforced by the existing husky + lint-staged `prettier --write` pre-commit hook, so nothing regresses.
3. **Register `eslint-plugin-jsx-a11y`** in the shared config (as the old `eslint-config-next` bundled it) so leftover inline `eslint-disable jsx-a11y/*` comments resolve instead of erroring "Definition for rule not found". (`eslint-plugin-import` was NOT needed — its only inline-disable references were in now-ignored tooling folders.)

**Result:** `apps/web`, `apps/kiosk`, `packages/ui` all lint **exit 0** on ESLint 9.39.4 flat config. Web = **0 errors, ~248 warnings** (the documented backlog, cleaned incrementally later). **Zero `.ts/.tsx` source files touched** — the diff is entirely `eslint.config.mjs` / `package.json` / deleted `.eslintrc.json`.

**Why the whole "next lint → eslint ." backlog appeared (root cause).** Next 16 **removed `next lint`**. On develop, web's lint was `next lint`, whose default scope is only `app/pages/components/lib/src` — it never linted `services/` or `utils/`. The migration changed the script to `eslint .` (whole app), newly surfacing ~800 pre-existing issues (366 `console` calls, 289 formatting, etc.) that are **unrelated to the Next 16 upgrade**. Ignoring the never-linted folders + warn-downgrades restores parity without fixing that debt.

**Key technical notes for the real migration:**
- `@typescript-eslint/no-var-requires` was **removed** in typescript-eslint v8 → use `@typescript-eslint/no-require-imports`.
- `eslint-plugin-rulesdir` has no reliable ESLint 9 flat path → register the custom `no-direct-drizzle-write` rule as a local `local/` plugin inside the flat config instead.
- The custom rule's `context.getFilename()` is deprecated in ESLint 9 → `context.filename`.
- `next lint` is gone in Next 16; kiosk's `--ext` flag is also gone in ESLint 9 (`eslint .` only). Prettier options should NOT be hard-coded in `eslint-plugin-prettier` — let it read `.prettierrc` (which carries the `organize-imports` / `sort-json` plugins) so eslint and `npm run format` never ping-pong.
- **Perf flag:** `apps/web` `eslint .` takes ~2 min (eslint-plugin-prettier runs Prettier per file). Worth a CI-time look, but not a blocker.

**Committed:** S6c landed as checkpoint `cd5700508` ("chore(lint): migrate LISA … to ESLint 9 flat config"), 14 files, config-only. `apps/api` + `apps/mcp` are back at their `5a3b424e4` state. (The husky pre-commit hook ran prettier over the staged config files.)

---

#### (Original repo-wide scoping — superseded by the narrowed LISA scope above; kept for context)

#### Obstacle — ESLint 9 flat-config is a REPO-WIDE migration, not a web-only "finish the flat config"

**How it surfaced.** Trying to complete the S1 lint codemod's half-done flat config for `apps/web`, I found the config can't run: the generated `apps/web/eslint.config.mjs` imports from `eslint/config` (an **eslint 9** API) and `sourceType: "script"` (wrong — should be `module`), but `node_modules` still has **eslint 8.57.1** because the repo-root `package.json` pins `eslint@8.57.1` exactly and hoisting wins.

**Root finding — the whole monorepo is ESLint 8 + legacy `.eslintrc`:**
- Shared `@repo/eslint-config` (`base.js`, `nextjs.js`) is **legacy `.eslintrc`-style** (`module.exports`, `extends`), peer `eslint: ^8.0.0`, built on `@typescript-eslint@7.16.0`, `eslint-plugin-rulesdir` (+ custom `no-direct-drizzle-write` rule), `next/core-web-vitals`, `plugin:@next/next/recommended`, `react`, `react-hooks`, `prettier`.
- Root pins `eslint@8.57.1`, `@eslint/eslintrc@2.1.4`, `@eslint/js@8.57.1`.
- `apps/api`, `apps/mcp`, `apps/kiosk` are all on **eslint 8** using the same shared legacy config.
- **Only `apps/web` was bumped to eslint 9 + flat config — accidentally, by the S1 codemod**, not a decision. Its flat config only loads the legacy shared config via `FlatCompat`.

**Decision (user, 2026-07-01):** Next 16 means we are **not staying on eslint 8** — do the **full repo-wide ESLint 9 + flat-config migration** (Option B). Not reverting web to eslint 8. The PoC's job is to solve every blocker, not dodge it.

**Scope of S6c (execute next session):**
1. Convert `@repo/eslint-config` (`base.js` + `nextjs.js`) from legacy `.eslintrc` to **flat config** (export config arrays; replace `extends` strings with imported flat configs / `FlatCompat` where a plugin has no flat export yet).
2. Bump `@typescript-eslint` 7.16 → 8.x (eslint 9 requires v8) in the shared config and wherever pinned; bump `eslint` 8.57.1 → 9.x at the **root** and every app; add `@eslint/js` 9.
3. Verify each plugin under eslint 9: `eslint-plugin-rulesdir` (+ the custom `no-direct-drizzle-write` rule — may need reauthoring for flat config / new rule API), `eslint-plugin-react`, `eslint-plugin-react-hooks`, `eslint-plugin-prettier` / `eslint-config-prettier`, `eslint-config-next` 16 (flat entry).
4. Fix the web flat config: `sourceType` `script` → `module`; drop the FlatCompat bridge once the shared config is flat.
5. Delete the dead legacy `.eslintrc.json` files (web + others once migrated).
6. Convert `apps/api`, `apps/mcp`, `apps/kiosk` configs to flat too (they share `@repo/eslint-config`).
7. Green bar: `npm run lint` (turbo, all workspaces) passes on ESLint 9 flat config.

**Risk notes:** `@typescript-eslint` 7→8 is a major bump (rule behavior/renames may surface new lint errors to fix); `eslint-plugin-rulesdir` eslint-9 support is uncertain (the custom drizzle rule may need porting to the modern rule format); airbnb-style/other legacy plugin configs (if any in the apps) may lack flat support and need FlatCompat. Expect follow-on lint-error fixes once it runs.

**Status:** scoped and parked; **continue tomorrow (2026-07-02)**. No code changed yet for S6c.

### S7a — Resolve Blocker #2: NextAuth v4 → Auth.js v5 ✅ (done + runtime-verified 2026-07-02)

#### RESULT (2026-07-02) — done, committed `41e54b08b`

**Verified working at runtime.** After the migration, Google login completes end-to-end on Next 16: v5 mints `authjs.session-token` (confirmed live in the browser cookie store, alongside the now-ignored stale `next-auth.*` v4 cookies), the `proxy.ts` middleware `getToken` reads it, and the authenticated app shell renders. The old v4 failure (`cookies() returns a Promise` / 500 at the OAuth callback) is gone.

**Scale (actual vs. estimate):** the plan estimated ~166 files; the real blast radius was **336 `getServerSession` files + 304 `authOptions` imports**. Split: **182 non-test source files** (migrated) + **156 test files** (deferred — not in the build/typecheck graph, already red from other blockers, same treatment as S6b).

**Library decision:** `next-auth@5.0.0-beta.31`, **pinned exact** (no range). There is no stable v5 — `beta.31` is the de-facto production Next 14/15/16 line. User approved after being shown the beta-only status + 2× blast radius.

**What was done (all in `41e54b08b`):**
1. New root `apps/web/auth.ts` — `NextAuth()` exporting `{ handlers, auth, signIn, signOut }`; Google + Credentials providers, MongoDB adapter, jwt session, and signIn/redirect/jwt/session callbacks ported verbatim; `trustHost: true`.
2. `[...nextauth]/route.ts` → `export const { GET, POST } = handlers`.
3. `proxy.ts` — `getToken({ req, secret })` (secret now required in v5); kept getToken (not the `auth()` wrapper) so the middleware stays edge-safe and never imports the Node-only MongoDB adapter.
4. Codemod over 182 source files: `getServerSession(authOptions)` / `getServerSession()` → `auth()`; `getServerSession(req, res, authOptions)` (Pages Router) → `auth(req, res)`; imports rewired to `import { auth } from '@/auth'`.
5. Type augmentation `types/next-auth.d.ts` — unchanged, still valid under v5.
6. One real type fix (debug log passing `Session | null` to `logger.info`).
7. `next.config.mjs` — `transpilePackages: ['next-auth', '@auth/core']` + `experimental.esmExternals: 'loose'` (v5 is ESM-only; the webpack build otherwise externalizes it as CommonJS and fails with "ESM packages need to be imported").

**Gotchas found (useful for the real migration):**
- v5 is **ESM-only** → the webpack build needs `transpilePackages` + `esmExternals: 'loose'`; without them it fails on `pages/api/*.test.ts` files (which Next builds as routes because `pageExtensions: ['ts','tsx']`).
- The codemod initially inserted `import { auth }` *inside* multiline first-imports; a follow-up pass relocated it before the first `import` statement.
- Session **cookie name changes** (`next-auth.` → `authjs.`) → existing sessions invalidated (one-time re-login). **No new required env vars** (kept `NEXTAUTH_SECRET` + Google vars).
- Pre-existing smell now visible: `pages/api/**/*.test.ts` (29 files) are built as live API routes. Not introduced here; separate cleanup.

**`npm run build:web` GREEN** (Next 16.2.9 webpack, 4/4); `tsc --noEmit` clean.

---

#### (Original plan — done as described above)

Resolves **BLOCKER #2** (documented in the S6 log): `next-auth@4` breaks all authentication on Next 16 (async `cookies()`/`params`). Migration to Auth.js v5 (`next-auth@5`):
- Root-level `auth.ts` config carrying over the current providers (Google + Credentials), the MongoDB adapter, `session: { strategy: 'jwt' }`, and the existing `signIn` / `session` / `jwt` / `redirect` callbacks from `app/api/auth/[...nextauth]/route.ts`.
- Restructure the `[...nextauth]` route to the v5 handler; export `auth`, `handlers`, `signIn`, `signOut`.
- Replace `getServerSession(authOptions)` → `auth()` across **~166 files**; update imports.
- Update `proxy.ts` (middleware) auth logic to v5.
- Note: v5 changes the session cookie name → existing sessions invalidated (acceptable for the spike).
- Verify: dev login end-to-end (Google + credentials), role-gated menu renders.
- **Needs user approval on the v5 approach before starting** (library major migration). No code changed yet.

### S7b — Resolve Blocker #1: replace react-input-mask ✅ (done + runtime-verified 2026-07-02)

#### RESULT (2026-07-02) — done, committed `d2cff3bf4`

**Verified working at runtime (user, browser).** After S7a made login work, the `findDOMNode` crash surfaced in the global header (GlobalSearch → MaskedInput) on every authenticated page. After the swap, the header renders and the **intake create-patient phone field** (`/intakes/[id]/review`, MuiPhoneField) masks correctly — both confirmed by the user.

**Library:** `@mona-health/react-input-mask@3.0.3`, pinned (user-approved). Maintained fork of react-input-mask built specifically to remove `findDOMNode`.

**Key finding — it is NOT a pure drop-in.** The import-path swap alone left a *different* runtime crash (`Cannot read properties of undefined (reading 'disabled')`). The fork changed two API surfaces vs. the original:
1. **`children`: render-prop function → single `forwardRef` element.** The fork clones the child with the masked input props (`children: PropTypes.element`). `MaskedInput` was passing a function → crash. Fixed by passing `<StyledInput />` as an element child.
2. **ref: `inputRef` prop → the `ref` on `<InputMask>` itself** (the fork is a `forwardRef` component; its internal refCallback forwards the input node to `forwardedRef`). `MaskedInput` moves `ref` onto `<InputMask>`; `MuiPhoneField` switched `inputRef=` → `ref=`.

**Affected render sites (all fixed):** `MaskedInput.tsx` (used app-wide via the Header GlobalSearch + inside `DatePicker`) and `MuiPhoneField.tsx` (document-intake + patient prior-auth). `StyledInput.tsx` (types only) re-typed to `InputState`. **`MaskedInputExtended` uses `cleave.js` — a different library, unaffected.**

**Type shim added** (`types/mona-health-react-input-mask.d.ts`): the fork ships only `declare const InputMask: any` and drops the `Props`/`InputState` exports (and `@types/react-input-mask` left with the old package). The augmentation restores both types, input-attribute shaped with the same permissive index signature the originals had, so all ~30 call sites compile untouched.

**`npm run build:web` GREEN** (Next 16.2.9 webpack); `tsc --noEmit` clean (0 errors).

---

#### (Original plan — done as described above)

Resolves **BLOCKER #1** (documented in the S6 log): `react-input-mask@2.0.4` uses `findDOMNode` (removed in React 19), crashing masked/phone inputs. After the user reviews usage and picks a replacement (candidates in the Blocker #1 table: `@mona-health/react-input-mask` near-drop-in, `react-imask`, `@react-input/mask`):
- Rewrite `apps/web/components/UI/MaskedInput/MaskedInput.tsx` and `apps/web/components/UI/MUI/MuiPhoneField.tsx` (+ any other render sites found during usage review) onto the chosen library; keep masking behavior (phone mask etc.).
- Verify masked/phone inputs render + behave at runtime; the ~54 blocker-gated jsdom test failures should clear.
- **Needs user approval on the library before starting.** No code changed yet.

### S8 — Findings & teardown ⬜
—

---

## PoC Results

**Verdict: Next.js 16 + React 19 is reachable AND the app runs clean — the migration is real and viable.** Both `npm run build:web` (production, webpack) and `npm run dev:web` (runtime) are green, with login and masked inputs browser-verified. The two hard blockers were third-party library upgrades forced by the framework/React bump, both now resolved.

**React 19 dependency audit outcome:** Most libraries were compatible or fixed by minor bumps. Two required real work: (1) `react-select` 5.8 → 5.10.2 (type collapse under React 19, fixed at S5); (2) `react-input-mask` → `@mona-health/react-input-mask` (removed `findDOMNode`; **Blocker #1**, S7b). Framework-level: `next-auth@4` → Auth.js v5 (`next-auth@5.0.0-beta.31`; async cookies/params; **Blocker #2**, S7a). `next-auth` v5 is **beta-only** (no stable v5) — pinned to the exact version.

**Builder (`--webpack`) outcome:** Kept webpack (added `--webpack` to dev + build); Turbopack deferred to a separate future ticket. Auth.js v5 is ESM-only, which required `transpilePackages: ['next-auth','@auth/core']` + `experimental.esmExternals: 'loose'` in `next.config.mjs`.

**Build / typecheck outcome:** GREEN. 64 React 19 / Next 16 type errors fixed at S5; `next.config.mjs` cleaned of removed/renamed keys; async `params` migrated across 71 files (S3); `middleware` → `proxy` (S1). Deferred to the real ticket (not in the build/typecheck graph, already red): ~156 test files still on v4 auth mocks / old mask mocks.

**Runtime / browser verification:** ✅ Google login works end-to-end (v5 `authjs.session-token` set, proxy honors it). ✅ Header GlobalSearch + intake create-patient phone field render and mask correctly. The three original target routes (`/intakes`, `/admin`, `/orders-tracker`) are reachable now that the app shell no longer crashes.

**Lint:** ESLint 8 → 9 flat-config migration (S6c) landed **config-only** (no source churn) so it can merge against an active team; scoped to LISA + related packages (`apps/web`, `packages/ui`, `packages/eslint-config`, `apps/kiosk`, root). `apps/api`/`apps/mcp` left as-is.

**Cost summary (what a real migration must budget for):**
- Core codemods (async params, middleware→proxy) — mostly mechanical.
- ~64 React 19 type fixes + a `react-select` bump.
- ESLint 9 flat-config migration (config-only if the pre-existing lint backlog is kept as warnings).
- **Auth.js v5 migration — the big one: ~336 files touched (getServerSession→auth), a beta dependency, changed session cookie (one-time re-login).**
- react-input-mask replacement — small, but the fork is not a pure drop-in (children + ref API changed).

**Branch:** pushed to `origin/spike/MLID-364-next16-direct-poc` (preserved, not torn down) so the team can review the concrete change-set as evidence.

---

## End-of-Project Conclusions and Pending Issues

> **Two different objectives, do not conflate them.**
> - **PoC objective (this spike, MLID-364):** *prove that LISA can run on Next.js 16.* — **MET.**
> - **Real migration objective (the production MLID-364 work, planned separately):** *have LISA successfully and durably running on Next.js 16 in production* — the same green bar the team relies on every day (build, dev, **full test suite**, lint, all workspaces, all runtime surfaces). The "not yet resolved" list below is the gap between those two objectives and becomes the backlog for the real migration plan.

### What was successfully achieved

The PoC proved LISA can run on Next 16. Concretely:

- **The hardest path was taken and it worked** — a single **direct 14 → 16 / React 18 → 19 jump** (no staging), run on an isolated git worktree so breakage was free.
- **Production build is GREEN** — `npm run build:web` completes with no errors on **Next.js 16.2.9 (webpack) + React 19.2.7**. All **64** React 19 / Next 16 type errors were fixed (not catalogued), and `next.config.mjs` was cleaned of every removed/renamed key.
- **Dev runtime is clean and browser-verified** — `npm run dev:web` boots clean; **Google login works end-to-end** and **masked/phone inputs render correctly**, both confirmed live in the browser by the user. The three target routes (`/intakes`, `/admin`, `/orders-tracker`) are reachable.
- **Both hard blockers were resolved** (each a user-owned third-party upgrade):
  - **NextAuth v4 → Auth.js v5** (`next-auth@5.0.0-beta.31`, pinned) — async `cookies()`/`params` incompatibility fixed. Real blast radius **~336 files** (S7a, `41e54b08b`).
  - **`react-input-mask` → `@mona-health/react-input-mask@3.0.3`** — removed the React-19-incompatible `findDOMNode` (S7b, `d2cff3bf4`).
- **Mechanical migrations completed** — async `params` migrated across **71 source files** (S3); `middleware` → `proxy` rename (S1); `react-select` 5.8 → 5.10.2 (React 19 type collapse, S5).
- **Lint moved to ESLint 9 flat config** for the LISA scope (`apps/web`, `packages/ui`, `packages/eslint-config`, `apps/kiosk`, root) — **config-only, zero source files touched** (S6c, `cd5700508`), so it can land against an active team without conflicts.
- **Evidence preserved** — spike branch pushed to `origin/spike/MLID-364-next16-direct-poc`, worktree kept, so the team can review the concrete change-set.

**Conclusion: the PoC objective is met — LISA runs on Next 16, in both build and dev, with the two library blockers solved.**

### What is not yet resolved and will be needed to address soon (real-migration backlog)

Acceptable to leave open for a PoC, but **must be closed for the real migration** (LISA durably running on Next 16). Grouped by area:

1. **Test suite — the largest gap. `npm test` is not green.**
   - **~130 async-params PAGE tests** (`[id]/page.test.tsx`: ai-chat-settings, locations, drug-admin-panel drugs, insurance-payors guidelines, insurance-plans) need the render restructured for React 19 async `use()` — `params={Promise.resolve({...})}` **plus** `act()`-wrapped async render. Per-suite work, not a find-and-replace (see S6b log).
   - **~156 test files still mock NextAuth v4** (`getServerSession`) — must migrate to Auth.js v5 `auth()` mocking (see S7a: 156 test files deferred).
   - **~54 masked-input test failures** still mock the old `react-input-mask` — must move to the `@mona-health` fork (see S6b / S7b).
   - **~69 API-route `params` mocks** pass synchronous `params` — currently harmless (`isolatedModules` transpiles without type-checking), optional type-accuracy cleanup (see S3 / S6b).
   - **`app/layout.test.tsx` (RootLayout)** — `reactRender is not a function` + `RangeError: Maximum call stack exceeded`; React 19 render-helper/layout issue, **not yet investigated** (see S6b log).

2. **Auth.js v5 hardening.**
   - Running on a **beta dependency** (`next-auth@5.0.0-beta.31`, no stable v5) — revisit when v5 goes stable.
   - **Session cookie name changed** (`next-auth.*` → `authjs.*`) → all existing sessions invalidated on deploy (one-time forced re-login) — plan and communicate for production.

3. **Turbopack migration.** PoC kept webpack (`--webpack` on dev + build). The **webpack → Turbopack** migration was deferred to a separate future ticket.

4. **Lint backlog (real cleanup, deferred by design).**
   - The ESLint 9 landing was config-only: it **ignored folders `next lint` never linted** (`services/`, `utils/`, `db-update/`, `scripts/`, `buildserver/`, `e2e/`, `tests/`, `__mocks__/`) and **downgraded noisy rules to `warn`**. The underlying backlog (**~800 pre-existing issues**: ~366 `console` calls, ~289 formatting, etc.; ~248 web warnings) is untouched (see S6c log).
   - **`apps/api` and `apps/mcp` were reverted to develop** — still on ESLint 8 legacy `.eslintrc`; their flat-config migration is not done.
   - **Perf:** `apps/web` `eslint .` runs ~2 min (Prettier per file) — worth a CI-time look.

5. **Other workspaces / surfaces.**
   - **`apps/storybook`** still on **React ^18** — not migrated (outside the web run path).
   - **`pages/api/**/*.test.ts` (29 files) are built as live API routes** (because `pageExtensions: ['ts','tsx']`) — a pre-existing smell surfaced by the v5 ESM build, needs separate cleanup (see S7a gotchas).

6. **Runtime verification gaps (broaden before production).**
   - **`react-pdf` under forced SWC minify** — Next 16 always applies SWC minification (the old `swcMinify: false` was "required for react-pdf v10"); build passed but a dedicated runtime confirmation of react-pdf rendering is still open (S5 watch item).
   - **`react-modal@3.16.1`** and **`react-collapsible@2.10.0`** (both declare React ≤18 peers) were flagged "verify at runtime, bump if it breaks" (S2) — not yet confirmed.
   - Only login + masked inputs + reachability of the 3 target routes were browser-verified; a **broader route-by-route runtime pass** is still needed.

7. **PoC housekeeping.**
   - **S8 teardown skipped on purpose** (worktree kept, branch pushed to preserve evidence) — clean up when the real migration lands.
   - **Jira comment `37197`** on MLID-364 still needs updating to record that both blockers are resolved and the branch is pushed.
