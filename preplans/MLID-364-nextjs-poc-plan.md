# MLID-364 — Next.js v16 Upgrade: Proof of Concept Plan

> This is a **throwaway proof of concept (PoC)**, not the implementation plan. Its only goal is to learn what actually breaks when we upgrade, on a temp branch we will delete afterward. The findings feed back into the real plan (built later via `/plan-task MLID-364`).
>
> **Companion documents:**
> - [`MLID-364-next-v16-upgrade.md`](./MLID-364-next-v16-upgrade.md) — the pre-plan.
> - [`MLID-364-nextjs-poc-implementation-history.md`](./MLID-364-nextjs-poc-implementation-history.md) — the running log: live status, every step taken, errors found and how they were solved, and the final results/verdict. **This plan is the stable design; the history file is the source of truth for "where we are."**

---

## Goal

Prove we can **fully and cleanly migrate** to Next.js 16 + React 19 — a complete, error-free migration, not a reachability catalogue. The deliverable is a working app that runs **cleanly in both modes**: `npm run dev` and `npm run build` must each complete with **no errors**. "Migrate" means bringing everything current on the new version (fix the code, don't just prove it compiles). This is done on a temp branch off `develop`, taking the **hardest path first**: a single direct 14 → 16 jump. The point is to show the team the migration is real and viable.

> **Objective clarified 2026-07-01 (user):** the earlier framing ("prove reachability, catalogue what breaks") was too weak. The real bar is a **clean, error-free app on Next 16 in both dev and prod**. Every error found (type, build, runtime, config) is to be **fixed**, not catalogued and deferred. See the implementation-history log for the objective-change note.

## Why the direct jump for the PoC (even though staged is recommended for real work)

- **The direct jump is the worst case.** If a throwaway PoC of the direct jump builds and runs, the staged 14 → 15 → 16 path is strictly easier from there. A successful direct PoC de-risks both strategies at once.
- **It surfaces the two real unknowns immediately and together:**
  1. React 19 dependency audit (Risk 1 in the pre-plan) — whether our third-party UI libraries hold up.
  2. The webpack → Turbopack builder change (Risk 2) — whether `next build` breaks.
- **A temp branch makes breakage free.** Falling over is the information we want, not a failure.

## Important scope boundary

A successful direct-jump PoC proves the **end state (Next 16) is reachable**. It does **not** by itself prove we must stage the rollout — staging is a team-velocity / merge-conflict argument, not a technical one. So:

- The PoC answers: "Can we get to 16 at all, and what breaks?"
- The staging recommendation stays a **separate decision** about how to land it without freezing the team.

## Method note

This is **spike / exploratory work**, so TDD is intentionally skipped here (the documented exception). The goal is "what breaks and how bad," not shippable code. Nothing from this branch gets committed to `develop` or pushed.

---

## Isolation strategy — run the PoC in a separate git worktree

The upgrade rewrites `package.json`, `package-lock.json`, and the whole `node_modules` tree (Next 16 + React 19). Git only tracks the first two; `node_modules` lives in the working folder and is **not** tracked. So switching branches in a single folder would leave the wrong `node_modules` on disk and force a full `npm ci` every time we bounce between the PoC and other work. The requirement is therefore **two different `node_modules` trees existing side by side at the same time** — which is exactly what a git worktree provides.

**Layout:**

| Folder | Branch | `node_modules` state | Purpose |
|---|---|---|---|
| `C:\Users\alejandro.gomez\Dev\li-call-processor` (main) | `develop` | Next 14 / React 18 (untouched) | Day-to-day work and any QA hotfix |
| `C:\Users\alejandro.gomez\Dev\li-call-processor-nextjs-v16-poc` (worktree) | `spike/MLID-364-next16-direct-poc` | Next 16 / React 19 | The PoC, isolated |

Both folders share the same `.git` object store, so there is no re-clone and no re-fetch — only one extra `node_modules` on disk.

**Create the worktree** (run from the main folder, on Git Bash):

```bash
# Make sure develop is current first
git checkout develop
git pull

# Create the sibling worktree + the PoC branch off develop in one step
git worktree add ../li-call-processor-nextjs-v16-poc -b spike/MLID-364-next16-direct-poc develop
```

Then work entirely inside the new folder (open the Claude Code session there, pointed at these PoC docs):

```bash
cd ../li-call-processor-nextjs-v16-poc
npm install          # this worktree gets its OWN node_modules (the whole point)
# ... run the PoC sequence below ...
```

**Mid-PoC hotfix flow:** the main folder stays on `develop` with its React 18 `node_modules` completely untouched. When QA needs a hotfix, work it in the main folder immediately — no reinstall, no branch juggling. The PoC worktree keeps its Next 16 state in parallel; `cd` back to it when the hotfix is done.

**Visibility of these docs:** `docs/agomez/` is excluded from the main repo (it is versioned in a separate nested git repo), so the new worktree will **not** contain these files — and that is exactly what we want. The plan and the status/history tracker live in **one place only**: the main folder's copies under `docs\agomez\preplans\`. Reference them from the worktree session by their absolute paths; never commit them onto the spike branch. All status and history updates happen in the main folder's implementation-history file.

**Teardown** (after findings are recorded into the implementation-history file):

```bash
# From the main folder
git worktree remove ../li-call-processor-nextjs-v16-poc   # add --force if the tree is dirty
git branch -D spike/MLID-364-next16-direct-poc            # throwaway branch, nothing to keep
```

---

## Proposed PoC sequence

Runs inside the worktree folder `li-call-processor-nextjs-v16-poc` on branch `spike/MLID-364-next16-direct-poc` (see Isolation strategy above).

1. **Create the worktree off a fresh local `develop`** (the `git worktree add` step above), then `cd` into it.
2. **Run the whole-upgrade codemod:** `npx @next/codemod@canary upgrade latest`. Bumps `next` → 16, `react` / `react-dom` → 19, applies config cleanups, renames `middleware` → `proxy`, migrates lint.
3. **Run the async-params codemod** across the ~147 dynamic route files (`next-async-request-api` transform).
4. **Add `--webpack` to the build script** — keep our existing webpack config for now, defer the Turbopack migration to a separate future ticket.
5. **Attempt `npm run build` and `npm run types:check`**, and catalog every failure — especially React 19 peer-dependency fallout.
6. **Run the app and verify in the browser (the Option B success bar):** the user starts `npm run dev` and navigates `/intakes`, `/admin`, and `/orders-tracker`, confirming each renders with no console or runtime errors. Catalog any runtime-only React 19 issues that build/typecheck did not catch.
7. **Write findings into the implementation-history file** (the "PoC Results" section there), then delete the branch.

---

## Decision — PoC depth: **full clean migration (dev + prod both green)**

**The PoC is successful only when the app is a clean, error-free Next 16 / React 19 migration in BOTH modes.** All of the following must hold — every error is **fixed**, not catalogued:

1. **`npm run build` completes with no errors** — webpack build + full typecheck green (all React 19 type errors fixed, `next.config.mjs` cleaned of removed/renamed keys). Production mode works.
2. **`npm run dev` runs clean** — boots on Next 16 / React 19 with no startup, console, or runtime errors, including runtime-only React 19 breaks fixed (e.g. `react-input-mask`/`findDOMNode`).
3. **The following routes render with no console or runtime errors** when navigated in the browser: `/intakes`, `/admin`, `/orders-tracker` (minimum set; more opportunistically).
4. **"Everything current"** — deferred cleanups are brought current as part of a real migration: `next.config.mjs` key updates, the ESLint flat-config / eslint 9 finish, and the async-params test mocks so the suite is consistent. (dev + build green in 1–3 are the hard gates; these keep the migration honest.)

Browser verification (item 3) is **manual, done by the user**. Build/dev cleanliness (items 1–2) is verified by running the commands.

**Rejected shallower depths** (they only *catalogue* failures — insufficient for the "prove we can migrate" goal): build+typecheck-only stopping at a catalogued list; dependency-audit-only; or "runs in dev but production build stays red." The whole point is a migration the team can trust, so production build must be green too.

---

## Known blockers to success — BOTH RESOLVED ✅ (2026-07-02)

The build side is green (`npm run build` passes). The **runtime** side was gated by **two third-party library upgrades** (user-owned decisions — the assistant researched and got approval before swapping). **Both are now resolved and runtime-verified in the browser.** Detailed diagnosis, evidence, and the resolution live in the implementation-history log (S7a / S7b); summary here:

1. **NextAuth v4 → Auth.js v5 — ✅ RESOLVED (S7a, commit `41e54b08b`).** `next-auth@4` read Next 16's now-async `cookies()` / route `params` synchronously, breaking all authentication. Fixed by migrating to Auth.js v5 (`next-auth@5.0.0-beta.31`, pinned — v5 is beta-only): root-level `auth.ts`, `auth()` in place of `getServerSession`, restructured `[...nextauth]` handler, changed session cookie name. **Actual blast radius: ~336 files** (larger than the ~166 estimate). Google login verified end-to-end.
2. **`react-input-mask` → `@mona-health/react-input-mask` — ✅ RESOLVED (S7b, commit `d2cff3bf4`).** The original used `findDOMNode` (removed in React 19), crashing masked/phone inputs. Replaced with the maintained fork (React 19 compatible). Not a pure drop-in — the fork changed the `children` (function → element) and ref (`inputRef` → `ref`) APIs. Masked/phone inputs verified rendering.

**Success bar met:** `npm run build` GREEN **and** `npm run dev` runs clean with login + masked inputs working in the browser. Full verdict in the implementation-history **PoC Results** section.

**Tracked in Jira:** MLID-364 comment `37197` documented both blockers for the team (should be updated to note both are now resolved with the branch pushed).

---

## Staged execution — stage definitions

The PoC runs as a set of **small, resumable stages** (S0 through S8, including S6b/S6c/S7a/S7b), a grouping of the sequence above into stop/start checkpoints. (Extra stages were added as the PoC surfaced blockers — see the implementation-history log.) The design goals: every stage is achievable in a short sitting, and every stage ends at a **persisted checkpoint** so an interruption never loses progress.

> The **live status, per-stage notes, and error log** live in the implementation-history file, not here. This section is the stable definition of what each stage entails.

**What makes it resumable:**

- The **worktree folder and its `node_modules` persist on disk** between sessions — we do not tear it down until S8. Resuming is just `cd` back into the worktree.
- Each stage (S1–S6) ends at a **git commit on the spike branch**, so the code + `package.json` + lockfile state of each checkpoint is captured. (`node_modules` is not committed, but it stays on disk in the worktree.)

### Stages

| Stage | Actions | Resumable checkpoint |
|---|---|---|
| **S0 — Setup & isolation** | Pull `develop`; `git worktree add` the spike branch (Isolation strategy); `cd` in; `npm install` to get a **Next 14 baseline**; confirm the app runs and the 3 target routes render on 14 (the "before" reference). | Worktree exists with a working Next-14 `node_modules` on disk. No commit needed (branch == `develop`). |
| **S1 — Core upgrade codemod** | Run `npx @next/codemod@canary upgrade latest` (bumps `next`→16, `react`/`react-dom`→19, `@types/*`, `eslint-config-next`; config cleanups; `middleware`→`proxy`; lint migration; installs). | Commit on spike branch: "S1 core upgrade". First view of React 19 peer-dep fallout. |
| **S2 — React 19 dependency audit** | Resolve peer-dep issues surfaced by S1 (e.g. `react-input-mask`, `next-auth@4`, `react-pdf`, `react-day-picker`, `react-modal`, `react-select`); bump or replace as needed until install is clean. | Commit: "S2 react19 deps". `npm install` runs with no blocking peer errors. |
| **S3 — Async params sweep** | Run the `next-async-request-api` codemod across the ~147 dynamic route files (+ test mocks). | Commit: "S3 async params". |
| **S4 — Builder + lint config** | Add `--webpack` to the build script (keep webpack, defer Turbopack); finish ESLint flat-config migration if the codemod left anything. | Commit: "S4 build+lint config". |
| **S5 — Build GREEN (`npm run build` clean)** | Fix **all** React 19 / Next 16 build + type errors until `npm run build` completes with **no errors** (no "catalogue and defer"). Includes cleaning `next.config.mjs` of removed/renamed keys (`swcMinify`, `experimental.instrumentationHook`, `experimental.typedRoutes` → `typedRoutes`, `images.domains` → `images.remotePatterns`). | `npm run build` exits 0, fully green. |
| **S6 — Dev GREEN + browser verify (`npm run dev` clean)** | Fix runtime-only React 19 breaks so `npm run dev` runs clean — including replacing `react-input-mask` (`findDOMNode` removed in React 19). User navigates `/intakes`, `/admin`, `/orders-tracker`; each renders with no console/runtime errors. | `npm run dev` boots clean; the 3 routes render error-free. **Both dev and prod now green = migration proven.** |
| **S6b — Test-suite consistency** | Update the async-params **page** tests (`[id]/page.test.tsx`) for React 19 `use()` (stable `Promise.resolve` params + `act()`-wrapped async render), and the route/test mocks, so `npm test` is consistent with the new signatures. (Note: suites that render masked inputs stay red until Blocker #1 / react-input-mask is resolved.) | `npm test` green except the blocker-gated suites. |
| **S6c — Repo-wide ESLint 8→9 + flat-config migration** | Move the **whole monorepo** off ESLint 8 legacy `.eslintrc` to **ESLint 9 flat config** (required by Next 16 — we are not staying on eslint 8): convert `@repo/eslint-config` (`base.js`/`nextjs.js`) to flat config; bump `@typescript-eslint` 7→8 and `eslint` 8→9 at the root and every app; verify the plugins under eslint 9 (`eslint-plugin-rulesdir` + the custom `no-direct-drizzle-write` rule, `react`, `react-hooks`, `prettier`, `eslint-config-next` 16); remove the dead `.eslintrc.json` files; fix the generated web flat config (`sourceType` `script`→`module`, etc.); make `npm run lint` pass across `web`, `api`, `mcp`, `kiosk`. | `npm run lint` (turbo, all workspaces) green on ESLint 9 flat config. |
| **S7a — Resolve Blocker #2: NextAuth v4 → Auth.js v5** | Migrate `next-auth@4` → Auth.js v5 (`next-auth@5`) so authentication works on Next 16 (async `cookies()`/`params`): root-level `auth.ts` config (Google + Credentials providers, MongoDB adapter, jwt session + the existing `signIn`/`session` callbacks), restructure the `[...nextauth]` route, replace `getServerSession(authOptions)` with `auth()` across ~166 files, update the `proxy.ts` (middleware) auth checks. Verify end-to-end login + role-gated menu in dev. **User-owned: confirm the v5 approach before starting.** | `npm run dev` login works end-to-end; `getServerSession` fully replaced; auth flow green. |
| **S7b — Resolve Blocker #1: replace react-input-mask** | After the user reviews usage and picks a replacement (candidates researched in the history log: `@mona-health/react-input-mask` near-drop-in, `react-imask`, `@react-input/mask`), rewrite `MaskedInput.tsx` + `MuiPhoneField.tsx` (and any other render sites) onto it; preserve masking behavior (phone mask, etc.). Verify masked/phone inputs render and behave at runtime; unblock the affected test suites. **User-owned: confirm the library before starting.** | Masked/phone inputs render with no `findDOMNode` crash; affected suites pass. |
| **S8 — Findings & teardown** | Write the PoC Results section (in the implementation-history file); state the verdict (migration is viable, with the concrete change-set + resolved blockers as evidence); `git worktree remove` + `git branch -D` the spike branch. | PoC Results filled in; worktree removed; branch deleted. |

### How to resume next session

1. Open the **implementation-history file** in the **main folder** and read the **Resume pointer** + status table.
2. `cd` into `../li-call-processor-nextjs-v16-poc` (the worktree persists; its `node_modules` is intact).
3. Continue from the first stage that is not ✅. Read its per-stage log entry if it was left 🟡.

---

## Authorization policy

- npm install / codemod-driven dependency changes are **gated**: nothing is installed or executed until explicitly approved for that step. Each granted authorization is logged in the implementation-history file.
