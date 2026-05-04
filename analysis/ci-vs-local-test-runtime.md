# CI/CD vs. local test runtime gap (apps/web)

**Date:** 2026-04-28
**Trigger:** PR check on the MLID-2011 epic took ~83 minutes for the apps/web Jest stage; locally the same suite ran in ~92 seconds. ~54× wall-clock difference.
**Observation surface:** `apps/web/` Jest suite (Web App Tests job in `unit-tests.yml`).

> **Update 2026-04-28:** ran the exact CI flag set (`--ci --changedSince=origin/develop --coverage ...`) locally as well — measured ~177 s (vs. 92 s no-coverage). That gives us three data points and lets us isolate coverage cost from runner-constraint cost. See §1.1.

---

## 1. Local baseline

Run from `apps/web/` on developer laptop (Windows, multi-core), no coverage:

```bash
npx jest --no-coverage
```

| Metric              | Value                  |
|---------------------|------------------------|
| Test suites total   | 867                    |
| Suites passed       | 866                    |
| Suites failed       | 1 (V8 crash, see §5)   |
| Tests passed        | 14,606                 |
| Tests skipped       | 2                      |
| Tests failed        | 0                      |
| **Total time**      | **91.86 s** (~1.5 min) |

### Top 10 slowest suites (local, no coverage)

| #  | Suite                                                                                                                           | Time     |
|----|---------------------------------------------------------------------------------------------------------------------------------|----------|
| 1  | `services/apiKeys/generateKey.test.ts`                                                                                          | 44.17 s  |
| 2  | `app/intakes/IntakesPageLegacy.test.tsx`                                                                                        | 23.53 s  |
| 3  | `services/mongodb/__tests__/document.test.ts`                                                                                   | 23.26 s  |
| 4  | `components/Modules/provider-management/ProviderOfficeManagement/ProviderOfficeManagement.test.tsx`                             | 19.66 s  |
| 5  | `components/Modules/admin/drug-clinical-review/drug-detail-view/components/EditDrugModal.test.tsx`                              | 18.16 s  |
| 6  | `services/apiKeys/verifyApiKey.test.ts`                                                                                         | 16.17 s  |
| 7  | `components/UI/FileViewer/EmbeddedFileViewer.test.tsx`                                                                          | 16.15 s  |
| 8  | `components/Modules/admin/drug-clinical-review/drug-detail-view/__tests__/index.test.tsx`                                       | 16.14 s  |
| 9  | `services/apiKeys/listApiKeys.test.ts`                                                                                          | 15.12 s  |
| 10 | `services/apiKeys/createApiKey.test.ts`                                                                                         | 15.10 s  |

The whole `services/apiKeys/*` cluster (6 suites: `generate`, `verify`, `list`, `create`, `update`, `revoke`) totals ~120 s on its own. Likely candidate for shared `beforeAll` Mongo setup instead of per-file connection.

## 1.1. Local with the exact CI flag set (apples-to-apples)

Re-ran from `apps/web/` with the same flag set CI uses:

```bash
CI=true NODE_OPTIONS='--max-old-space-size=4096' \
  npx jest --ci --changedSince=origin/develop --coverage \
    --coverageReporters=json-summary --coverageReporters=text \
    --coverageThreshold='{}'
```

| Metric              | Value                          |
|---------------------|--------------------------------|
| Test suites total   | 527 (filtered by `--changedSince`) |
| Suites passed       | 526                            |
| Suites failed       | 1 (same V8 crash as §5)        |
| Tests passed        | 8,754                          |
| Tests skipped       | 3                              |
| **Total time**      | **176.6 s** (~2:57)            |
| **Per-test rate**   | ~49 tests/s                    |

### Three-way comparison

| Run                                | Suites run | Tests | Time      | Per-test rate |
|------------------------------------|------------|-------|-----------|---------------|
| Local, no coverage                 | 867        | 14,606 | 92 s      | 159 tests/s   |
| Local, **CI flags** (cov + changedSince) | 527  | 8,754 | 176.6 s   | 49 tests/s    |
| **CI**, same flags                 | 527        | 8,754 | ~4,980 s (~83 min) | 1.76 tests/s |

