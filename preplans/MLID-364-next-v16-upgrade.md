# MLID-364 — Upgrade Next.js 14 → 16 (Pre-Plan)

> This is a **pre-plan**, not the implementation plan. It is the briefing that the real plan (built later via `/plan-task MLID-364`) and any resulting work will be based on. The Jira description is a starting point, not the sole source of truth.

**Ticket:** [MLID-364](https://localinfusion.atlassian.net/browse/MLID-364) — Story — Priority High (P2) — Status TO DO. Summary: "P2: Complete upgrade nextjs to v16." Description (verbatim): "For security compliance." There are no acceptance criteria; the real specification is the Next.js upgrade guide plus what our own codebase actually touches.

---

## 1. High-Level Summary

We need to upgrade the web application (`apps/web`) from **Next.js 14 to Next.js 16**, for security compliance. This also requires moving from **React 18 to React 19**, because Next.js 16 mandates it.

This is a framework upgrade with **no intended change to product behavior**. The end user should see the same application. The work is almost entirely internal: bumping dependencies, adapting our code to a small number of breaking API changes, and confirming our third-party libraries still work on React 19.

The largest single piece of mechanical work is that Next.js 16 makes route parameters **asynchronous**. About **147 files** must change a few lines each to `await` their parameters. The good news: an official automated tool (a "codemod") performs the bulk of this edit, and **133 of those 147 files are backend API route handlers** with a very simple, repetitive change. The remaining 14 are page/layout files.

The real effort and risk are **not** in those 147 mechanical edits. They are in two places:

1. **Confirming our third-party libraries support React 19** (the biggest unknown — one unmaintained library could force a replacement).
2. **Our custom build configuration**, which Next.js 16 changes the default builder for. We have a low-risk way to keep our current builder for now and treat the builder migration as a separate future task.

**Key recommendation:** we can perform this upgrade **without a team-wide code freeze and without weekend work** if we are allowed to land an intermediate Next.js 15 step on `develop` first (Next.js 15 is backward-compatible, so the wide change becomes non-breaking and can merge gradually). If a single direct 14 → 16 jump is mandated instead, the fallback is a **short, scoped code freeze** (a few hours, limited to route files), **not** weekend work — see section 3.4.

**Decision needed:** is an intermediate Next.js 15 allowed on `develop`, or is a single 14 → 16 jump required? That one answer determines whether any freeze is needed at all.

---

## 2. High-Level Effort Estimation

| Dimension | Estimate |
|---|---|
| **Story points** | **5–8 SP** |
| **Hands-on time** | ~3–5 focused working days |
| **Calendar time** | 1 sprint if staged (the gradual, no-freeze path spreads merges across days) |
| **Files affected** | ~**200 total** — ~147 route files (codemod-driven) + ~40–60 test files + ~7 config/package files |
| **Cost driver** | The React 19 dependency audit and the builder decision — **not** the 147 mechanical edits |

### npm modules to update (required)

| Module | Current | Target |
|---|---|---|
| `next` | `14.2.33` | `16.x` |
| `react` | `18` | `19.2` |
| `react-dom` | `18` | `19.2` |
| `@types/react` | `18.3.3` | `19.x` |
| `@types/react-dom` | (add if absent) | `19.x` |
| `eslint-config-next` | `^14.2.5` | `16.x` |
| `@next/bundle-analyzer` | `^16.1.6` | already on 16 ✅ |

### npm modules to audit for React 19 compatibility (may need bump or replacement)

| Module | Current | Concern |
|---|---|---|
| `next-auth` | `4.24.7` | v4 + React 19 peer warnings are common |
| `react-pdf` | `10.1.0` | verify 19 support |
| `react-day-picker` | `9.0.9` | verify 19 support |
| `react-input-mask` | `2.0.4` | likely unmaintained for React 19 — highest replacement risk |
| `react-modal` | `3.16.1` | verify 19 support |
| `react-select` | `5.8.0` | verify 19 support |

Known-fine on React 19: **MUI 7**, **zustand 5**.

---

## 3. Detail Level

### 3.1 Current state vs. target state

| | Now | Next.js 16 needs | Status |
|---|---|---|---|
| `next` | `14.2.33` | `16.x` | upgrade |
| `react` / `react-dom` | `18` | `19.2` | upgrade |
| `@types/react` / `@types/react-dom` | `18.3.3` | `19.x` | upgrade |
| Node | `>=20` | `>=20.9` | ✅ already OK |
| TypeScript | `5.5.3` | `>=5.1` | ✅ already OK |

**The framing that matters — we jump two majors at once.** We are on 14, not 15. A direct 14 → 16 jump absorbs *both* sets of breaking changes in one move: the v15 changes (async request APIs introduced; `fetch` and GET route handlers no longer cached by default) **and** the v16 changes (synchronous access fully removed; Turbopack default; `next lint` removed; `middleware` → `proxy`). On 14 our `params` / `searchParams` are still plain synchronous objects, so a direct jump takes the entire async migration as one breaking change. Routing through 15 first avoids this (section 3.4).

### 3.2 How many files need to change

Measured against the real `apps/web` source. The async-params change is the bulk; it lands almost entirely in **backend API route handlers**.

**By file type:**

| Category | Count | Nature |
|---|---|---|
| App-router files using `params` / `searchParams` | **147** | Mechanical — must `await params`. Codemod-assisted. |
| → of those: **API `route.ts` handlers** | **133** | Simple `(req, { params })` → `await params`. Low risk, repetitive. |
| → of those: **page / layout files** | **14** | `{ params, searchParams }` → async. Slightly more care. |
| Co-located `*.test.tsx` passing `params` | ~40–60 (subset) | Test mocks must pass `params` as a `Promise` |
| `next.config.mjs` | 1 | The hard one (builder decision — see 3.3) |
| `middleware.ts` → `proxy.ts` | 1 | File + function rename |
| ESLint config + `package.json` scripts | ~3 | `next lint` removed |
| `package.json` dependencies | 1 | version bumps |

**By directory (the 147 param-using route files):**

| Directory | Count |
|---|---|
| `app/api/admin/**` | 40 |
| `app/api/intakes/**` | 13 |
| `app/api/weekly-provider-reports/**` | 12 |
| `app/api/patient/**` | 9 |
| `app/api/ai-chat-dashboard/**` | 9 |
| `app/api/orders/**` | 8 |
| `app/api/chat/**` | 6 |
| `app/api/content-library/**` | 5 |
| `app/api/clinical-reviews/**` | 5 |
| `app/api/weinfuse/**` | 4 |
| `app/api/hospitals/**` | 4 |
| `app/messages/**` (pages) | 4 |
| `app/api/notifications/**` | 3 |
| `app/api/insurance-payors/**` | 3 |
| `app/intakes/**` (pages) | 3 |
| `app/api/documents/**` | 2 |
| `app/admin/**` (pages) | 2 |
| Long tail (15 dirs × 1 file each: `api/v2`, `api/provider-offices`, `api/provider-mappings`, `api/patients`, `api/locations`, `api/jobs`, `api/insurance-plans`, `api/feature-flags`, `api/auditor`, `api/appointments`, `insurance-plans`, `insurance-payors`, `hospital-system-agreements`, `drug-admin-panel`, `appointments`) | 15 |
| **Total** | **147** |

**What shrinks the job (good news):**

- **`next/headers` — zero usages.** No `cookies()`, `headers()`, or `draftMode()` anywhere in the app. That is normally the nastiest async-API surface; we dodge it entirely.
- **Client hooks are not affected.** The 468 hits of `useRouter` / `useSearchParams` / `usePathname` across 163 files are **client-side** hooks. The async-params breaking change does **not** touch them.
- **133 of 147 are API handlers** — the simplest, most repetitive variant of the change.

### 3.3 Real risks (where time actually goes — not file count)

**Risk 1 — React 18 → 19 peer-dependency audit (biggest unknown).** ~30 third-party UI deps; six need verification (table in section 2). A single incompatible, unmaintained dependency (notably `react-input-mask`) can force a replacement and widen scope. This is the item most likely to change the estimate.

**Risk 2 — Turbopack becomes the default builder, and our `next.config.mjs` has a large custom webpack config.** The config contains `IgnorePlugin` entries (`ssh2`, `applicationinsights`, `li-public-api`), a `node:`-scheme `NormalModuleReplacementPlugin`, server/client `externals`, and `resolve.fallback`. In v16, `next build` with a webpack config **fails by design**.
- **Mitigation (recommended):** add the `--webpack` flag to the build script. Low effort, low risk; decouples the framework upgrade from the bundler migration.
- **Follow-up (separate ticket):** port the webpack concerns to Turbopack (`turbopack.resolveAlias`, etc.). Should not block MLID-364.

**Risk 3 — `next lint` is removed.** Our `lint` / `lint:fix` scripts and `next build --no-lint` all rely on it (v16 removes it; `next build` no longer lints). Migrate to the ESLint CLI; `@next/eslint-plugin-next` now defaults to **flat config**. Codemod available: `next-lint-to-eslint-cli`.

**Risk 4 — caching-semantics drift (v15 behavior).** `fetch` and GET route handlers are no longer cached by default. Confirm nothing silently relied on the old default-on caching before assuming parity.

**Risk 5 — merge conflict vs. team velocity.** Multiple PRs merge to `develop` daily. The 147-file change is exactly the kind of wide diff that rots against a fast-moving base. Section 3.4 is the mitigation.

**Smaller config cleanups in `next.config.mjs` (low risk):**

| Item | Action |
|---|---|
| `swcMinify: false` | Remove — no longer a valid option |
| `experimental.instrumentationHook: true` | Remove — now stable (we already have `instrumentation.ts`) |
| `experimental.typedRoutes: true` | Move to top-level `typedRoutes` |
| `images.domains` | Deprecated → fold into `remotePatterns` (already declared) |
| `build` script `--no-lint` | Re-evaluate — build no longer lints |
| `middleware.ts` → `proxy.ts` | Rename file + exported function; our middleware is already Node-runtime, so `proxy` fits |

### 3.4 Suggested strategy of implementation

The core problem is that, in a direct 14 → 16 jump, the 147-file async-params change is **breaking** — the moment it lands, every un-migrated route fails. That is what makes people reach for a freeze or weekend work. The strategy below removes that pressure.

#### Primary strategy (recommended): stage 14 → 15 → 16

Next.js **15 keeps synchronous-param compatibility**. If we bridge through 15 first, migrating a route to `await params` is **backward-compatible** — it works on 15 either way. The wide change stops being breaking and can merge **incrementally, in small PRs, on normal workdays**, with no freeze and no weekend.

| Stage | What lands | Breaking? | Team impact | How it merges |
|---|---|---|---|---|
| **A — bridge bump** | Next **15** + React 19, config cleanups, lint migration, React-19 dependency audit fixes | No (15 keeps sync compat) | None | Normal PR, any day. Surfaces React 19 problems early, while sync still works. |
| **B — async sweep** | Codemod the ~147 routes (+ test mocks) to `await params` | **No** — async params already work on 15 | None | **Split by directory** into several PRs, each merges independently on its own day. No freeze, no weekend. |
| **C — flip to 16** | Bump 15 → 16, `middleware` → `proxy`, `--webpack` flag, drop sync-compat, `revalidateTag` second arg | Yes — but nothing is left to break (all params already async) | One short window | Small diff. Merges in a normal low-traffic window. |

**Safeguard for stage C:** routes added by teammates during the bridge window will use sync params (they work fine on 15). Immediately before the flip, **re-run the async-params codemod on the latest `develop`** to catch stragglers. Treat the migration as *regenerable*, not as a precious branch to defend.

#### Fallback strategy (only if a single direct 14 → 16 jump is mandated)

If staging through 15 is not allowed, the version bump and the 147-file sweep must land together as one breaking merge. In that case, the choice is between a code freeze and weekend work. **Recommendation: a scoped, short code freeze — not the weekend.**

| | Scoped code freeze (recommended fallback) | Weekend work (not recommended) |
|---|---|---|
| What it is | A few-hour window where nobody merges PRs that **add or rename a dynamic route param** (not a full team stop) | Run the whole upgrade solo, off-hours |
| Conflict handling | Re-run the codemod on latest `develop` right before merge, then merge in the quiet window | Same, but alone |
| Main risk | Small, bounded — team loses a little merge flexibility for a few hours | The **React 19 unknowns** (Risk 1): if the build breaks, you debug alone and the whole team is blocked the next workday |

The weekend's problem is not the merge mechanics (Monday's open PRs simply rebase onto a migrated `develop`). It is that the React 19 audit is the highest-uncertainty part of the work, and a weekend concentrates that uncertainty into the least-supported window. A freeze keeps the work in business hours where people can help.

