# [MLID-2268] fix(test): replace auto-mock with factory in orders-tracker export route test

## Summary

Removes a long-standing perpetually-failing Jest suite from the `apps/web` test run.

`apps/web/app/api/jobs/orders-tracker/export/route.test.ts` was failing on every Jest run — locally and in CI — with a fatal V8 error during module load, before any test code executed. Root cause: a single line of test setup using Jest's *auto-mock* pattern (no factory function) on `@/services/jobs/scheduler`, which forced Jest to load the real scheduler module and walk its entire export graph (the full Pulse job-definition tree). That walk pushed past V8's hardcoded internal allocation limit at ~167 MB.

The fix replaces the bare `jest.mock(...)` call with a one-line factory mock — matching the pattern the same file already uses for `@/utils/logger`. No behavioral change to the test assertions; all 6 tests pass in ~12 s.

## Source / Target

- **Branch:** `fix/MLID-2268-jest-automock-crash` → `develop`
- **Files changed:** 1 (`apps/web/app/api/jobs/orders-tracker/export/route.test.ts`)
- **Diff:** +3 / -1
- **Commit:** `ffe89059`

## Before vs. After

### Before (failing on every run)

```
PASS jsdom ...
PASS node  ...
FAIL node  app/api/jobs/orders-tracker/export/route.test.ts
  ● Test suite failed to run
    Jest worker encountered 4 child process exceptions, exceeding retry limit

#
# Fatal error in , line 0
# Fatal JavaScript invalid size error 174895934 (see crbug.com/1201626)
#
```

### After

```
PASS node app/api/jobs/orders-tracker/export/route.test.ts (12.24 s)
  Orders Tracker Export API Route
    POST /api/jobs/orders-tracker/export
      ✓ schedules NEW orders export with default params (41 ms)
      ✓ schedules MAINTENANCE orders export with dateFilter and maxRecords (8 ms)
      ✓ allows null dateFilter for full export (7 ms)
      ✓ returns 400 for invalid dateFilter format (13 ms)
      ✓ returns 400 for invalid maxRecords (10 ms)
      ✓ returns 500 when scheduling throws (7 ms)

Test Suites: 1 passed, 1 total
Tests:       6 passed, 6 total
```

## What changed

`apps/web/app/api/jobs/orders-tracker/export/route.test.ts:9`

```diff
- jest.mock('@/services/jobs/scheduler');
+ jest.mock('@/services/jobs/scheduler', () => ({
+   scheduleImmediateJob: jest.fn(),
+ }));
```

That's the entire change.

## Why this fix works

`jest.mock(modulePath)` with no factory triggers **automatic mocking**: Jest loads the real module, introspects every exported value, and synthesizes a deep mock proxy mirroring the shape of the original. For `@/services/jobs/scheduler` this is catastrophic because the scheduler module is the entry point that pulls in the whole Pulse job graph — every job definition (orders-tracker-export, jotForm, weinfuseUploader, abbvie, claims, etc.). Walking that graph and constructing the mock proxy hits V8's hardcoded internal allocation limit at 174,895,934 bytes (~167 MB), reported as `Fatal JavaScript invalid size error 174895934 (see crbug.com/1201626)`.

`jest.mock(modulePath, factory)` skips the auto-mock walk entirely and uses the factory's return value as the module. The original module is never loaded for introspection. The factory used here exposes only the single function the test actually calls (`scheduleImmediateJob`), which is also the only export referenced via the imported `mockScheduleImmediateJob` reference inside the test bodies.

## Bisection evidence

The cause was identified by isolating each piece of the import chain in a temp test file:

| Imports in temp test | `jest.mock(...)` calls | Result |
|---|---|---|
| `next/server` | none | PASS (1 s) |
| `exportOrdersTrackerToS3` (transitive heavy module) | none | PASS (7 s) |
| `exportOrdersTrackerToS3` + `next/server` | none | PASS (20 s) |
| All of route.ts's imports manually via `require()` | none | PASS (7 s) |
| `require('.../route')` | none | PASS (7 s) |
| `require('.../route')` | **`jest.mock('@/services/jobs/scheduler')` (auto-mock)** | **CRASH** ← |
| `require('.../route')` | `jest.mock('@/services/jobs/scheduler', () => ({ scheduleImmediateJob: jest.fn() }))` | PASS (7 s) |

The crash is fully attributable to the bare `jest.mock(...)` call. It is **deterministic**, not memory-pressure-dependent — `--max-old-space-size=4096` (CI default) does not help because the limit is a hardcoded V8 internal check, not a heap budget.

## Why the second mock in the same file was already correct

Right below the broken line, the same file mocks the logger with a factory:

```ts
jest.mock('@/utils/logger', () => ({
  logger: { info: jest.fn(), warn: jest.fn(), error: jest.fn() },
}));
```

This is the pattern the scheduler mock should also follow — and now does. Whoever wrote the file likely knew the risk for `logger` (it's another widely-imported entry point) but missed it for `scheduler`.

## Behavioral impact on the test

None. The test bodies access the mock through:

```ts
const mockScheduleImmediateJob = scheduleImmediateJob as jest.MockedFunction<
  typeof scheduleImmediateJob
>;
```

That works identically with both auto-mock and factory-mock — the imported `scheduleImmediateJob` reference still points at a `jest.fn()`, just one created by the factory rather than by Jest's introspection walk. All 6 tests' calls to `mockScheduleImmediateJob.mockResolvedValue(...)` and `mockScheduleImmediateJob.mockRejectedValue(...)` still operate on a real Jest mock function.

## CI impact

- One perpetually-failing suite is removed from every PR and develop push.
- Jest's worker retry loop (4 child-process-exception attempts before giving up) no longer burns wall-clock on this file. Small absolute savings (~30-90 s per CI run depending on how long each retry attempt takes before crashing) but stabilizes the suite.
- Coverage for `app/api/jobs/orders-tracker/export/route.ts` will now appear in the merged report instead of being missing.

## Test Plan

```bash
# From apps/web/

# 1. Confirm the failing suite now passes in isolation
npx jest "app/api/jobs/orders-tracker/export/route.test.ts" --no-coverage --runInBand
# Expect: 6/6 passing in ~12 s, no V8 crash

# 2. Confirm the full suite is green except for environment-specific issues
npx jest --no-coverage
# Expect: 867/867 suites pass (was 866/867 with this one failing)

# 3. Type check is unchanged
npx tsc --noEmit
```

## Out of scope

- Other test files using bare `jest.mock(...)` patterns are not audited or changed in this PR. If others trigger similar V8 crashes, file individually.
- The `--coverage` slowdown and PR-side sharding gaps identified during the same investigation are separate concerns — they're tracked elsewhere and not addressed here.

## Jira

[MLID-2268](https://localinfusion.atlassian.net/browse/MLID-2268) — V8 invalid-size crash in `apps/web/app/api/jobs/orders-tracker/export/route.test.ts` during Jest auto-mock
