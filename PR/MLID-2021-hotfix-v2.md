# [MLID-2021] Fix: dev:web includes @repo/ui via Turbo filter

## Summary

- Reverts the previous approach (`dependsOn: ["^build"]` on the `dev` task in `turbo.json`) which was rejected because it affected all `turbo dev` invocations globally.
- Uses Turborepo's dependency filter syntax (`--filter=@repo/web...`) in the `dev:web` script so that Turbo automatically includes transitive dependencies like `@repo/ui` when running `npm run dev:web`.
- `@repo/ui` runs `tsup --watch` alongside Next.js, keeping the dist folder built and up-to-date during development.

## Test plan

- [x] Delete `packages/ui/dist/` and run `npm run dev:web` — Turbo starts `@repo/ui` dev (tsup --watch) and `@repo/web` dev (Next.js) together
- [x] Verify no "Module not found" errors for `@repo/ui`
- [x] Verify `packages/ui/dist/` is rebuilt automatically
- [ ] Confirm `npm run dev` (all apps) still works as before
