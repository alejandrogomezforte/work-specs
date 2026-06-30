# MLID-364 — Next.js v16 Upgrade: Proof of Concept Plan

> This is a **throwaway proof of concept (PoC)**, not the implementation plan. Its only goal is to learn what actually breaks when we upgrade, on a temp branch we will delete afterward. The findings feed back into the real plan (built later via `/plan-task MLID-364`). Companion document: [`MLID-364-next-v16-upgrade.md`](./MLID-364-next-v16-upgrade.md) (the pre-plan).

---

## Goal

Prove whether we can reach Next.js 16 + React 19 at all, by running the **hardest path first**: a single direct 14 → 16 jump on a temp branch off `develop`.

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

Then work entirely inside the new folder (open the Claude Code session there, pointed at this PoC doc):

```bash
cd ../li-call-processor-nextjs-v16-poc
npm install          # this worktree gets its OWN node_modules (the whole point)
# ... run the PoC sequence below ...
```

**Mid-PoC hotfix flow:** the main folder stays on `develop` with its React 18 `node_modules` completely untouched. When QA needs a hotfix, work it in the main folder immediately — no reinstall, no branch juggling. The PoC worktree keeps its Next 16 state in parallel; `cd` back to it when the hotfix is done.

**Visibility of this doc:** `docs/agomez/` is excluded from the main repo (it is versioned in a separate nested git repo), so the new worktree will **not** contain this file — and that is exactly what we want. The status tracker lives in **one place only**: the main folder's copy at `docs\agomez\preplans\MLID-364-nextjs-poc.md`. Reference it from the worktree session by that absolute path; never commit it onto the spike branch. All status-table updates below happen in the main folder.

**Teardown** (after findings are recorded back into this file):

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
7. **Write findings back into this file** (the "PoC Results" section below), then delete the branch.

---

## Decision — PoC depth: **Option B (run the web app)**

**Chosen: Option B — the PoC is successful only when the web app runs and renders pages with no errors.**

Success bar (all must hold):

- `npm run dev` boots the web app on Next 16 / React 19 with no startup errors.
- The following routes render with no console or runtime errors when navigated in the browser:
  - `/intakes`
  - `/admin`
  - `/orders-tracker`
- These three are the minimum set; more routes can be checked opportunistically, but these must pass.

Verification is **manual in the browser, done by the user** (the user runs the dev server and navigates). This depth goes past build/typecheck on purpose, because runtime-only React 19 issues (hydration, third-party UI components) will not show up at build time.

Options considered and not chosen:

| Option | What it means | Why not chosen |
|---|---|---|
| **A — Build + typecheck only** | Stop once `npm run build` and `types:check` pass or fail with a catalogued list. | Misses runtime-only React 19 issues; too shallow to trust. |
| **C — Just the dependency audit** | Only resolve the React 19 peer-dep question; skip the 147-file sweep. | Narrowest; does not prove the app actually runs. |

---

## Staged execution & status tracking

The PoC runs as **8 small, resumable stages** (S0–S7), a grouping of the 7-step sequence above into stop/start checkpoints. The design goals: every stage is achievable in a short sitting, and every stage ends at a **persisted checkpoint** so an interruption never loses progress.

**What makes it resumable:**

- The **worktree folder and its `node_modules` persist on disk** between sessions — we do not tear it down until S7. Resuming is just `cd` back into the worktree.
- Each stage (S1–S6) ends at a **git commit on the spike branch**, so the code + `package.json` + lockfile state of each checkpoint is captured. (`node_modules` is not committed, but it stays on disk in the worktree.)
- This status table is the **single source of truth for "where we stopped."** It lives only in the main folder (see Isolation strategy) and is updated at the end of every stage.

### Resume pointer

> **Next stage to run: S0** &nbsp;|&nbsp; **Last updated: (not started)** &nbsp;|&nbsp; **Spike branch HEAD: (none yet)**
>
> _Update this line at the end of each working session so the next pickup is instant._

### Status legend

⬜ Not started &nbsp; 🟡 In progress &nbsp; ✅ Done &nbsp; ⛔ Blocked

### Stages

| Stage | Actions | Resumable checkpoint | Status |
|---|---|---|:--:|
| **S0 — Setup & isolation** | Pull `develop`; `git worktree add` the spike branch (Isolation strategy); `cd` in; `npm install` to get a **Next 14 baseline**; confirm the app runs and the 3 target routes render on 14 (the "before" reference). | Worktree exists with a working Next-14 `node_modules` on disk. No commit needed (branch == `develop`). | ⬜ |
| **S1 — Core upgrade codemod** | Run `npx @next/codemod@canary upgrade latest` (bumps `next`→16, `react`/`react-dom`→19, `@types/*`, `eslint-config-next`; config cleanups; `middleware`→`proxy`; lint migration; installs). | Commit on spike branch: "S1 core upgrade". First view of React 19 peer-dep fallout. | ⬜ |
| **S2 — React 19 dependency audit** | Resolve peer-dep issues surfaced by S1 (e.g. `react-input-mask`, `next-auth@4`, `react-pdf`, `react-day-picker`, `react-modal`, `react-select`); bump or replace as needed until install is clean. | Commit: "S2 react19 deps". `npm install` runs with no blocking peer errors. | ⬜ |
| **S3 — Async params sweep** | Run the `next-async-request-api` codemod across the ~147 dynamic route files (+ test mocks). | Commit: "S3 async params". | ⬜ |
| **S4 — Builder + lint config** | Add `--webpack` to the build script (keep webpack, defer Turbopack); finish ESLint flat-config migration if the codemod left anything. | Commit: "S4 build+lint config". | ⬜ |
| **S5 — Build & typecheck green** | Iterate `npm run build` and `npm run types:check`; fix compile/type errors until they pass or are clearly catalogued. | Commit: "S5 build green" (or a catalogued failure list in PoC Results). | ⬜ |
| **S6 — Run app + browser verify (Option B)** | User runs `npm run dev`; navigate `/intakes`, `/admin`, `/orders-tracker`; confirm each renders with no console/runtime errors; fix runtime-only React 19 issues. | Commit: "S6 runtime fixes" (if any). **This is the Option B success bar.** | ⬜ |
| **S7 — Findings & teardown** | Write the PoC Results section (in the main folder); decide verdict; `git worktree remove` + `git branch -D` the spike branch. | PoC Results filled in; worktree removed; branch deleted. | ⬜ |

### Per-stage notes (free-form, written when stopping mid-stage)

- **S0:** —
- **S1:** —
- **S2:** —
- **S3:** —
- **S4:** —
- **S5:** —
- **S6:** —
- **S7:** —

### How to resume next session

1. Open this file in the **main folder** and read the **Resume pointer** + status table.
2. `cd` into `../li-call-processor-nextjs-v16-poc` (the worktree persists; its `node_modules` is intact).
3. Continue from the first stage that is not ✅. Read its "Per-stage notes" if it was left 🟡.

---

## Authorization status

- npm install / codemod-driven dependency changes: **not yet authorized.** Nothing will be installed or executed until explicitly approved.

---

## PoC Results

_To be filled in after the PoC runs._

- React 19 dependency audit outcome:
- Builder (`--webpack`) outcome:
- Build / typecheck failures catalogued:
- Runtime / browser verification (Option B):
  - `/intakes` renders with no errors:
  - `/admin` renders with no errors:
  - `/orders-tracker` renders with no errors:
- Verdict (is Next 16 reachable, and at what cost):
