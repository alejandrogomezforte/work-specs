# [MLID-364] chore(next-upgrade): migrate LISA (apps/web) to Next.js 16 / React 19

## Summary

This PR migrates the LISA web application (`apps/web`) from **Next.js 14 / React 18** to **Next.js 16 / React 19**, along with every dependency and code change that upgrade requires. It is the deliverable of the whole MLID-364 epic and lands as a single epic to `develop`.

The end state matches the team's normal green bar: `npm run build:web` clean, `npm run dev:web` clean, the full test suite green, lint passing with zero errors, and types checking clean. The two hard third-party blockers (authentication and masked inputs) are resolved, and three production runtime bugs surfaced by React 19 are fixed.

**Epic branch:** `epic/MLID-364-nextjs-v16-migration` -> `develop`

**Size:** 14 commits · 417 files changed (+4,525 / -7,032) · 176 of those are test files.

---

## Why

Next.js 14 and React 18 are two major versions behind. Staying on them blocks access to newer framework features, security patches, and the React 19 ecosystem, and the gap grows more expensive to close over time. This epic closes that gap in one controlled migration, built on a completed proof of concept that already proved the migration is technically reachable (clean build and clean dev runtime, both browser-verified).

---

## What changed and why

### 1. Core framework upgrade (Next 16 + React 19)

- `next` `14.2.33` -> `16.2.9`, `react` / `react-dom` `18` -> `19.2.7`.
- `next.config.mjs` cleaned of every removed or renamed key.
- `middleware` renamed to `proxy` (Next 16 rename).
- The build and dev commands pin `--webpack` to keep the existing custom `webpack()` config (IgnorePlugin / NormalModuleReplacementPlugin / externals for the SFTP client, Application Insights, and Node built-ins). Moving to Turbopack is deliberately deferred to a separate ticket (see Out of scope).
- All 64 React 19 / Next 16 build and type errors were fixed.

### 2. Authentication: NextAuth v4 -> Auth.js v5

The single largest change. Next 16's async request APIs (`cookies()`, `params`) are incompatible with NextAuth v4, so authentication was migrated to **Auth.js v5** (`next-auth@5.0.0-beta.31`, pinned exact).

- All server-side session reads moved from `getServerSession(authOptions)` to `auth()` from `@/auth` (App Router `auth()`, Pages Router `auth(req, res)`, and the proxy auth check).
- The NextAuth callbacks were extracted into their own module so they can be unit-tested directly.
- The client API is unchanged — Auth.js v5 keeps `next-auth/react` (`useSession`), so client components were not touched.
- Blast radius: roughly 336 files.

### 3. React-19-incompatible dependency swaps

- **`react-input-mask` -> `@mona-health/react-input-mask@3.0.3`** — the original package uses `findDOMNode`, removed in React 19. The fork removes it. This was not a pure drop-in (children and ref API changed), so the masked-input call sites were adjusted.
- **`react-simple-toasts` `6.0.0` -> `^6.1.0`** — 6.0.0 calls the removed `ReactDOM.render`, which crashed **every toast app-wide under React 19**. 6.1.0 detects React 18+ and uses `createRoot`. This is a required production fix, not a test-only change (see Runtime fixes below).
- **`react-select` `5.8.0` -> `5.10.2`** — React 19 type collapse.

### 4. Async request APIs

- The async-`params` transform was applied across all dynamic route files (`[param]/route.ts`, `[param]/page.tsx`): `params` is now a `Promise` and is awaited (server) or read with React 19 `use()` (client pages).

### 5. Production runtime fixes found during verification

These are real application bugs, not test scaffolding:

- **`react-simple-toasts` crash** — described above; repaired app-wide toasts under React 19.
- **`useFetchData` render loop** — consumers passing an inline callback received an unstable `customFetch` / `refetch`, which, when listed in an effect dependency array (for example on `PatientProvider`-backed routes), re-fired the effect every render and looped ("Maximum update depth exceeded"). Fixed by holding the callback in a ref so the memoized functions keep stable identities. A dedicated test covers the stable-identity guarantee. This is a latent bug shared with `develop`; it is resolved here.

