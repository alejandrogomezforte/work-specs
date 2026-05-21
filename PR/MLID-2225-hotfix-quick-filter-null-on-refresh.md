# [MLID-2225] Hotfix: Prevent 500 on refresh when "Active Orders by Due Date" quick filter was applied

## Summary

- Detected locally on 2026-05-18: applying the **Active Orders by Due Date** quick filter on `/orders-tracker/new`, navigating away (e.g. to `/intakes`), navigating back, and hard-refreshing the browser caused a 500 from `/api/orders/new`. The table rendered an empty "No rows" state instead of the filtered rows.
- Root cause is a three-defect alignment surfaced by the MLID-2225 filter persistence layer; this hotfix closes two layers, the third is tracked separately in **[MLID-2409](https://localinfusion.atlassian.net/browse/MLID-2409)**.
- After this hotfix, the cold-mount refresh path returns 200 and matches the post-click table state.

**Branch:** `hotfix/MLID-2225-quick-filter-null-on-refresh` → `develop`

**Parent Jira:** MLID-2225 — original Saving Filters & Column Ordering story already on develop; this is a hotfix on top of that work, tracked under the same Jira ID. No separate ticket filed for this fix.

**Follow-up Jira (separate change):** [MLID-2409](https://localinfusion.atlassian.net/browse/MLID-2409) — defer the orders-tracker initial fetch until feature-flag-gated columns settle (architectural fix; this hotfix is the immediate user-visible repair).

---

## Why the bug only triggered on refresh, not on first click

The same `null` reaches the server in both flows. Only the cold-mount flow routes it through the regex code path that can't handle it.

```
First click (page open long enough for flags to resolve):
  ORDERS_TRACKER_IG_COLUMNS = true
    → columns include { field: 'status', type: 'singleSelect' }
    → fieldTypes payload includes status: 'singleSelect'
    → server picks the singleSelect branch in gridFilterToMongoQuery
    → builds { status: { $in: [null, "Not Started", ...] } }
    → valid MongoDB, no regex involved
    → 200 OK

Hard refresh (cold mount, flags still loading):
  ORDERS_TRACKER_IG_COLUMNS = false (initial value)
    → columns exclude the IG-flag block (no status, no dueDate, ...)
    → fieldTypes payload omits status entirely
    → server fallback: t = fieldTypes['status'] ?? 'string'
    → server picks the string branch
    → coerced.map(escapeRegex) → escapeRegex(null) → 💥 500
```

The intermediate navigate-away step in the repro is just the simplest path that persists the bad filter to localStorage *before* the cold reload. A bare F5 immediately after clicking the chip the first time reproduces the same crash.

---

## Changes

| File | Change |
|------|--------|
| `apps/web/app/orders-tracker/[category]/page.tsx` | Quick filter `'activeOrdersByDueDate'` uses `NO_VALUE_SENTINEL` instead of `undefined` to seed the "no status" bucket. `JSON.stringify` of an array containing `undefined` substitutes literal `null` by spec; the MLID-2225 persistence layer then preserved that `null` in URL+localStorage. The sentinel survives JSON faithfully, is already the codebase-wide primitive for "no value" (see `withNoValueOption`), and is already handled by `gridFilterToMongoQuery` via the `hasSentinel` / `emptyPredicate` path — preserving user intent. |
| `apps/web/app/orders-tracker/[category]/page.test.tsx` | +2 tests in a new "Quick filter apply emits a JSON-round-trip-safe filter value" describe block. One asserts the emitted `filterModel.items[0].value[0]` is `NO_VALUE_SENTINEL` (and the array contains no `undefined`). The other does a JSON round-trip on the emitted filter model and asserts the value array contains no `null` — the test that maps 1:1 to the user-visible bug. |
| `apps/web/utils/api/orders/gridToMongoQuery.ts` | The `coerced` filter in the `isAnyOf` branch now drops both `undefined` *and* `null`. Pre-fix, `coerced.map(escapeRegex)` in the string branch called `escapeRegex(null)` → `null.replace(...)` → `TypeError` → 500. This is a defense-in-depth change: any future null reaching the server (legacy saved filter, hand-edited URL, unrelated race) is now a silent no-op rather than a crash. A short comment cross-references the postmortem. |
| `apps/web/utils/api/orders/gridToMongoQuery.test.ts` | +4 tests: (a) string `isAnyOf` with `null` in the value array does not throw, (b) string `isAnyOf` with `[null, "Approved"]` produces the same predicate as `["Approved"]`, (c) singleSelect `isAnyOf` with `null` drops it from the `$in`, (d) string `isAnyOf` with only `null` returns no stages (no-op). |

### NOT touched

- **`useGridUrlModels` / `useUrlParams`** — the filter persistence machinery is working as designed. The defect was the value the persistence layer was asked to persist (`undefined`), not the persistence layer itself.
- **The columns-vs-fieldTypes race in `Orders.tsx`** — broader architectural fix, tracked in MLID-2409. Three options are listed there for design discussion: (a) gate the initial fetch on a "feature flag context settled" boolean, (b) compute `fieldTypes` from a static field registry, or (c) have the server own the schema entirely. This hotfix only closes the immediate crash surface.
- **`coerceValue` itself** — its `if (value === null) return null;` short-circuit is unchanged; the new filter at the call site is the cleaner place to enforce the post-coercion contract, since other callers may legitimately want to coerce `null` for other operators (`is null`, etc., not used today).
- **The `taskUnread` / `userUnread` notification work and other recently-merged hotfixes** — out of scope.

---

## Verification

| Check | Result |
|-------|--------|
| `npx jest utils/api/orders/gridToMongoQuery.test.ts` | 25/25 passed (+4 new) |
| `npx jest 'app/orders-tracker/[category]/page.test.tsx'` | 56/56 passed (+2 new) |
| `npm run types:check` (apps/web) | Clean |
| `npx eslint` on 4 changed files | Clean |
| `npx prettier --check` on 4 changed files | Clean |
| Coverage on changed lines | All changed lines exercised by the new tests. Pre-existing file-level coverage of `gridToMongoQuery.ts` (75%) unchanged by this hotfix — uncovered branches are unrelated date/number/dateTime operators, out of scope. |

### RED-before-GREEN evidence

Server side, pre-fix:

```
× string "isAnyOf" tolerates a null value in the array (drops it, does not throw)
  TypeError: Cannot read properties of null (reading 'replace')
     at escapeRegex (utils/api/orders/gridToMongoQuery.ts:11:38)
     at Array.map (<anonymous>)
     at stringPredicate (utils/api/orders/gridToMongoQuery.ts:101:30)
     at gridFilterToMongoQuery (utils/api/orders/gridToMongoQuery.ts:367:20)
```

Client side, pre-fix:

```
× uses NO_VALUE_SENTINEL (not undefined) as the no-status marker so JSON round-trip is lossless
  Expected: "__LI_NO_VALUE__"
  Received: undefined

× survives JSON.stringify → JSON.parse without introducing null into the value array
  Expected value: not null
  Received array: [null, "Not Started", "Working this week", "Urgent",
                  "Blocked/Waiting", "Blocked by Payer", "Escalate to Sales"]
```

All 6 went green after the corresponding implementation step.

---

## Manual verification (already completed on local by the engineer before commit)

- [x] Clean dev session. Cleared `ordersTrackerLastQueryString-new` from localStorage. Clicked **Active Orders by Due Date** → 200, table shows filtered rows. Chip label still reads correctly (no regression — chip rendering already handles `NO_VALUE_SENTINEL`).
- [x] Same flow, then hard-refresh (F5). 200, table shows the same filtered rows. (Pre-fix this is the failing step.)
- [x] Navigated to `/intakes`, navigated back, hard-refreshed. 200, same rows.
- [x] Opened filter panel manually, added `Status is any of (No Value)`. 200, table shows only orders with no status.
- [x] Confirmed `Clear Filters` still clears the chip and re-fetches the unfiltered view.

---

## Test Plan (staging)

- [ ] On `/orders-tracker/new`, click the **Active Orders by Due Date** quick filter. Table updates with filtered rows; no console errors.
- [ ] Navigate to `/intakes`, come back, hard-refresh the browser. The orders table loads with the same filtered rows (pre-fix this errors out with `No rows` + 500).
- [ ] Manually compose a `Status is any of (No Value)` filter through the filter panel; verify it persists across refresh.
- [ ] Sanity: log in as a user *without* the `ORDERS_TRACKER_IG_COLUMNS` feature flag — the quick filter button should be hidden as before. No regression to that surface.
- [ ] Sanity: clearing all filters works and the URL/localStorage entry resets.
- [ ] Regression: other quick filter chips (`RTS`, `Orders Updated Last 7 Days`) still apply and persist correctly.
