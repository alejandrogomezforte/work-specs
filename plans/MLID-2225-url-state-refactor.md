# MLID-2225 — URL State Refactor (Sister Plan)

## Task Reference

- **Parent plan**: [`MLID-2225-saving-user-filters-and-column-ordering.md`](./MLID-2225-saving-user-filters-and-column-ordering.md)
- **Jira**: [MLID-2225](https://localinfusion.atlassian.net/browse/MLID-2225) (no separate ticket — same story, follow-up architectural cleanup)
- **Branch**: `feature/MLID-2225-url-state-refactor`
- **Branched from**: `feature/MLID-2225-persist-filters-column-order` at commit `dbeb1b8b`
- **Base for PR**: `feature/MLID-2225-persist-filters-column-order` (or directly `develop` — decide at PR time)
- **Status**: Awaiting approval of the step order and the three open questions below
- **Safety net**: original feature branch is untouched. If this refactor doesn't pan out, drop the sister branch and ship the original branch as-is.

---

## Why This Refactor Exists

The original MLID-2225 implementation introduced a second writer to the orders-tracker URL state: `usePersistedRouteQuery` does `router.replace` on mount to restore from localStorage, while `useGridUrlModels` is the existing URL writer. The two writers raced, which forced **Post-implementation Fix 2** in the parent plan — `liveStateFromUrl()` reading `window.location` directly inside `buildQs` to side-step a stale `stateRef`.

That fix works, but it's a band-aid for a layering issue. Reviewing the timeline of a fresh page load also surfaced:

1. **Empty-write race in `usePersistedRouteQuery` Effect 2** — on mount, the effect fires with React's stale empty `searchParams`, momentarily writing `''` to localStorage before the restored value lands on the next render. Self-heals, but anything reading storage in that window sees nothing.
2. **`quickSearchValue` never re-syncs from URL** — `useState(search)` only uses the initial render's value. If the restored URL has `search=foo`, the textbox shows empty and the mount-time `setSearch('')` flush actively strips the search param from the URL.
3. **N setters wrap a single `updateModels` call** — `setPaginationModel`, `setSortModel`, `setFilterModel`, `setSearch`, `setActiveQuickFilter` are all sugar over `updateModels({ field })`. The page already calls `updateModels` directly in several places (quick-filter chips, `removeFilterItem`).

The clean architecture: **one hook owns URL-as-source-of-truth + storage mirror + restore. The URL has exactly one writer. The schema (parse/serialize/defaults) is data passed in as config, not logic baked into the hook.**

This makes the same primitive reusable on intakes (Step 5 of the original plan), eliminates Fix 2's `liveStateFromUrl` band-aid, eliminates the empty-write race, and collapses the N-setter API to a single `update(overrides)` call.

---

## Target Architecture

```ts
// apps/web/utils/hooks/useUrlStateWithStorage.ts

interface UrlSchema<TState> {
  defaults: TState;
  parse: (params: URLSearchParams) => TState;
  serialize: (state: TState) => URLSearchParams;
}

interface Config<TState> extends UrlSchema<TState> {
  storageKey?: string; // optional persistence; undefined = URL only
}

interface UseUrlStateWithStorageReturn<TState> {
  state: TState;
  update: (overrides: Partial<TState>) => void;
  clear: () => void;
  /** Serialized current URL state, e.g. for tab caches */
  queryString: string;
}

export function useUrlStateWithStorage<TState>(
  config: Config<TState>
): UseUrlStateWithStorageReturn<TState>;
```

**Invariants:**

1. **One writer.** Only `update` and `clear` mutate the URL. The restore path on mount also routes through this writer (it just calls `router.replace` directly with the stored qs — no parse/serialize round-trip needed).
2. **Live reads.** Inside the microtask flush, the hook reads `window.location.search` to compute the next URL — never a React-cached `searchParams` snapshot. This is necessary because of Next.js App Router's deferred `router.replace`. The synchronous `window.history.replaceState` keeps `window.location` consistent with the writer's intent.
3. **Storage is a passive mirror.** Storage is written inside `update` (after computing the next URL), not as a separate effect that watches `searchParams`. This eliminates the empty-write race.
4. **Schema is data.** `parse`, `serialize`, and `defaults` are passed in as config. They must be stable references (module-scope constants) to avoid invalidating the hook's `useMemo`/`useCallback` dependencies.

---

## Restore Semantics (Locked)

**Decided 2026-05-08**: storage wins, with seed-from-URL fallback.

On mount (and on storage-key change, e.g. tab switch):

```
if (storage has a value):
    if (stored qs !== current URL qs):
        redirect URL to stored qs (history.replaceState + router.replace)
    // else: URL already matches, no redirect
else if (URL has params):
    seed storage with current URL qs
else:
    no-op (both empty)
```

After mount, every `update()` writes both the URL and storage in the same step, keeping them in sync.

**Consequence:** the orders-tracker `tabParamsCache` module-level object becomes redundant — per-tab storage keys already preserve each tab's last qs across switches. Step 3 deletes `tabParamsCache`.

**Trade-off (unchanged from parent plan Fix 1):** bookmarked or shared URLs with explicit filters get overwritten by the user's saved state. Accepted.

---

## Open Questions (Awaiting User Confirmation)

These answers gate the start of implementation:

1. **Pagination on intakes URL: 0-based?** Original plan said yes (matches orders tracker). Confirmed?
2. **`useIntakesTable` shape:** today returns ~15 fields (selectedLocations, selectedTypes, debouncedSearch, etc.) and owns API fetching + debouncing. The refactor swaps only the URL plumbing; the hook's external shape and fetching responsibility stay the same. Agreed?
3. **`tabParamsCache` deletion:** with per-tab storage keys, the in-session module cache is redundant. Step 3 deletes it. Agreed?

---

## Step Order

Each step is its own commit. TDD red-green-refactor inside each. Visual verification gates Steps 3 and 5 (UI changes per `feedback_visual_test_before_commit`).

### Step 1 — Build `useUrlStateWithStorage` primitive

**Files**:
- Create: `apps/web/utils/hooks/useUrlStateWithStorage.ts`
- Create: `apps/web/utils/hooks/useUrlStateWithStorage.test.ts`
- Modify: `apps/web/utils/hooks/index.ts`

**RED test cases** (`useUrlStateWithStorage.test.ts`):

- `"derives state from URL via parse on mount"` — searchParams `?foo=1`; assert `state` matches `parse({foo:'1'})`.
- `"update applies overrides on top of live URL and writes via router.replace + history.replaceState"` — render with `?a=1`; call `update({ b: 2 })` (where schema serializes both); advance microtask; assert `mockReplace` called with URL containing both, and `window.history.replaceState` called with same.
- `"update coalesces multiple calls in the same tick into one router.replace"` — call `update` three times synchronously; advance microtask; assert `mockReplace` called once.
- `"update reads window.location at flush time, not stale React state"` — render Render-1; mutate `window.location.search` externally between render and flush; call `update({ x: 'y' })`; assert flushed URL builds from the live window.location, not the React `searchParams` snapshot.
- `"update writes serialized qs to localStorage when storageKey is provided"` — call `update({ a: 1 })`; assert localStorage[storageKey] === serialized qs.
- `"update does not touch localStorage when storageKey is undefined"` — same setup without storageKey; assert localStorage untouched.
- `"update skips router.replace when computed qs equals current window.location.search"` — call `update({})`; assert no replace.
- `"restore on mount: storage hit + URL empty → redirects URL to stored qs"` — pre-seed storage `'a=1&b=2'`; render with empty URL; assert mockReplace + history.replaceState called once with `/path?a=1&b=2`.
- `"restore on mount: storage hit + URL different → redirects URL to stored qs (storage wins)"` — pre-seed storage `'a=1'`; render with URL `?a=2`; assert redirect to `/path?a=1`.
- `"restore on mount: storage hit + URL identical → no redirect"` — pre-seed storage `'a=1'`; render with URL `?a=1`; assert mockReplace NOT called.
- `"restore on mount: no storage + URL has params → seeds storage from URL"` — render with URL `?a=1`; storage empty; assert localStorage written with `?a=1`'s qs; assert no router.replace.
- `"restore on mount: no storage + empty URL → no-op"` — assert no replace, no storage write.
- `"restore on mount: malformed JSON in storage → treated as empty (seeds from URL if URL has params)"` — pre-seed `'not-json'`; render with URL `?a=1`; assert storage rewritten with `a=1`.
- `"restore on mount: version mismatch → treated as empty"` — pre-seed `{version: 0, queryString: 'a=1'}`; render with URL `?b=2`; assert storage seeded from URL.
- `"re-runs on storageKey change (tab switch)"` — render with storageKey `'k1'`, then re-render with `'k2'` having a different stored value; assert restore fires for `'k2'`.
- `"clear removes storage entry, clears pending overrides, navigates to bare pathname"` — call `clear()`; assert localStorage entry removed, mockReplace called with bare pathname.
- `"queryString reflects the current state via serialize"` — assert `result.current.queryString` equals `serialize(state).toString()`.

**Test mock setup** (mirror existing `usePersistedRouteQuery.test.ts`):
```ts
const mockReplace = jest.fn();
const mockSearchParams = jest.fn(() => new URLSearchParams(''));
jest.mock('next/navigation', () => ({
  useRouter: () => ({ replace: mockReplace }),
  useSearchParams: (...args: unknown[]) => mockSearchParams(...args),
  usePathname: () => '/test-path',
}));
```

**Restore semantics (locked above):** storage wins. Restore condition is "stored qs differs from current URL qs". When storage is empty AND URL has params, seed storage from URL. When both are empty, no-op. Storage write inside `update` keeps the mirror in sync after mount.

**Commit**: `[MLID-2225] - feat(url-state): add useUrlStateWithStorage primitive`

---

### Step 2 — Extract orders-tracker URL schema

**Files**:
- Create: `apps/web/app/orders-tracker/urlSchema.ts`
- Create: `apps/web/app/orders-tracker/urlSchema.test.ts`

Pure functions: `parse(URLSearchParams) → OrdersTrackerUrlState` and `serialize(state) → URLSearchParams`. No React. Tests are pure round-trip:

- `"parse returns DEFAULT_PAGINATION when page/pageSize absent"`
- `"parse returns DEFAULT_SORT when sort absent or malformed JSON"`
- `"parse returns DEFAULT_FILTER when filter absent or malformed JSON"`
- `"parse reads search and quickFilter as plain strings"`
- `"serialize omits page when pagination matches default"`
- `"serialize omits sort when sort matches DEFAULT_SORT"`
- `"serialize omits filter when filter is empty"`
- `"serialize omits search/quickFilter when empty/null"`
- `"round-trip: parse(serialize(state)) === state for every non-default permutation"`

Constants `DEFAULT_PAGINATION`, `DEFAULT_SORT`, `DEFAULT_FILTER` move from `useGridUrlModels.ts` into `urlSchema.ts`. The old hook re-exports them temporarily so nothing else breaks until Step 3.

**Commit**: `[MLID-2225] - refactor(orders-tracker): extract URL schema to pure module`

---

### Step 3 — Migrate `orders-tracker/[category]/page.tsx` to `useUrlStateWithStorage`

**Files**:
- Modify: `apps/web/app/orders-tracker/[category]/page.tsx`
- Modify: `apps/web/app/orders-tracker/[category]/page.test.tsx`

**RED**:
- Update existing mocks: replace `useGridUrlModels` mock and `usePersistedRouteQuery` mock with one `useUrlStateWithStorage` mock.
- Add tests:
  - `"useUrlStateWithStorage is called with the correct storage key per tab"` (one per tab).
  - `"clearFilters calls clear from useUrlStateWithStorage"`.
  - `"DataGrid prop changes route through update with the right shape"` (verify `update({ paginationModel: m })`, `update({ sortModel: m, activeQuickFilter: null })`, etc.).
  - Existing tests continue to pass after the rewire.

**GREEN**:
- Replace `const { paginationModel, sortModel, filterModel, search, activeQuickFilter, setPaginationModel, setSearch, updateModels, getCurrentQs, clearModels } = useGridUrlModels();` with:
  ```ts
  const { state, update, clear, queryString } = useUrlStateWithStorage({
    ...ordersTrackerUrlSchema,
    storageKey: `ordersTrackerLastQueryString-${tab}`,
  });
  ```
- Delete the `usePersistedRouteQuery` call and its `clear()` call in `clearFilters` — replaced by the new `clear()`.
- Replace all references: `paginationModel` → `state.paginationModel`, etc.
- DataGrid handlers: `onPaginationModelChange={(m) => update({ paginationModel: m })}`, `onSortModelChange={(m) => update({ sortModel: m, activeQuickFilter: null })}`, `onFilterModelChange={(m) => update({ filterModel: m, activeQuickFilter: null })}`.
- `quickFilter.apply` calls: `update({ activeQuickFilter, sortModel, filterModel })` (already shaped right — just rename).
- `removeFilterItem`: `update({ filterModel: { items: next }, activeQuickFilter: null })`.
- `clearFilters`: `setQuickSearchValue('')`, `tabParamsCache[tab] = ''`, `clear()`.
- `tabParamsCache[tab] = getCurrentQs()` → `tabParamsCache[tab] = queryString ? '?' + queryString : ''`.

**Address the `quickSearchValue` re-sync bug** (latent issue surfaced during timeline review):
- Replace `const [quickSearchValue, setQuickSearchValue] = useState(search);` with a controlled-from-URL pattern: `const [quickSearchValue, setQuickSearchValue] = useState(state.search);` plus a `useEffect` to re-sync when `state.search` changes from outside (e.g., restore, tab switch).
- Or simpler: drop the local `quickSearchValue` state entirely and debounce `state.search` directly. Decide during implementation.

**REFACTOR**:
- Run `npm run test && npm run types:check && npm run lint:fix` from `apps/web/`.
- **Visual verification** in browser: fresh load, restore, clear, tab switch, deep link with explicit params, quick filters, search debounce. Wait for user confirmation before commit (`feedback_visual_test_before_commit`).

**Commit**: `[MLID-2225] - refactor(orders-tracker): adopt useUrlStateWithStorage`

---

### Step 4 — Extract intakes URL schema

**Files**:
- Create: `apps/web/app/intakes/urlSchema.ts`
- Create: `apps/web/app/intakes/urlSchema.test.ts`

Same shape as Step 2. Schema fields: `tab`, `selectedLocations[]`, `selectedTypes[]`, `searchQuery`, `paginationModel`, `sortModel`, `filterModel`. Encoding decisions from the parent plan (comma-separated arrays, JSON-encoded sort/filter, 0-based pagination) are inherited.

**Commit**: `[MLID-2225] - refactor(intakes): extract URL schema to pure module`

---

### Step 5 — Migrate `useIntakesTable` to `useUrlStateWithStorage`

**Files**:
- Modify: `apps/web/app/intakes/hooks/useIntakesTable.ts`
- Modify: `apps/web/app/intakes/hooks/useIntakesTable.test.tsx`
- Modify: `apps/web/app/intakes/IntakesPageNew.tsx` (drop `usePersistedRouteQuery` call — handled by the new hook)
- Modify (or create): `apps/web/app/intakes/IntakesPageNew.test.tsx`

**Key constraint** (open question 2): `useIntakesTable`'s external shape (the ~15 fields it returns) must not change unless the user agrees. The refactor swaps only the URL plumbing — fetch/debounce stay.

**RED → GREEN → REFACTOR**: same pattern as Step 3.

**Visual verification** in browser before commit.

**Commit**: `[MLID-2225] - refactor(intakes): adopt useUrlStateWithStorage`

---

### Step 6 — Delete superseded hooks

**Files**:
- Delete: `apps/web/app/orders-tracker/hooks/useGridUrlModels.ts`
- Delete: `apps/web/app/orders-tracker/hooks/useGridUrlModels.test.ts` (if exists)
- Delete: `apps/web/utils/hooks/usePersistedRouteQuery.ts`
- Delete: `apps/web/utils/hooks/usePersistedRouteQuery.test.ts`
- Modify: `apps/web/utils/hooks/index.ts` (remove export)

**Pre-flight**: `grep` for `useGridUrlModels` and `usePersistedRouteQuery` across the codebase. If the only consumers are the files migrated in Steps 3 and 5, delete safely. If there are other consumers, decide whether to migrate or leave the old hooks in place (per `feedback_delete_unused_code`).

**Commit**: `[MLID-2225] - chore: remove superseded URL state hooks`

---

## Files Affected

| File | Action | Step |
|------|--------|------|
| `apps/web/utils/hooks/useUrlStateWithStorage.ts` | Create | 1 |
| `apps/web/utils/hooks/useUrlStateWithStorage.test.ts` | Create | 1 |
| `apps/web/utils/hooks/index.ts` | Modify | 1, 6 |
| `apps/web/app/orders-tracker/urlSchema.ts` | Create | 2 |
| `apps/web/app/orders-tracker/urlSchema.test.ts` | Create | 2 |
| `apps/web/app/orders-tracker/[category]/page.tsx` | Modify | 3 |
| `apps/web/app/orders-tracker/[category]/page.test.tsx` | Modify | 3 |
| `apps/web/app/intakes/urlSchema.ts` | Create | 4 |
| `apps/web/app/intakes/urlSchema.test.ts` | Create | 4 |
| `apps/web/app/intakes/hooks/useIntakesTable.ts` | Modify | 5 |
| `apps/web/app/intakes/hooks/useIntakesTable.test.tsx` | Modify | 5 |
| `apps/web/app/intakes/IntakesPageNew.tsx` | Modify | 5 |
| `apps/web/app/intakes/IntakesPageNew.test.tsx` | Create/Modify | 5 |
| `apps/web/app/orders-tracker/hooks/useGridUrlModels.ts` | Delete | 6 |
| `apps/web/utils/hooks/usePersistedRouteQuery.ts` | Delete | 6 |

---

## Architectural Wins

| Original concern | How the refactor fixes it |
|---|---|
| `liveStateFromUrl` band-aid (Fix 2 in parent plan) | Hook is the only writer. Reading `window.location` inside `update` is the natural source — no second writer to race. |
| Empty-write race in `usePersistedRouteQuery` Effect 2 | Storage write happens inside `update`, not in a separate effect watching `searchParams`. Cannot see a stale value. |
| `quickSearchValue` doesn't re-sync from URL | Addressed in Step 3 — controlled from `state.search` with a re-sync effect, or debounce `state.search` directly. |
| N redundant setters | One `update(overrides)` API. Caller passes whatever subset of state changed. |
| Restore in a separate hook from URL writes | Both live in the same hook. Restore is a `router.replace` on mount; subsequent updates flow through `update`. |

## Architectural Trade-offs

| Trade-off | Notes |
|---|---|
| Schema must be stable references | Defaults/parse/serialize must be module-scope constants. Document this in the hook's JSDoc and assert via React Strict Mode in dev. |
| Restore semantics | Storage wins on every entry, with seed-from-URL fallback when storage is empty. Same user-visible behavior as parent plan Fix 1. |
| Bigger blast radius | Touching both orders-tracker and intakes in the same PR. Original plan kept them independent. Mitigated by sister-branch isolation. |
| Two renders on fresh load still happen | Inherent to "URL empty → URL filled" requiring `useSearchParams` to refresh. Not a regression vs. current code. |

---

## Testing Strategy

- Unit tests for `useUrlStateWithStorage`: 14 cases (Step 1 above).
- Unit tests for each schema's parse/serialize: round-trip property + edge cases (Steps 2 and 4).
- Page-level mocks updated in Steps 3 and 5; existing test cases must continue to pass.
- Coverage target: 100% on new files, no regression on modified files.
- Manual / browser verification per `feedback_visual_test_before_commit` for Steps 3 and 5.

---

## Recovery Notes (Session Crash)

If a fresh session picks this up:

1. `git checkout feature/MLID-2225-url-state-refactor`
2. Read this file + the parent plan (`MLID-2225-saving-user-filters-and-column-ordering.md`) for original context.
3. Run `git log --oneline feature/MLID-2225-persist-filters-column-order..HEAD` to see which steps are committed.
4. Resume at the next uncommitted step. TDD red-green-refactor per step.
5. **Do not proceed past Step 1 without confirming the three open questions** (above) with the user.
6. **Do not commit Steps 3 or 5 without browser visual verification** by the user.

---

## Status Log

- **2026-05-08**: Sister branch created off `dbeb1b8b`. Plan file written. Awaiting user approval of step order + answers to the three open questions before starting Step 1.
