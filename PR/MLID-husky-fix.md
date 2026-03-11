# PR: Fix Husky v9 Compatibility

**Branch:** `fix/NO-MLID-fixing-husky-hooks-v9` → `develop`

## Summary

- Fix `prepare` script to use Husky v9 command (`husky`) instead of deprecated v4/v8 command (`husky install`)
- Remove legacy `.husky/_/husky.sh` sourcing and `HUSKY_SUBEXEC_MODULE` re-exec trick from hook scripts
- Clean up dead code and commented-out blocks in `.husky/pre-commit`

## Problem

The repo has **Husky v9** (`9.0.11`) installed, but the hooks were configured using the **v4/v8 pattern**:

1. **`prepare` script uses deprecated command** — `"prepare": "husky install"` is a no-op in v9. It prints a deprecation warning and does nothing. This means:
   - **New developers** cloning the repo and running `npm install` get **no git hooks at all** — no lint-staged, no JIRA ticket validation, no pre-push checks. Zero warnings.
   - Hooks silently don't exist and commits go through unvalidated.

2. **Old trampoline architecture** — The hooks relied on a `.husky/_/` runtime directory (gitignored, generated locally) that acted as a middleman:
   - Git called `.husky/_/pre-commit` → which sourced `.husky/_/h` → which delegated to `.husky/pre-commit` via `sh -e`
   - `.husky/pre-commit` then re-executed itself in `bash` via a `HUSKY_SUBEXEC_MODULE` environment variable trick
   - This caused **lint-staged to run twice** on every commit (double prettier formatting, double output)
   - The `sh -e` flag could also swallow `exit 0` statements if any intermediate command returned non-zero

3. **Version mismatch origin** — Husky v9 was added in commit `97df5825` (July 2024) with the v4/v8 `husky install` command, meaning the incompatibility existed from day one.

## Fix

| File | Change |
|------|--------|
| `package.json` | `"prepare": "husky install"` → `"prepare": "husky"` |
| `.husky/pre-commit` | Removed `husky.sh` sourcing, `HUSKY_SUBEXEC_MODULE` re-exec trick, and dead commented code. Now just runs `npx lint-staged` |
| `.husky/commit-msg` | Removed `husky.sh` sourcing. JIRA validation logic unchanged |

In Husky v9, the `husky` command (without `install`) correctly sets `core.hooksPath = .husky` in the local git config, pointing git directly to the hook files — no trampoline directory needed.

## Action Required for Existing Developers

After pulling this change, run:

```bash
npm install
```

This triggers the fixed `prepare` script which sets `core.hooksPath = .husky` automatically.

If hooks still misbehave, run manually:

```bash
npx husky
git config core.hooksPath .husky
```

Optionally clean up the legacy trampoline directory:

```bash
rm -rf .husky/_
```

## Test Plan

- [ ] Verify `npm install` on a fresh clone sets `core.hooksPath = .husky` (`git config core.hooksPath` should output `.husky`)
- [ ] Verify committing a `.ts` file runs lint-staged **once** (not twice)
- [ ] Verify committing without a JIRA ticket in message or branch name is rejected by commit-msg hook
- [ ] Verify committing with a JIRA ticket in branch name auto-prepends it to the message
- [ ] Verify merge commits skip JIRA validation
