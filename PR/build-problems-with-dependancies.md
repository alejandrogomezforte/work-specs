# CI/CD Build Failure: Dependency Conflict Analysis

## Date: 2026-01-28

## Problem

The CI/CD pipeline fails during `npm install` with a peer dependency conflict:

```
eslint-config-next@16.1.1 requires eslint >=9.0.0,
but package.json specifies eslint@8.56.0
```

## Root Cause

Two separate Dependabot PRs were merged into `develop` that are incompatible together:

1. **`aa1fcb45` (Dec 26, 2025)**: Bumped `eslint-config-next` to `16.1.1`
2. **`28ff63b7` (Jan 28, 2026)**: Bumped `next` from `14.2.33` to `16.1.6`

Neither PR updated `eslint` from `8.56.0` to `>=9.0.0`, which `eslint-config-next@16.1.1` requires as a peer dependency.

### Why it worked before today

The `eslint-config-next@16.1.1` bump from December likely passed CI because the lockfile resolved the peer dependency without strict enforcement. Today's `next` bump to `16.1.6` regenerated the lockfile in a way that now surfaces the peer dependency error during `npm install`.

### Affected scope

This is a `develop` branch issue — **all open PRs** that merge from `develop` will be affected, not just a single PR.

## Current Dependency State

| Package              | Current Version | Required By                  |
| -------------------- | --------------- | ---------------------------- |
| `next`               | `16.1.6`        | Dependabot bump (was 14.2.33)|
| `eslint-config-next` | `16.1.1`        | Dependabot bump              |
| `eslint`             | `8.56.0`        | Not upgraded                 |
| ESLint config format | `.eslintrc.json` | ESLint 8 legacy format      |

## Recommendations

### Option 1: Revert Dependabot bumps (Recommended - lowest risk)

Revert both Dependabot commits on `develop` to restore the previous working state:

- Revert `next` back to `14.2.33`
- Revert `eslint-config-next` back to a 14.x-compatible version

**Why**: Next.js 14 to 16 is a major version upgrade that requires significant migration work (React 19, new App Router changes, breaking API changes). This should not be done via an unreviewed Dependabot PR.

```bash
# On develop branch
git revert 28ff63b7   # revert next 16.1.6 bump
git revert aa1fcb45   # revert eslint-config-next 16.1.1 bump
```

After reverting, pin versions in `package.json`:

```json
{
  "dependencies": {
    "next": "14.2.33"
  },
  "devDependencies": {
    "eslint-config-next": "14.2.33"
  }
}
```

### Option 2: Upgrade ESLint to 9+ (high risk, high effort)

Upgrade `eslint` to `>=9.0.0` and migrate the ESLint configuration:

- Update `eslint` from `8.56.0` to `^9.0.0`
- Migrate `.eslintrc.json` to new flat config format (`eslint.config.mjs`)
- Update all ESLint plugins for v9 compatibility
- Fix any new lint errors introduced by the upgrade

**Why not**: ESLint 8 to 9 is a major migration with breaking changes (flat config, removed formatters, rule changes). Combined with a Next.js 14 to 16 jump, this introduces too much risk for a CI fix.

### Option 3: Pin eslint-config-next to match Next.js (quick fix)

If reverting is not desirable, pin `eslint-config-next` to match the installed Next.js version:

```json
{
  "devDependencies": {
    "eslint-config-next": "14.2.33"
  }
}
```

This resolves the peer dependency conflict without touching `next` or `eslint`. However, `next@16.1.6` would still be in the project, which may cause other issues since the codebase was built for Next.js 14.

## Preventive Measures

1. **Configure Dependabot to ignore major version bumps** for critical packages (`next`, `eslint`, `react`). Add to `.github/dependabot.yml`:

   ```yaml
   updates:
     - package-ecosystem: "npm"
       directory: "/"
       schedule:
         interval: "weekly"
       ignore:
         - dependency-name: "next"
           update-types: ["version-update:semver-major"]
         - dependency-name: "eslint"
           update-types: ["version-update:semver-major"]
         - dependency-name: "eslint-config-next"
           update-types: ["version-update:semver-major"]
         - dependency-name: "react"
           update-types: ["version-update:semver-major"]
         - dependency-name: "react-dom"
           update-types: ["version-update:semver-major"]
   ```

2. **Require CI checks to pass** before merging Dependabot PRs (ensure branch protection rules enforce this).

3. **Review Dependabot PRs manually** for packages in the core framework stack rather than auto-merging.