**Bottom line:** (1) best — stage 14 → 15 → 16, no freeze, no weekend; (2) if a direct jump is mandated — scoped short freeze; (3) avoid — the weekend.

### 3.5 Extra info (what we already have)

**Codemods and commands to lean on** (npm/npx only — project policy):

```bash
# Whole-upgrade driver (updates deps, config, lint, middleware→proxy, strips unstable_ prefixes)
npx @next/codemod@canary upgrade latest

# Async params/searchParams migration across the app (next-async-request-api transform)

# next lint → ESLint CLI
npx @next/codemod@canary next-lint-to-eslint-cli .

# Generate type helpers for async params (PageProps / LayoutProps / RouteContext)
npx next typegen
```

Type-safe async-params pattern:

```tsx
export default async function Page(props: PageProps<'/blog/[slug]'>) {
  const { slug } = await props.params;
  const query = await props.searchParams;
}
```

**Relevant entry points (for planning):**

- `apps/web/package.json` — dependency versions + `dev` / `build` / `lint` / `lint:fix` scripts.
- `apps/web/next.config.mjs` — webpack block (builder decision), `swcMinify`, `experimental.instrumentationHook`, `experimental.typedRoutes`, `images.domains`.
- `apps/web/middleware.ts` — rename to `proxy.ts`, rename the exported function.
- `apps/web/instrumentation.ts` — already present; confirms `instrumentationHook` flag can be dropped.
- The 147 dynamic route files under `apps/web/app/**/{page,layout,route}.{ts,tsx}` — the async-params sweep target (directory breakdown in 3.2).
- ESLint config (flat-config migration) + `@repo/eslint-config` interplay.

**Open questions to resolve before the real plan:**

1. **Staged or direct?** Is an intermediate Next.js 15 allowed on `develop` (enables the no-freeze path), or is a single 14 → 16 jump required (triggers the scoped-freeze fallback)? — the key decision.
2. **Turbopack now or later?** Recommend `--webpack` for MLID-364 and a separate follow-up ticket for full Turbopack adoption.
3. **React 19 audit outcome** — does any dependency (notably `react-input-mask`, `next-auth@4`) force a replacement that widens scope?
4. **Deadline?** The P2 + "security compliance" framing suggests this may be time-boxed; confirm the target sprint.

---

**Next step:** once the open questions in section 3.5 are settled, run `/plan-task MLID-364` using this pre-plan as context to produce the implementation plan.
