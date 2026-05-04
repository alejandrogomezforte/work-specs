# [MLID-2263] - fix(shared): add dev script so dist is built when running npm run dev

## Summary

`@repo/shared` was introduced in MLID-1291 with `main` and `types` pointed at `./dist/index.js` / `./dist/index.d.ts`, but the package shipped without a `dev` script. Turbo's `dev` task does not chain off `^build`, so a fresh `npm run dev` never produced `packages/shared/dist/`, and every consumer (`@repo/web`, `li-public-api`) failed to resolve `@repo/shared` at startup.

This hotfix adds a `dev` script that runs `tsc --watch --preserveWatchOutput`, mirroring what `@repo/ui` already does. The dist artifact is now produced on dev startup and kept fresh for the lifetime of the session — no manual pre-build required.

**Hotfix branch:** `hotfix/MLID-2263/shared-dev-script` -> `develop`

**Jira ticket:** [MLID-2263](https://forteinfusion.atlassian.net/browse/MLID-2263)

---

## Root Cause

`packages/shared/package.json` declared a `build` script (`tsc`) but no `dev` script. Turbo's pipeline at `turbo.json` has:

```json
"dev": {
  "cache": false,
  "persistent": true
}
```

— no `dependsOn: ["^build"]`. So when `npm run dev` is run from the repo root, turbo starts every package's `dev` script in parallel and skips `@repo/shared` entirely (no `dev` script). Because `dist/` is git-ignored, on a fresh checkout it does not exist, and consumers that `require('@repo/shared')` blow up resolving the missing `dist/index.js`.

---

## Why Option B (package-local) and not Option C (turbo-wide)

Two repo-wide alternatives were considered and tested:

- **Option C — `dev` depends on `^build` in `turbo.json`.** Tested locally and confirmed it builds shared correctly, **but** it also forces `@repo/ui#build` to run alongside `@repo/ui#dev` (tsup --watch). Both tasks clean and write `packages/ui/dist/` simultaneously, producing an `ENOENT` race during `unlink`. Turbo aborts the whole run with exit 1. Unsafe for any package that ships both `build` and `dev` scripts.
- **Refined Option C — `dev` depends specifically on `@repo/shared#build`.** Avoids the `@repo/ui` race but couples turbo's pipeline to a single package by name. More invasive than necessary.
- **Option B — add `dev` to `@repo/shared`.** Single-line, package-local change. Symmetric with `@repo/ui`. No turbo.json change. Chosen.

---

## Changes by File

### `packages/shared/package.json`

- Added `"dev": "tsc --watch --preserveWatchOutput"` to `scripts`.

`--preserveWatchOutput` keeps prior compile output visible in turbo's TUI panes instead of clearing the screen on every rebuild, matching how `tsup --watch` behaves in `@repo/ui`.

---

## Files Changed

| File | Change | Lines |
|------|--------|:-----:|
| `packages/shared/package.json` | Add `dev` watch script | +1 |

**Total: 1 file, +1 insertion, 0 deletions**

---

## Test Plan

- [x] Reverted any local turbo.json edits.
- [x] Killed all stale node processes from prior dev attempts.
- [x] Wiped `packages/ui/dist/` and `packages/shared/dist/` to simulate a fresh checkout.
- [x] Ran `npm run dev` from repo root.
- [x] Verified `li-mcp-server`: `Found 0 errors. Watching for file changes.`
- [x] Verified `li-public-api`: `Found 0 errors. Watching for file changes.`
- [x] Verified `@repo/web`: `✓ Compiled /instrumentation in 10.8s` and `✓ Ready in 20.6s`.
- [x] Confirmed no `Cannot find module '@repo/shared'` errors anywhere in the dev output.
- [ ] Reviewer pulls the branch, deletes `packages/shared/dist/`, runs `npm run dev`, and confirms all apps boot without the resolution error.

> **Note:** Two unrelated runtime errors appeared during testing — `li-mcp-server` complained about a missing OAuth `clientId` env var, and `li-public-api` complained about an uninitialized MongoDB connection. Both are local-environment env-var gaps, not regressions introduced by this change. Both processes had already passed compilation and module resolution before crashing.