### Per-factor isolation

- **`--changedSince=origin/develop`** — drops 867 → 527 suites (~40% reduction). The epic *did* get a partial filter benefit; the touched modules (Notification model, SignalR service, feature flag enum, audit-log config) are widely imported so the dependency graph still pulls in most of the suite.
- **`--coverage` instrumentation** — accounts for the local per-test rate dropping from 159 → 49 tests/s, **~3.2× slowdown** in isolation on a multi-core dev box. Earlier estimate was 5–10×; the real measured number is closer to **3×** when the rest of the environment is held constant.
- **Runner constraint (CI vs. local with coverage normalized)** — after taking coverage out of the equation, local is 49 tests/s vs. CI 1.76 tests/s: **~28× slower** on `ubuntu-latest`. That's pure runner: 1 Jest worker on 2 vCPU vs. 7+ on the dev box, plus retry overhead from the V8 crash.

Updated mental model: coverage is responsible for **~3× of the gap**; **the remaining ~28× is the single-worker 2-vCPU runner**. Sharding is therefore the larger raw win, not coverage removal.

---

## 2. CI configuration that runs on PR

`.github/workflows/unit-tests.yml` — triggered on `pull_request` to `develop`. Relevant step (lines 110-121):

```yaml
- name: Run tests with coverage for changed files
  id: test
  working-directory: apps/web
  run: |
    npx jest --ci \
      --changedSince=origin/develop \
      --coverage \
      --coverageReporters=json-summary --coverageReporters=text \
      --coverageThreshold='{}'
  env:
    NODE_OPTIONS: '--max-old-space-size=4096'
    CI: 'true'
```

Runner: `ubuntu-latest` (single GitHub-hosted, **2 vCPU / 7 GB RAM**).

For comparison, the develop-merge job in `.github/workflows/unit-tests-full.yml` runs:

```yaml
npx jest --ci --coverage --coverageReporters=json --shard=${{ matrix.shard }}/4
```

— same coverage flag, but **4-way matrix shard** across 4 parallel runners.

---

## 3. Why CI is ~54× slower than local

The 83-minute wall-clock is the product of compounding factors. Listed by impact, largest first.

### 3.1. `--coverage` instrumentation (5–10× slower per test)

Coverage rewrites every source file to add line/branch counters before Jest executes. The local baseline ran without coverage. Realistic per-test overhead with Istanbul/V8 coverage in a Next.js + ts-jest pipeline is **5–10×**, especially for component tests that traverse many imports.

### 3.2. CI worker count: 1 vs. 7-15 locally (7–15× wall-clock)

Jest defaults to `os.cpus().length - 1` workers.

- **Local** (8-core dev machine): ~7 workers. With `npx jest --no-coverage` we observed all suites finishing in 92 s.
- **CI** (`ubuntu-latest`, 2 vCPU): **1 worker**. Suites run effectively serially.

### 3.3. `--changedSince=origin/develop` does not actually shrink the run for this epic

`--changedSince` runs every test that *transitively* imports a changed file. The MLID-2011 epic touches:

- `Notification` model + service (`apps/web/models/Notification.ts`, `apps/web/services/mongodb/notification.ts`)
- `SignalR` service (`apps/web/services/signalr/index.ts`)
- Order tracker components + hooks (`apps/web/app/orders-tracker/...`)
- `useDocumentsTabNotifications` hook
- Audit log config (`apps/web/services/mongodb/auditLog.config.ts`, `auditLog.ts`)
- Feature flag enum (`apps/web/utils/featureFlags/featureFlags.ts`)

These are imported across most of the codebase. So `--changedSince` is effectively running close to the full suite anyway, but with the coverage tax on top.

### 3.4. No sharding on the PR workflow

`unit-tests-full.yml` shards 4-way across 4 parallel runners (lines 71-73). `unit-tests.yml` does **not** — the entire `--changedSince` set runs on a single 2-core runner. That alone is a missed 4× speedup.

### 3.5. V8 fatal-error retries on one suite

`apps/web/app/api/jobs/orders-tracker/export/route.test.ts` triggers:

```
Fatal JavaScript invalid size error 174895934 (see crbug.com/1201626)
Jest worker encountered 4 child process exceptions, exceeding retry limit
```

