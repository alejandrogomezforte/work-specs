# [MLID-2021] Fix: dev task missing @repo/ui build dependency

## Summary

- After PR #946 merged `@localinfusion/ui` into the monorepo as `@repo/ui`, running `npm run dev:web` fails with `Module not found: Can't resolve '@repo/ui'` because the `dev` task in `turbo.json` was not configured to build workspace dependencies first.
- Added `dependsOn: ["^build"]` to the `dev` task, aligning it with `build`, `lint`, and `test` tasks that already have this dependency.

## Test plan

- [ ] Delete `packages/ui/dist/` and run `npm run dev:web` — Turbo should build `@repo/ui` automatically before starting Next.js
- [ ] Verify no "Module not found" errors for `@repo/ui`