### 6. Lint: ESLint 9 flat config (LISA scope)

- `apps/web`, `packages/ui`, `packages/eslint-config`, `apps/kiosk`, and the root moved to ESLint 9 flat config. Configuration only — no source files were changed for the lint migration.

### 7. Test suite (closing the TDD exception)

The proof of concept used the documented spike exception to skip tests. This PR meets the normal bar. Work included:

- Migrating roughly 156 test files from the NextAuth v4 `getServerSession` mock to a global Auth.js v5 `auth()` mock (`moduleNameMapper` maps `@/auth` and `next-auth/react` to shared mocks, so no test loads the ESM-only real modules).
- Restructuring the async-`params` page tests for React 19 `use()` via a `fulfilledParams()` helper (reads a fulfilled thenable synchronously, so existing assertions keep working).
- Updating the masked-input test mocks to the `@mona-health` fork.
- Fixing assorted React 19 behavioral changes in component tests (component-call argument change, effect-flush timing, DOM-nesting message change).

---

## Dependency changes

| Package | Before | After | Reason |
|---|---|---|---|
| `next` | `14.2.33` | `16.2.9` | Core upgrade |
| `react` / `react-dom` | `18` | `19.2.7` | Core upgrade |
| `next-auth` | `4.24.7` | `5.0.0-beta.31` (pinned exact) | Async request API compatibility |
| `react-input-mask` | `2.0.4` | removed | `findDOMNode` removed in React 19 |
| `@mona-health/react-input-mask` | — | `3.0.3` (added) | Drop-in-ish replacement without `findDOMNode` |
| `react-simple-toasts` | `6.0.0` | `^6.1.0` | 6.0.0 crashed under React 19 (`ReactDOM.render`) |
| `react-select` | `5.8.0` | `5.10.2` | React 19 type collapse |

---

## Operational / deployment notes (please confirm before merge)

1. **Beta dependency.** `next-auth@5.0.0-beta.31` is beta only because stable Auth.js v5 has not shipped — a timeline outside our control. It is pinned exact and can be swapped to stable when it exists. Risk accepted rather than waiting.

2. **One-time forced re-login on deploy.** Auth.js v5 renames the session cookie (`next-auth.*` -> `authjs.*`). Every active session is invalidated once when this deploys — all users are logged out and must sign in again. No data impact; worth a heads-up to the team so it is not read as an incident.

3. **Landing strategy.** This is a single large epic PR. If the team prefers a staged rollout to reduce merge-conflict / freeze pressure, that is a velocity decision to confirm before merge; the branch is compatible with either.

---

## Out of scope (split into their own tickets)

These were deliberately separated to keep this epic focused. None is a blocker (the build is green with the current choices):

- **MLID-2740** — webpack -> Turbopack migration. The custom `webpack()` config must be re-expressed in Turbopack's model.
- **MLID-2741** — `pages/api/**` test-route smell. `pageExtensions: ['ts','tsx']` makes `.test.ts` / `.spec.ts` files under `pages/` build as live routes. Pre-existing; only surfaced by the stricter v5 ESM build.
- **MLID-2742** — replace the `@mona-health/react-input-mask` fork with an in-house `@repo/ui` masked-input component, so production does not depend on a third-party fork. May gate the release or land as a fast-follow (PE / PO call).

---

## Known pre-existing issues (not introduced by this PR)

- **V8 worker crash** in `jobs/analytics-process-intake-overview/export/route.test.ts` — fails to *run* under the jest-on-V8 setup, not an assertion failure. Present on `develop`; unrelated to the migration.
- **Feature-flags 401 in the PHI-masking path** — `getAllFeatureFlags` makes a cookie-less internal fetch to the session-gated feature-flags endpoint and gets 401; the error is caught and masking falls back to defaults (request still returns 200). Byte-identical to `develop`; a candidate separate bug ticket.

