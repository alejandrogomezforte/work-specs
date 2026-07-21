# MLID-2740 — Migrate LISA web bundler from webpack to Turbopack (Pre-Plan)

> This is a **pre-plan**, not the implementation plan. It is the briefing that the real plan (built later via `/plan-task MLID-2740`) and any resulting work will be based on. The Jira description is the starting point; the real specification is the Turbopack config surface plus what our own `next.config.mjs` actually does.

**Ticket:** [MLID-2740](https://localinfusion.atlassian.net/browse/MLID-2740) — Task — Priority Medium — 5 SP — Status IN DEVELOPMENT. Epic parent: MLID-983 (Dev Tech Enhancements). Relates to MLID-364 (Next.js 16 migration, already in production). This ticket closes the explicit follow-up that MLID-364 deferred: MLID-364 kept webpack via the `--webpack` escape-hatch flag so the framework upgrade would not also carry a bundler migration.

**Procedure note:** implemented in a **git worktree**, same procedure as the Next.js upgrade. Branch `feature/MLID-2740-webpack-to-turbopack` off `develop`.

---

## 1. High-Level Summary

Next.js 16 (we are on **16.2.9**) makes **Turbopack the default** bundler for both `next dev` and `next build`. Today `apps/web` opts out with an explicit `--webpack` flag because `next.config.mjs` carries an ~80-line custom `webpack()` block. That flag is transitional — Next.js has signalled webpack support is being wound down and a future release is expected to remove it. This ticket moves `apps/web` onto Turbopack and re-expresses (or retires) each webpack workaround, because **Turbopack does not run the `webpack()` function and does not support webpack plugins**.

**The single most important framing:** this is *not* a 1:1 port of 7 webpack workarounds into 7 Turbopack settings. Most of those workarounds exist to fight webpack's default behavior of trying to bundle everything — and Turbopack + the App Router do not have those defaults. Once you look at each one against how our code actually imports these modules, the custom config **collapses to a very small Turbopack config** — realistically `serverExternalPackages` plus, at most, a couple of defensive `turbopack.resolveAlias` browser stubs. The residual effort and risk are almost entirely in **verification**, not in writing config.

**Where the risk actually lives (per the ticket, Medium):** an incomplete translation can silently bundle server-only or native code into the *client* bundle, or fail to load the native SSH `.node` binary at runtime in a deployed environment. Both are runtime/production failure modes that a green local `next build` will not necessarily reveal. So the deliverable is "small config change + a real verification pass," and the verification pass is the expensive half.

---

## 2. High-Level Effort Estimation

| Dimension | Estimate |
|---|---|
| **Story points** | 5 SP (as ticketed — the number reflects verification + deploy risk, not config volume) |
| **Config edit** | Small — likely a **net reduction** in `next.config.mjs` (delete ~80 lines of `webpack()`, add a handful of Turbopack lines) |
| **Files touched** | ~2: `apps/web/next.config.mjs` and `apps/web/package.json` (scripts). Possibly one tiny `empty.ts` stub if a defensive alias is needed. |
| **Cost driver** | Real-browser + deployed-environment verification that nothing server-only/native leaks to the client and that SFTP / App Insights / background jobs still work — **not** the config edit |

**No new npm dependencies.** Turbopack ships inside `next`. No install step.

---

## 3. Detail Level

### 3.1 What the custom `webpack()` block does today (7 workarounds)

Verbatim from `apps/web/next.config.mjs`:

1. `IgnorePlugin` for `sshcrypto\.node$` — the native SSH binary pulled in transitively by `ssh2`.
2. `IgnorePlugin` for `li-public-api` — stop webpack compiling the sibling NestJS app folder.
3. `IgnorePlugin` for `diagnostic-channel|console\.sub$` scoped to `applicationinsights` — silence App Insights' optional-dependency sub-imports.
4. `NormalModuleReplacementPlugin` for `^node:` — strip the `node:` import scheme so webpack resolves the bare built-in.
5. **Client externals + alias**: `externals: ['ssh2','ssh2-sftp-client']` and `resolve.alias.applicationinsights = false` — keep these out of the browser bundle.
6. **Server externals**: `fs/promises`, `child_process`, `worker_threads`, `perf_hooks`, plus a function that externalizes `applicationinsights` and `node:` imports as `commonjs`.
7. `resolve.fallback` = `false` for `fs/promises`, `child_process`, `worker_threads`, `perf_hooks`.

### 3.2 How our code actually imports these modules (the fact base that shrinks the job)

Measured against `apps/web` source:

- **`ssh2-sftp-client` (which pulls in `ssh2` → `sshcrypto.node`)**: imported statically only in `services/storage/baseSftpService.ts` — a **server-only** service. Consumed by `worker.ts` and by `app/api/jobs/**` route handlers. `ssh2-sftp-client` is a direct dep (`^12.0.0`); `ssh2` is transitive.
- **`applicationinsights`** (`^3.3.0`, direct): **never statically imported.** Loaded only via **guarded dynamic `import('applicationinsights')`** in `instrumentation.ts` and `utils/logger.ts`, both gated behind `process.env.NEXT_RUNTIME === 'nodejs'`. Never reaches client code.
- **`li-public-api`**: **zero real imports.** Every hit in `apps/web` is a comment or a Key-Vault secret-name string (e.g. `li-public-api-basic-auth-username`). The sibling NestJS app is not imported by the web app at all.
- **`node:` scheme**: exactly **one** app-source usage — `import { createHash } from 'node:crypto'` in `services/vectorSearch/chunking/types.ts` (server-only).
- **`worker.ts` runs via `ts-node`** (`worker: ts-node -r tsconfig-paths/register worker.ts`) — it is **not** built by Next/Turbopack. So the worker's SFTP path is unaffected by this migration; only the `app/api/jobs/**` routes are in scope for the bundler.

### 3.2.1 Deployment architecture — NOTE (verified from CI workflows + Dockerfile)

> Added after reviewing `.github/workflows/azure-deploy.yml`, `.github/workflows/deploy-container-app.yml`, and `apps/web/Dockerfile`. This note is what resolves open question 1 (the deployed SFTP test).

**The web app and the pulse worker run the same single Docker image.** In `azure-deploy.yml`, both `build-and-deploy-main-app-*` (web) and `build-and-deploy-worker-app-*` (worker) use `service_name: local-infusion-app` and `dockerfile_path: apps/web/Dockerfile`; the worker job is explicitly commented "Same image as web." The worker container (`li-stage-local-infusion-worker`, `li-prod-local-infusion-worker`) is that same image started with a different command.

**That image (`apps/web/Dockerfile`) does the following:**
1. Installs `python3` + `build-essential` — the toolchain that compiles native modules such as `ssh2`'s `sshcrypto.node`.
2. Runs `npm ci` — installs the **complete** `node_modules`, **not pruned**.
3. Copies the **full source tree** into the image.
4. Runs `npm run build:web` (= `next build` — the only step Turbopack changes).
5. Default `CMD` is `npm start` (= `next start`, the web app). The worker container overrides the command to run `worker.ts` via `ts-node`.

**Why this de-risks the migration:**
- The worker executes the SFTP jobs through **`ts-node worker.ts` at runtime**, using the full source + full `node_modules` in the image — **not** the Next.js build output. Turbopack only changes `next build`, which is a different artifact. **Turbopack does not change how the worker runs SFTP transfers.**
- Because the image keeps the **full, unpruned `node_modules`** (no `output: 'standalone'`), `serverExternalPackages` cannot remove `ssh2`/`applicationinsights` from the image. It only changes what is bundled into `.next/`; the packages (and the compiled native binary) stay on disk for both the web app and the worker to `require()` at runtime.
- The web app itself only **schedules** these jobs (it "initializes for job scheduling"; the worker "processes jobs"), so the web process never performs a transfer regardless of bundler.

**Consequence:** the correctness of the file transfers does not depend on this change, and no special SFTP test needs to be built. Since web + worker share one image, a single stage deploy of the Turbopack-built image exercises both paths at once.

### 3.3 Proposed webpack → Turbopack mapping

The recommended primary tool is **`serverExternalPackages`** — a top-level Next.js option (works with both bundlers) that tells Next to leave a package **unbundled** and `require()` it from `node_modules` at runtime on the server. For packages that contain native binaries (`ssh2`) or optional-dependency sub-imports (`applicationinsights`), this is exactly the intended mechanism and it makes several webpack workarounds unnecessary at once.

| # | Webpack workaround | Turbopack equivalent | Rationale |
|---|---|---|---|
| 1 | `IgnorePlugin` for `sshcrypto.node` | **`serverExternalPackages: ['ssh2-sftp-client', 'ssh2']`** | Externalized packages are `require()`d at runtime, so the native `.node` binary loads normally and is never handed to the bundler. No ignore needed. |
| 2 | `IgnorePlugin` for `li-public-api` | **Delete — not needed** | No real imports exist. Turbopack only resolves what is imported; the sibling app is never pulled in. `turbopack.root` auto-resolves to the monorepo root via `package-lock.json`. |
| 3 | `IgnorePlugin` for App Insights `diagnostic-channel`/`console.sub` | **Covered by `serverExternalPackages: ['applicationinsights']`** | Unbundled → its optional sub-imports are resolved by Node at runtime, not the bundler, so the build-time complaints disappear. |
| 4 | `NormalModuleReplacementPlugin` for `^node:` | **Delete — not needed** | Turbopack supports the `node:` scheme natively. Our single `node:crypto` usage resolves without help. |
| 5 | Client externals + `applicationinsights: false` alias | **Likely unnecessary; defensive fallback = `turbopack.resolveAlias` browser stub** | All three modules are server-only in our code, so the App Router should never bundle them into the client. If a client build error surfaces, use the officially documented pattern `resolveAlias: { ssh2: { browser: './empty.ts' }, ... }`. Prefer to add these only if verification shows a leak. |
| 6 | Server externals (`fs/promises`, `child_process`, `worker_threads`, `perf_hooks`, App Insights, `node:`) | **Delete — Turbopack externalizes Node built-ins on the server automatically**; App Insights handled by #3 | These externals existed because webpack tried to bundle Node built-ins. Turbopack does not. |
| 7 | `resolve.fallback: false` for Node built-ins | **Delete; defensive fallback = browser stub if needed** | Same as #5 — only required if client code references a Node built-in, which our audit does not show. Documented Turbopack replacement is `turbopack.resolveAlias: { <mod>: { browser: './empty.ts' } }`. |

**Net expected config after migration:** delete the entire `webpack()` block; add `serverExternalPackages: ['ssh2-sftp-client', 'ssh2', 'applicationinsights']` at the top level; keep a documented `turbopack.resolveAlias` browser-stub block on standby (only committed if verification proves a client leak). Remove `--webpack` from `dev` and `build` scripts.

### 3.4 Config items that are NOT part of the `webpack()` block but need a Turbopack sanity check

These already live in `next.config.mjs` and must be confirmed to behave the same under Turbopack (they are not webpack-plugin based, so expected fine, but verify):

- `experimental.esmExternals: 'loose'` — this is a **webpack-oriented** ESM/CJS interop knob. Turbopack handles ESM/CJS interop natively; confirm whether it is still needed or becomes a no-op. Related to the next-auth ESM handling below.
- `transpilePackages: ['next-auth', '@auth/core']` — supported by Turbopack. Confirm next-auth (`5.0.0-beta.31`) still bundles correctly without the webpack externals path.
- `typedRoutes`, `pageExtensions`, `images.remotePatterns`, `env`, `reactStrictMode` — bundler-agnostic; no change expected.

### 3.5 Risks (where time actually goes)

**Risk 1 — Client-bundle leakage (the ticket's primary risk).** The whole point of workarounds #5 and #7 was to keep server-only/native code out of the browser. Our import audit says none of these modules reach client code, so Turbopack should exclude them by default — but "should" is not "verified." Mitigation: inspect the actual client output and/or drive the app and watch for `Module not found` / native-module errors. Add the documented `resolveAlias` browser stubs only if a leak is proven.

**Risk 2 — Native `.node` at runtime in a deployed environment. (LARGELY RESOLVED — see note 3.2.1.)** The SFTP transfer runs in the **worker** via `ts-node`, using the full source + unpruned `node_modules` of a shared image whose native binary is compiled at `npm ci`. Turbopack changes only `next build`, not the worker's runtime path, and `serverExternalPackages` cannot remove `ssh2` from the (non-standalone) image. So the transfer's correctness does not depend on this change. Residual mitigation is light: one stage deploy of the Turbopack-built image, then confirm a scheduled worker SFTP job still runs (expected, since its path is unchanged).

**Risk 3 — App Insights / instrumentation.** `instrumentation.ts` sets up App Insights via guarded dynamic import. Confirm `serverExternalPackages: ['applicationinsights']` keeps telemetry flowing after deploy (traces appear, no diagnostic-channel crash).

**Risk 4 — Background jobs (Pulse).** `instrumentation.ts` also boots Pulse. Confirm jobs still initialize/schedule under the Turbopack build.

**Risk 5 — Standalone/Docker output. (RESOLVED — see note 3.2.1.)** Confirmed: `next.config.mjs` has **no** `output` key (not standalone) and `apps/web/Dockerfile` keeps the full, unpruned `node_modules` and runs `npm start`. So externalized packages are guaranteed present in the runtime image; dropping `--webpack` only changes the `next build` step. No pruning/tracing path to break.

### 3.6 Suggested strategy of implementation

Small, reversible, verification-first. In the worktree off `develop`:

1. **Dev first, config minimal.** Drop `--webpack` from `dev`; delete the `webpack()` block; add `serverExternalPackages`. Run `next dev` and exercise the app. Fix leaks with documented `resolveAlias` stubs only as proven necessary.
2. **Then build.** Drop `--webpack` from `build`; run `next build` (now Turbopack). Confirm green with no native/server-only errors.
3. **Inspect the client output** for `ssh2`, `ssh2-sftp-client`, `applicationinsights`, and Node built-ins — confirm absence (the AC's explicit "no leakage" check).
4. **Route-by-route smoke** of the SFTP/RxPreferred, App Insights, and background-job surfaces — locally, then in a deployed stage (Risk 2/3/4 can only be closed on deploy).
5. Keep the diff tiny and self-documenting; the value is in the verification evidence, not config cleverness.

**Division of labor:** per standing team practice, the user runs the dev server and performs all browser/deploy verification. AI produces the config change plus a precise, copy-pasteable verification checklist and the exact commands/evidence to capture.

### 3.7 Acceptance criteria (from the ticket)

- `next dev` and `next build` run on Turbopack (no `--webpack`).
- Production build green; no native `.node` / server-only errors.
- Client bundle verified to exclude `ssh2`, `ssh2-sftp-client`, `applicationinsights`, and Node built-ins.
- App runs correctly in dev and in a deployed environment (route-by-route smoke of SFTP/RxPreferred, App Insights, background jobs).

---

## 4. Open questions to resolve before / during the real plan

1. **Verification depth for "deployed environment." — RESOLVED (see note 3.2.1).** The worker (`li-stage-local-infusion-worker` / `li-prod-local-infusion-worker`) runs the **same image** as the web app but executes SFTP jobs via `ts-node` at runtime — a path Turbopack does not touch — and the shared image keeps full `node_modules` + the compiled native binary. No special SFTP test is needed. The deployed check reduces to a normal stage smoke of the Turbopack-built image: confirm web routes + Application Insights, and confirm a scheduled worker SFTP job still runs.
2. **`esmExternals: 'loose'` and `transpilePackages`** — keep as-is, or is dropping them in scope once Turbopack proves next-auth bundles natively? Recommend: keep for now, treat any removal as a separate cleanup.
3. **Docker/standalone output mode** — confirm the Dockerfile's build path so `serverExternalPackages` are guaranteed present in the runtime image (Risk 5).
4. **Defensive `resolveAlias` stubs** — commit them proactively (belt-and-suspenders) or only if a leak is proven? Recommend: only if proven, to keep the config honest.

---

**Next step:** once the open questions above are settled, run `/plan-task MLID-2740` using this pre-plan as context to produce the implementation plan.