Confirmed pre-existing on `develop` (per epic plan-progress note). With coverage on, the heavier memory footprint trips the OOM earlier and harder. Each retry burns wall-clock before Jest gives up.

### Multiplier stack (revised with measured numbers)

| Factor                              | Multiplier (measured)               |
|-------------------------------------|-------------------------------------|
| `--coverage`                        | **~3.2×** (measured locally, §1.1)  |
| 1 worker vs. 7+ locally + 2 vCPU runner constraint | **~28×** (after coverage normalized, §1.1) |
| No PR-side sharding                 | (4× recoverable in wall-clock)      |
| V8 crash retries on one suite       | adds dead time, included in 28× above |

Combined: **3.2 × 28 ≈ 90×** of capacity, observed wall-clock 54× gap (92 s no-coverage local → 83 min CI). The shortfall vs. capacity (90× vs. 54×) is consistent with the CI run also benefiting from `--changedSince` filtering out 40% of suites — the CI doesn't pay the no-coverage cost on the 340 filtered-out suites that the local 92 s baseline did pay.

---

## 4. Recommended changes (highest leverage first)

| #  | Change                                                                                                                                                                                       | Estimated impact (measured) | Effort  |
|----|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------|---------|
| 1  | **Add 4-way matrix sharding to `unit-tests.yml`**, mirroring the `unit-tests-full.yml` strategy block (`strategy.matrix.shard: [1, 2, 3, 4]` + `--shard=${{ matrix.shard }}/4`).             | **~4× wall-clock speedup**   | 5 lines |
| 2  | **Drop `--coverage` from `unit-tests.yml` (PR check).** Coverage is already enforced by `unit-tests-full.yml` on `develop` push. Move the per-file 80% threshold guard to a smaller dedicated job, or accept that it lives at the merge gate. | **~3× per-test speedup** (measured) | 1 line  |
| 3  | **Use a larger runner** (`runs-on: ubuntu-latest-4-core` or the org's `-large` SKU if available). 4-8 vCPU instead of 2 → more Jest workers per shard.                                       | 2-3×                         | 1 line  |
| 4  | **Fix the V8 crash on `app/api/jobs/orders-tracker/export/route.test.ts`.** Run alone with `--detectOpenHandles --logHeapUsage` to find the leak. Stabilizes shards and removes retry overhead. | -1-2 min absolute            | medium  |
| 5  | **Investigate `services/apiKeys/*.test.ts` cluster** (6 suites, ~120 s combined locally, much worse with coverage). Suspect each file spins up its own Mongo connection — consolidate into a shared `beforeAll`. | -1-2 min absolute            | medium  |

Updated order based on the §1.1 measurement: **#1 first** (sharding now ranks ahead of coverage-removal — the measured coverage cost is ~3×, not the earlier 5-10× estimate, while the runner-constraint factor is the dominant ~28× and is what sharding directly attacks). **#2 next** for the additional 3× per-shard speedup. Combined #1+#2 should land the PR check in **single-digit minutes** instead of 83.

---

## 5. Known pre-existing failure (not caused by epic)

```
FAIL node app/api/jobs/orders-tracker/export/route.test.ts
  ● Test suite failed to run
    Jest worker encountered 4 child process exceptions, exceeding retry limit
```

V8 fatal "invalid size error 174895934" — heap allocation failure documented in [crbug.com/1201626](https://crbug.com/1201626). Reproduces on plain `develop` (confirmed in MLID-2011 plan-progress). Not introduced by this epic. Should be tracked separately — see Recommendation #4.

---

## 6. Reproducing this analysis

```bash
# Local baseline (no coverage)
cd apps/web
npx jest --no-coverage

# Top-N slowest suites
npx jest --no-coverage 2>&1 | grep -E "PASS .* \([0-9]+\.[0-9]+ s\)" | sort -t '(' -k2 -rn | head -20

# Simulate CI worker constraint locally
npx jest --no-coverage --maxWorkers=1

# Simulate full CI flag set locally
npx jest --ci --coverage --maxWorkers=1
```

The last command is the closest local proxy for what the CI actually does on a single shard.