---

## Verification status (on epic HEAD)

| Gate | Result |
|---|---|
| `npm run build:web` | GREEN — exit 0, all tasks successful |
| Full test suite (`jest --ci`, both projects) | ~19,046 pass; only the pre-existing V8 worker crash fails-to-run |
| `tsc --noEmit` (whole app) | Clean |
| `apps/web` lint | 0 errors / 250 warnings (warnings are the pre-existing backlog, out of scope per the agreed lint decision) |
| Storybook | Builds clean on React 19 |
| `develop` currency | Epic is 0 commits behind `develop` at the time of writing |

**CI note for reviewers:** the PR unit-test workflow runs `jest --changedSince=origin/develop --shard=N/4`. Because this epic changes so much, the changed-graph selection is large (roughly 352 of ~1,264 test files) — expect a heavier-than-usual CI run. Every selected suite passes locally.

---

## Test plan (runtime verification)

Manual browser pass on the migrated app:

- [ ] Google login works end-to-end (note the one-time forced re-login after deploy).
- [ ] Masked / phone inputs render and accept input correctly (the `@mona-health` fork).
- [ ] Trigger an action that pops a toast (for example a save) — toast renders, no crash.
- [ ] Navigate auth-gated routes (`/intakes`, `/admin`, `/orders-tracker`) — no 401s, no console/runtime errors.
- [ ] Open a patient details route (`/patient/[id]/details`) — no "Maximum update depth exceeded" loop.
- [ ] Spot-check `react-pdf` rendering, and `react-modal` / `react-collapsible` at runtime.
- [ ] Confirm dynamic routes with async `params` resolve correctly.

---

## Commits

| Hash | Message |
|---|---|
| `fc256252b` | [MLID-364] - chore(next-upgrade): PoC S1-S3 — Next 16 + React 19 core upgrade, middleware->proxy, async params |
| `5a3b424e4` | [MLID-364] - fix(next-upgrade): PoC S4-S6b checkpoint — Next 16 build GREEN (React 19 type fixes, react-select bump) |
| `cd5700508` | [MLID-364] - chore(lint): migrate LISA (web + ui + kiosk + shared) to ESLint 9 flat config |
| `41e54b08b` | [MLID-364] - feat(auth): migrate NextAuth v4 -> Auth.js v5 for Next 16 (PoC S7a) |
| `d2cff3bf4` | [MLID-364] - fix(inputs): replace react-input-mask with @mona-health fork for React 19 (PoC S7b) |
| `7eca4504e` | [MLID-364] - fix(hooks): stabilize useFetchData callbacks to stop React 19 render loop |
| `6f101fb56` | [MLID-364] - test(auth): migrate test suite auth mocks to Auth.js v5 |
| `d0d92007b` | [MLID-364] - test(pages): fix React 19 async-params page tests + global next-auth/react mock |
| `d12040c9d` | [MLID-364] - fix(react19): repair toasts under React 19 + fix the CI-scoped jsdom test failures |
| `a85749584` | [MLID-364] - chore(sync): merge develop into Next 16 migration branch |
| `c87f4084d` | [MLID-364] - chore(sync): merge develop into Next 16 migration branch (sync 2) |
| `f0202ab8f` | Merge branch 'develop' into epic/MLID-364-nextjs-v16-migration (sync 3) |
| `6a0522fc6` | [MLID-364] - chore(gitignore): ignore storybook-static build output |
| `d6d9e3500` | Merge branch 'develop' into epic/MLID-364-nextjs-v16-migration (sync 4) |

---

## Jira

- [MLID-364](https://localinfusion.atlassian.net/browse/MLID-364)
- Related follow-ups: [MLID-2740](https://localinfusion.atlassian.net/browse/MLID-2740), [MLID-2741](https://localinfusion.atlassian.net/browse/MLID-2741), [MLID-2742](https://localinfusion.atlassian.net/browse/MLID-2742)
