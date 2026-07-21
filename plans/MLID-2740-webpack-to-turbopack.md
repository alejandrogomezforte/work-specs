# MLID-2740 — Migrate LISA web bundler from webpack to Turbopack

## Task Reference

- **Jira**: [MLID-2740](https://localinfusion.atlassian.net/browse/MLID-2740)
- **Story Points**: 5
- **Branch**: `feature/MLID-2740-webpack-to-turbopack`
- **Base Branch**: `develop`
- **Parent**: MLID-983 (umbrella epic "Dev Tech Enhancements") — standalone task, no epic plan-progress file
- **Status**: In QA — code merged to `develop` (2026-07-16), not yet complete. The ticket is going through the QA test series and is not "done" until QA clears it. Included an inherited-from-develop fix (`217f337f2`: export `HourglassEmptyIcon` from `@repo/ui` barrel) to unblock CI's `types:check`. Dev + build + client-bundle verified; developer staging check (2026-07-17) passed with no errors. QA owns the remaining verification and moves the Jira ticket.
- **Setup**: implemented in a **git worktree** off `develop` (same procedure as the Next.js 16 upgrade), not in the main working tree

---

## Session Progress Log

### As of 2026-07-15 (end of day) — resume here 2026-07-16

**Worktree:** `C:/Users/alejandro.gomez/Dev/worktrees/MLID-2740` on branch `feature/MLID-2740-webpack-to-turbopack` (off `develop`). Nothing pushed yet (local only).

**Worktree setup gotcha (for resuming):** a fresh worktree has NO `node_modules` and NO `apps/web/.env.local` (both gitignored). Already handled this session: ran `npm ci` in the worktree, and copied `.env.local` from the main checkout (`cp .../li-call-processor/apps/web/.env.local .../worktrees/MLID-2740/apps/web/.env.local`). If the worktree is recreated, redo both.

**Commits on the branch (4, in order):**
1. `c4eb78574` build(web): drop `--webpack` flag from dev and build scripts
2. `b254b93fc` build(web): migrate next.config from webpack to Turbopack (delete webpack block; add `serverExternalPackages` + `turbopack.resolveAlias`; remove `esmExternals`; add `turbopack-empty-module.ts`)
3. `d53f3c3f0` fix(web): replace webpack-tolerated `:global(> span)` CSS selectors
4. `bd1c0bc73` fix(web): add MUI `AppRouterCacheProvider` to fix Turbopack emotion hydration (adds `@mui/material-nextjs@^7.3.10`)

**Verified GREEN this session:**
- `npm run build:web` (Turbopack) — exit 0, full route table generated.
- `npm run types:check` — pass (only after workspace packages `@repo/db`/`@repo/ui` are built; a bare `tsc` in a fresh worktree fails on those until `build:web` runs once).
- `npm run test` (Jest) — 1366/1367 suites, 19,890 tests pass. The 1 failure (`rxPreferredClaims.parity.test.ts`) is a **Windows-only** embedded-Postgres temp-dir `EBUSY` teardown race, unrelated to the bundler; would not occur on Linux CI.
- Client-bundle greps against `.next/static/chunks` — no `ssh2` / `ssh2-sftp-client` / `applicationinsights` / `node:` leakage.
- **Hydration error (Notifications.tsx) — user-confirmed GONE in dev** after the `AppRouterCacheProvider` fix.
- Google/next-auth login — works in dev (after the `.env.local` copy; the earlier login failures were missing env, not the migration).

**OPEN — pick up here tomorrow:**
1. ~~`AbortError` at `useIntakesTable.ts:718`~~ **RESOLVED — pre-existing, out of scope (2026-07-16).** User reproduced the SAME error on `develop` (Webpack badge confirmed), so it is NOT caused by the Turbopack migration. Mechanism: on soft navigation (home → menu → `/intakes`), the effect's `buildParams` dep changes when the debounced values (`debouncedSearch`, `debouncedFilterModel`) settle just after mount → effect re-runs → cleanup `controller.abort()` cancels the first in-flight fetch → `AbortError`. It is intentional ("cancel stale requests on cleanup"), already swallowed in `loadData` (line ~530), and only surfaces as React 19 dev-overlay noise (hard refresh does not trigger it). No action for this ticket. Logged separately as **MLID-2837** (Task under MLID-983) with root cause + proposed fix: the real source is `fetchIntakes` in `apps/web/services/intakes/intakeService.ts` logging the abort via `logger.error` before re-throwing, one layer below `loadData` which already ignores it.
2. **Continue the manual dev smoke** on other routes for any further Turbopack-only console errors (user is going through them one by one).
3. **Deployed-stage smoke** (still pending): App Insights telemetry reports; a Pulse/SFTP job via `/admin/jobs` still runs in the worker; general routes fine. Note: worker + web share one Docker image; the worker runs SFTP via `ts-node`, unaffected by the bundler (see note 3.2.1).
4. **Residual risk to watch:** next-auth sign-in after `esmExternals` removal — looked fine in dev; confirm on stage.

**After open items close:** run `/verify MLID-2740`, then `/wrap-up MLID-2740` (PR doc to `docs/agomez/PR/MLID-2740.md`; user opens the PR manually — no `gh pr create`).

**How to start the dev server + log (from the worktree):**
```bash
cd /c/Users/alejandro.gomez/Dev/worktrees/MLID-2740
npm run dev:web 2>&1 | tee "../../li-call-processor/docs/agomez/logs/apps-web-$(date +%Y-%m-%d).txt"
```
(See `docs/agomez/dx/how-to-redirect-server-output.md` for both worktree and main-checkout variants.)

---

## Summary

`apps/web` runs on Next.js 16.2.9, where Turbopack is the default bundler, but currently opts out via an explicit `--webpack` flag on both `dev` and `build` because `next.config.mjs` carries a custom ~80-line `webpack()` block (7 workarounds for `ssh2`'s native binary, `applicationinsights`, Node built-ins, and a sibling-app exclusion). This task deletes that block, replaces it with the equivalent Turbopack-native config (`serverExternalPackages` plus one proven-needed client browser-stub alias for `applicationinsights`), drops `--webpack` from both scripts, and verifies the production build and client bundle are clean before a deployed-stage smoke pass.

---

## Codebase Analysis

**Current `apps/web/next.config.mjs`** (full top-level shape, confirmed by direct read):
- Top-of-file side effect: `if (process.env.NEXT_PHASE === 'phase-production-build') process.env.SKIP_DB_INIT = 'true';` — must be preserved verbatim, unrelated to the bundler.
- `env` — large explicit allow-list of `process.env.*` keys (NEXTAUTH_*, MONGODB_URI, AZURE_OPENAI_*, APPLICATIONINSIGHTS_CONNECTION_STRING, etc.) — bundler-agnostic, keep as-is.
- `typedRoutes: true` — keep.
- `pageExtensions: ['ts', 'tsx']` — keep.
- `transpilePackages: ['next-auth', '@auth/core']` — keep (see decision below).
- `experimental: { esmExternals: 'loose' }` — keep (see decision below).
- `images.remotePatterns` — keep.
- `reactStrictMode: false` — keep.
- `webpack: (config, { isServer, webpack }) => { ... }` — **delete entirely**. Contains: two `IgnorePlugin`s (`sshcrypto.node`, `li-public-api`), one `IgnorePlugin` for App Insights `diagnostic-channel`/`console.sub`, one `NormalModuleReplacementPlugin` for `^node:`, client `externals: ['ssh2', 'ssh2-sftp-client']` + `resolve.alias.applicationinsights = false`, server externals for `fs/promises`/`child_process`/`worker_threads`/`perf_hooks`/`applicationinsights`/`node:`, and `resolve.fallback` for the same four Node built-ins.

**`apps/web/package.json` scripts** — `--webpack` appears in exactly two strings:
- `"dev": "next dev --webpack -p 8080"`
- `"build": "next build --webpack"`

No other script, and no root `turbo.json` task, injects a bundler flag — `build:web`/`lint:web` at the repo root are plain `turbo build/lint --filter=@repo/web`.

**Why `serverExternalPackages` is load-bearing (not defensive).** `instrumentation.ts` (`register()`) unconditionally dynamic-imports `@/services/jobs/initialize-jobs` and calls `initializeAndStartPulse()` for **both** the web process and the worker process (`WORKER_PROCESS` only gates whether Pulse *starts processing*, not whether jobs are *defined*). That static import chain is: `initialize-jobs.ts` → `definitions/index.ts` (`defineAllJobs`) → job definition files → `*SftpService` → `services/storage/baseSftpService.ts` (`import Client from 'ssh2-sftp-client'`) → `ssh2` → native `sshcrypto.node`. So the **web server build** must successfully resolve this chain at build time; `serverExternalPackages: ['ssh2-sftp-client', 'ssh2', 'applicationinsights']` is what makes this resolve without bundling the native binary.

**Why the `applicationinsights` client stub is proven-needed (not speculative).** `apps/web/utils/logger.ts` does `await import('applicationinsights')` inside a class guarded at runtime by `typeof window === 'undefined' && !isEdgeRuntime()`. That is a **runtime** guard — it does not remove the `import('applicationinsights')` specifier from the module graph that a bundler statically discovers. `@/utils/logger` is imported by roughly 96 files under `components/`, many marked `'use client'` (e.g. `MessageList.tsx`, `ChatInterface.tsx`, `OrderForm.tsx`). Today's `webpack.resolve.alias.applicationinsights = false` (client-only) is what suppresses this leak. Turbopack needs the documented equivalent: `turbopack.resolveAlias: { applicationinsights: { browser: '<stub module>' } }`. This requires one new file — an empty stub module.

**Why `ssh2` / `ssh2-sftp-client` do NOT need a client stub.** All current importers of `services/storage/*` are server routes, server services, job definitions, or tests — zero client-component importers. `serverExternalPackages` alone is sufficient for these two.

**Node built-ins.** `app/api/admin/configuration/route.ts` does `import { exec } from 'child_process'` (server route, confirmed by direct read) and `services/vectorSearch/chunking/types.ts` does `import { createHash } from 'node:crypto'` (server-only, confirmed by direct read). Turbopack externalizes Node built-ins on the server automatically and natively supports the `node:` scheme, so no config entry is needed for either — these two files are the concrete "did the build still resolve this" checkpoints during verification.

**`li-public-api` exclusion** — no real import of the sibling NestJS app exists anywhere in `apps/web` (only comments / Key Vault secret-name strings). No Turbopack equivalent is needed; the `IgnorePlugin` for it is simply deleted.

**Deployment architecture (why Risk 2/5 from the pre-plan are low-severity here, not eliminated from verification).** Web and worker run the **same** Docker image (`apps/web/Dockerfile`); the worker overrides the container command to run `worker.ts` via `ts-node` against the full, unpruned `node_modules` (no `output: 'standalone'` in `next.config.mjs`). The worker's SFTP path is therefore unaffected by this migration — only `app/api/jobs/**` routes and the web server build are in the bundler's scope. `serverExternalPackages` controls only what `.next/` bundles, not what's present in the image, so removing it would not remove `ssh2` from the runtime image either way.

**No new dependency.** Turbopack ships inside `next`; no `npm install` step.

**`@next/bundle-analyzer`** is an unused devDependency (never imported in `next.config.mjs`). Out of scope for this task — not touched.

---

## Decisions carried from Open Questions (resolved, not left open)

1. **~~Keep `experimental.esmExternals: 'loose'`~~ → REMOVED during implementation (build-forced).** Original decision was to keep it, on the assumption Turbopack would ignore it harmlessly. **That assumption was wrong:** the real `next build` panicked with `TurbopackInternalError: esmExternals = "loose" is not supported` (Turbopack hard-errors on it rather than ignoring it). It has been removed from `next.config.mjs`. `transpilePackages: ['next-auth', '@auth/core']` is KEPT (Turbopack-supported) and now solely handles next-auth/@auth/core bundling; Turbopack's native ESM/CJS interop replaces what `esmExternals: 'loose'` did under webpack. The `next-auth` sign-in flow must be confirmed working in the dev + deployed smoke (Step 5) — this is the residual risk from dropping the escape hatch.

2. **Commit the `applicationinsights` client browser-stub alias proactively.** This is resolved by codebase finding #4 above (`utils/logger.ts` dynamic import reaches ~96 client-importing files today, currently suppressed only by the webpack alias) — it is proven-needed, not merely defensive. Do **not** add a browser stub for `ssh2` / `ssh2-sftp-client` — finding #5 shows zero client importers for those two, so add them only if step 3's client-bundle inspection proves otherwise (if that happens, treat it as a bug found during verification and add the stub in the same commit as the fix, documenting why).

---

## Implementation Steps

### Step 1 — Worktree setup, branch, and drop `--webpack` from scripts

- **Files**: `apps/web/package.json`
- **What**: Create a git worktree off `develop` for branch `feature/MLID-2740-webpack-to-turbopack`. In the worktree, edit `apps/web/package.json`:
  - `"dev": "next dev --webpack -p 8080"` → `"dev": "next dev -p 8080"`
  - `"build": "next build --webpack"` → `"build": "next build"`
  No other script changes. This is a config-only change (falls under the project's "Configuration Changes" TDD exception — no logic introduced). Commit alone so the diff for "the flag is gone" is isolated and reviewable independent of the config rewrite in Step 2.
- **Commit**: `[MLID-2740] - build(web): drop --webpack flag from dev and build scripts`

### Step 2 — Create the `applicationinsights` client browser-stub module

- **Files**: `apps/web/turbopack-empty-module.ts` (new)
- **What**: Add a minimal empty module with no exports and no side effects, used as the client-side substitute for `applicationinsights` so Turbopack's `resolveAlias` has a real file to point at in the browser compilation. Content:
  ```typescript
  // Empty stub used as a Turbopack client-bundle substitute for server-only
  // packages (see next.config.mjs `turbopack.resolveAlias`). Intentionally
  // has no exports — any code path that reaches this module in the browser
  // bundle is a bug (the real package must never be imported client-side).
  export {};
  ```
  No test needed — this is a static, side-effect-free stub with zero behavior (same "simple type/config, no logic" exception). Naming: `turbopack-empty-module.ts` at the `apps/web` root, next to `next.config.mjs`, so its relative path from the config is obvious (`./turbopack-empty-module.ts`).
- **Commit**: `[MLID-2740] - build(web): add empty stub module for Turbopack client resolveAlias`

### Step 3 — Rewrite `next.config.mjs`: delete the webpack block, add Turbopack config

- **Files**: `apps/web/next.config.mjs`
- **What**: Replace the file contents so the final shape is:
  ```javascript
  /** @type {import('next').NextConfig} */

  // Set build phase environment variable
  if (process.env.NEXT_PHASE === 'phase-production-build') {
    process.env.SKIP_DB_INIT = 'true';
  }

  const nextConfig = {
    env: {
      NEXTAUTH_URL: process.env.NEXTAUTH_URL,
      NEXTAUTH_SECRET: process.env.NEXTAUTH_SECRET,
      GOOGLE_CLIENT_ID: process.env.GOOGLE_CLIENT_ID,
      GOOGLE_CLIENT_SECRET: process.env.GOOGLE_CLIENT_SECRET,
      MONGODB_URI: process.env.MONGODB_URI,
      WS_MAKE_CALL_URL: process.env.WS_MAKE_CALL_URL,
      OPENAI_API_KEY: process.env.OPENAI_API_KEY,
      AZURE_OPENAI_API_KEY: process.env.AZURE_OPENAI_API_KEY,
      AZURE_OPENAI_ENDPOINT: process.env.AZURE_OPENAI_ENDPOINT,
      AZURE_OPENAI_DEPLOYMENT_NAME: process.env.AZURE_OPENAI_DEPLOYMENT_NAME,
      MAX_PARALLEL_CALLS: process.env.MAX_PARALLEL_CALLS,
      APPLICATIONINSIGHTS_CONNECTION_STRING:
        process.env.APPLICATIONINSIGHTS_CONNECTION_STRING,
      NEXT_PUBLIC_APPLICATIONINSIGHTS_CONNECTION_STRING:
        process.env.NEXT_PUBLIC_APPLICATIONINSIGHTS_CONNECTION_STRING,
      WEBHOOK_SECRET: process.env.WEBHOOK_SECRET,
      REPORT_RECIPIENTS: process.env.REPORT_RECIPIENTS,
    },
    typedRoutes: true, // Prevents treating li-public-api as app directory
    pageExtensions: ['ts', 'tsx'],
    // Auth.js v5 (next-auth) and @auth/core are ESM-only. Without this the
    // build externalizes them as CommonJS and fails with "ESM packages need
    // to be imported". Transpiling makes Next bundle them instead.
    transpilePackages: ['next-auth', '@auth/core'],
    // Allow CommonJS server bundles (e.g. pages/api routes) to import the
    // ESM-only next-auth. See https://nextjs.org/docs/messages/import-esm-externals
    experimental: {
      esmExternals: 'loose',
    },
    images: {
      remotePatterns: [
        {
          protocol: 'https',
          hostname: '**',
        },
      ],
    },
    reactStrictMode: false,
    // These packages are left unbundled and require()'d from node_modules at
    // runtime instead of being bundled by Next. Required for:
    // - ssh2 / ssh2-sftp-client: ssh2 loads a native sshcrypto.node binary
    //   that must never be handed to the bundler. Reached at server build
    //   time via instrumentation.ts -> initialize-jobs -> job definitions ->
    //   *SftpService -> services/storage/baseSftpService.ts.
    // - applicationinsights: has optional diagnostic-channel sub-imports
    //   that only resolve correctly via Node's runtime require(), not a
    //   bundler's static analysis.
    serverExternalPackages: ['ssh2-sftp-client', 'ssh2', 'applicationinsights'],
    turbopack: {
      resolveAlias: {
        // utils/logger.ts does a guarded dynamic import('applicationinsights')
        // that is imported by ~96 client components via @/utils/logger. The
        // runtime guard (typeof window === 'undefined') does not remove the
        // import specifier from the client module graph, so without this
        // alias the client bundle would include applicationinsights code.
        applicationinsights: {
          browser: './turbopack-empty-module.ts',
        },
      },
    },
  };

  export default nextConfig;
  ```
  Net effect: deletes ~90 lines (the whole `webpack()` function), adds `serverExternalPackages` (3 lines) and `turbopack.resolveAlias` (1 alias) — a net reduction in file size. No change to `env`, `typedRoutes`, `pageExtensions`, `transpilePackages`, `experimental`, `images`, `reactStrictMode`, or the `SKIP_DB_INIT` side effect.
- **Commit**: `[MLID-2740] - build(web): replace webpack() block with Turbopack serverExternalPackages and resolveAlias`

### Step 4 — Local verification pass (build, types, tests, client-bundle inspection)

- **Files**: none (verification only — no source changes expected; this step exists to produce evidence, and to be the checkpoint where any leak found gets fixed in a follow-up commit before moving on)
- **What**: Run, in order, from `apps/web/`:
  1. `npm run types:check` — must pass unchanged (no type surface changed).
  2. `npm run test` — full Jest suite must stay green. This is a regression guard only: no test in the repo references `next.config`, `--webpack`, or build-output snapshots, and Jest never shells out to `next build`/`next dev`, so this step does **not** validate the bundler swap itself — it only proves the config edit didn't break unrelated app code (e.g. accidentally deleting an env key).
  3. `npm run build` (now Turbopack, since Step 1 dropped `--webpack`) — must complete with exit code 0, no "Module not found," no native-module (`.node`) resolution errors, no `ssh2`/`applicationinsights`/`child_process`/`node:crypto` warnings.
  4. Client-bundle inspection — confirm the forbidden modules are absent from the client output:
     ```bash
     grep -rl "ssh2-sftp-client" .next/static/chunks apps/web/.next/static/chunks 2>/dev/null
     grep -rl "applicationinsights" .next/static/chunks apps/web/.next/static/chunks 2>/dev/null
     grep -rlE "require\(.node:" .next/static/chunks apps/web/.next/static/chunks 2>/dev/null
     ```
     (Run from `apps/web/`, so the real path is `.next/static/chunks`.) Expected: **no matches** for any of the three greps. If a match is found, that is a genuine leak — add the corresponding `turbopack.resolveAlias` browser-stub entry (mirroring Step 3's `applicationinsights` pattern) in a follow-up commit and re-run this step, rather than silently proceeding.
  5. `npm run lint` — confirm clean (note: `next.config.mjs` itself is excluded from ESLint scope per existing repo config, so this step validates the rest of the diff, not the config file).
- **Evidence to capture** (for the PR / for the user, since `npm run dev`/build verification and browser checks are performed by the user per team practice): terminal output of `npm run build` showing success, and the three `grep` commands above showing zero matches.
- **No commit** if all green (verification-only step). If a leak is found and fixed, that fix is its own commit per Step 3's note above, titled e.g. `[MLID-2740] - fix(web): add resolveAlias stub for <module> after client-bundle leak found in Turbopack build`.

### Step 5 — Manual dev-server and deployed-stage smoke checklist (user-performed)

- **Files**: none
- **What**: Per standing team practice, the user runs `npm run dev` and performs all browser/deploy verification. This step is a checklist handed to the user, not something the AI executes. Checklist (copy-pasteable):

  **Local dev (`npm run dev`, now Turbopack):**
  - [ ] `npm run dev` starts without error on port 8080.
  - [ ] App loads in browser; no console errors mentioning `ssh2`, `applicationinsights`, `child_process`, or `Module not found`.
  - [ ] Navigate to a page that renders a component importing `@/utils/logger` client-side (e.g. any page using `ChatInterface` or `MessageList`) — confirm no browser console error and no network fetch for an `applicationinsights` chunk.
  - [ ] Trigger the `/api/admin/configuration` route (uses `child_process`) — confirm it responds normally.
  - [ ] Trigger any route touching `services/vectorSearch/chunking` (`node:crypto` usage) — confirm it responds normally.

  **Deployed stage (after this branch is merged and deployed):**
  - [ ] Web app boots; `instrumentation.ts` logs `[instrumentation] Application Insights initialized` (not the "disabled" branch) — confirms `serverExternalPackages` didn't break the App Insights dynamic import server-side.
  - [ ] Confirm traces/telemetry appear in Application Insights for the stage resource (no diagnostic-channel crash at startup).
  - [ ] Trigger or wait for a scheduled background job (Pulse) that reaches an SFTP/RxPreferred step — confirm the worker container (`ts-node worker.ts`, unaffected by this migration since it doesn't run through `next build`) still completes the transfer. Use `/admin/jobs` to trigger on demand rather than waiting for the schedule.
  - [ ] Spot-check 2-3 unrelated routes (e.g. patient list, order details) to confirm no regressions from the config rewrite.
- **No commit** (verification only).

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/package.json` | Modify | Remove `--webpack` from `dev` and `build` scripts |
| `apps/web/next.config.mjs` | Modify | Delete `webpack()` block; add `serverExternalPackages` and `turbopack.resolveAlias`; **remove `experimental.esmExternals: 'loose'`** (Turbopack hard-errors on it — see deviation below); all other keys and the `SKIP_DB_INIT` side effect unchanged |
| `apps/web/turbopack-empty-module.ts` | Create | Empty stub module used as the client-bundle substitute for `applicationinsights` via `turbopack.resolveAlias` |
| `apps/web/components/Modules/patient/components/patientDetailsInfo/styles.module.css` | Modify | Fix two malformed CSS-Modules selectors `.item:global(> span)` / `.dataItem:global(> span)` → `.item > span` / `.dataItem > span` (Turbopack's CSS parser rejects the bare-combinator `:global()`; webpack tolerated it — see deviation below) |
| `apps/web/app/layout.tsx` | Modify | Wrap the app in `AppRouterCacheProvider` from `@mui/material-nextjs/v16-appRouter` to fix a Turbopack-only emotion hydration mismatch (see deviation 3 below) |
| `apps/web/package.json` + `package-lock.json` | Modify | Add dependency `@mui/material-nextjs@^7.3.10` (official MUI package, matches MUI 7) for the above |

### Implementation deviations (discovered by the real Turbopack build, not in the original plan)

Both were pre-existing configurations/code that **webpack silently tolerated but Turbopack rejects** — exactly the "incomplete translation / hidden strictness" class of risk the ticket flagged. Each was caught by running the actual `next build`, and each has an unambiguous fix:

1. **`experimental.esmExternals: 'loose'` removed.** Turbopack panics with `esmExternals = "loose" is not supported`. Removed; `transpilePackages` + Turbopack native interop cover next-auth. Residual risk: confirm next-auth sign-in in dev/deploy smoke.
2. **Two malformed `:global(> span)` selectors fixed** in `patientDetailsInfo/styles.module.css`. Turbopack's stricter CSS parser reports `Invalid empty selector`. Fixed to plain `> span` (behavior-identical — tag selectors are never scoped by CSS Modules; matches the sibling `.item > span:first-of-type` rules already in the same file). A full scan of all 14 `.module.css` files with `:global(` confirmed these two lines were the ONLY malformed instances — every other `:global(...)` wraps a real selector and is valid.
3. **`AppRouterCacheProvider` added (emotion hydration fix)** — caught during dev verification, not the build. Under Turbopack the app hydration-mismatched on every page (server rendered an emotion `<style data-emotion=css-global>` where the client rendered a CSS-Module div, first hit at `Notifications.tsx`). Root cause: the app never had a MUI App-Router emotion cache setup, so emotion injected its global style inline during SSR; webpack's ordering hid it, Turbopack exposed it. Confirmed Turbopack-only (absent on `develop`). Fix: wrap the app in `AppRouterCacheProvider` (`@mui/material-nextjs/v16-appRouter`), which flushes emotion styles into `<head>` via `useServerInsertedHTML`. Adds the `@mui/material-nextjs` dependency. User-verified the hydration error is gone in dev.

---

## Testing Strategy

This is a build-configuration change with no application logic — it falls under the project's documented "Configuration Changes" TDD exception (zero conditional logic introduced; a red/green unit test for "the bundler is Turbopack" would be artificial and is intentionally not written). Verification instead rests on:

- **Regression guard**: full existing Jest suite (`npm run test`) stays green — proves the config edit didn't break unrelated app code. This does NOT validate the bundler swap itself (no test shells out to `next build`/`next dev`).
- **Type safety**: `npm run types:check` passes unchanged.
- **Build verification**: `npm run build` (Turbopack) exits 0 with no native-module or server-only-module resolution errors — this is the primary evidence for the acceptance criterion "production build green."
- **Client-bundle inspection**: the three `grep` commands in Step 4 against `.next/static/chunks` confirming absence of `ssh2-sftp-client`, `applicationinsights`, and `node:`-scheme requires — this is the primary evidence for the acceptance criterion "client bundle excludes forbidden modules."
- **Manual dev verification**: user runs `npm run dev` and exercises the app per the Step 5 checklist — this is the primary evidence for "app works in dev."
- **Manual deployed-stage smoke**: user exercises the Step 5 deployed checklist post-deploy — this is the primary evidence for "app works in a deployed environment," covering SFTP/RxPreferred (via the worker, unaffected by the bundler but confirmed anyway), Application Insights, and background jobs (Pulse).
- **Mock boundaries**: N/A — no new runtime code, nothing to mock.

---

## Security Considerations

- **Input validation**: N/A — no new input-handling code.
- **Authorization**: N/A — no new routes, no RBAC surface touched.
- **Feature flags**: N/A — not a feature-flagged change; it's a build tool swap affecting all users identically.
- **PHI handling**: No change to any PHI-handling code path. The one risk this task exists to close is a *build-time* risk (server-only/native modules leaking into the client bundle) rather than a data-handling risk — verified via Step 4's client-bundle grep.
- **New environment variables**: None introduced.

---

## Open Questions

None — the two open items from the pre-plan (keep vs. drop `esmExternals`/`transpilePackages`; whether to proactively stub `ssh2`/`ssh2-sftp-client`) are resolved above under "Decisions carried from Open Questions."

---

## Conclusion — Dev vs. Production impact

To set expectations for the team: the visible speedup is a **development** experience. It is worth being precise about what changes in each phase, because "production" splits into two very different things — the build and the runtime.

| Phase | webpack → Turbopack change? | Who benefits |
|---|---|---|
| **Dev compile** (`next dev`, on-demand per route) | Large speedup — routes compile/hot-reload much faster | Developers, every day |
| **Prod build** (`next build`) | Now runs on Turbopack (the `--webpack` flag is gone); typically faster | CI/CD and Docker image build time |
| **Prod runtime** (`next start`, serving requests) | No per-route change — output is pre-built either way; the bundler does not run at request time | ~No user-facing change |

Why: in dev, Next.js compiles each route the first time it is visited — that on-demand step is what was slow on webpack and is now fast on Turbopack. In production everything is compiled ahead of time during `next build`, so at request time the server just serves pre-built output and no compilation happens. The production win is therefore a **faster build**, not faster request serving.

Minor caveat: Turbopack and webpack can emit slightly different bundles (chunk splitting, tree-shaking, minification), which can marginally shift client download size. It is a small, secondary effect — Next.js 16 even dropped its "First Load JS" build metric partly because the two bundlers accounted for it differently.

**Staging verification (2026-07-17):** the app was tested on staging (which runs the production build) and works well with no errors. Note that staging runs the production build, so the dev-only error overlay widget (bottom-left corner) is not present — that overlay is a `next dev` feature and does not exist in a production build; its absence is expected, not a problem.
