# MLID-2225 — URL Architecture Refactor (Post-impl)

## Task Reference

- **Jira**: [MLID-2225](https://localinfusion.atlassian.net/browse/MLID-2225)
- **Branch**: `feature/MLID-2225-persist-filters-column-order` (continues — no new branch)
- **Base Branch**: `develop`
- **Status**: In Development (refactor on top of dbeb1b8b)
- **Parent**: MLID-2226

---

## Why this refactor

The original implementation introduced two writers that both treat the URL query string as a single opaque blob:

- `useGridUrlModels.buildQs` rebuilds the entire QS from scratch every time any single field changes. It only knows about its own 6 keys; any param outside that vocabulary is silently dropped.
- `usePersistedRouteQuery` writes/reads the whole QS as one string and races against `buildQs` (Fix 2 papered over the race by reading `window.location` at flush time, but the rebuild model itself is fragile).

**Symptoms observed in develop+initial impl:**

- The DataGrid emits `onSortModelChange([{received,desc}])` after mount — a single-entry sort that does not match `DEFAULT_SORT` (two entries: `received` + `createdAt` tiebreaker). `buildQs` writes that single-entry sort to the URL on every page load, even with no user action.
- After persistence: the post-restore mount-emit clobbers the saved sort because `buildQs` rebuilds the whole QS using the override.
- Any unrelated param a future feature might add to the URL (e.g. `tab=`, `selectedRow=`) would be silently stripped on the next sort/filter change.

**Target model — granular per-key writes, three layers, one direction of authority:**

```
LocalStorage  ◄── source of truth across sessions
     │
     ▼ (seed once on mount, before any other writes)
URL params (Map<key, value>)
     │
     ▼ (parse)
UI components (DataGrid, search box, ...)
     │
     ▲ (user change)
     │
URL params  ── set/delete only the keys you own
     │
     ▼ (mirror on every change)
LocalStorage
```

Each writer owns specific keys and never reaches outside its vocabulary. The URL is a structured `Map<string,string>`, not a string blob.

---

## Architecture

**Layer 1 — Storage primitives** (`apps/web/utils/hooks/urlPersistence.ts`)

Pure module-level functions, no React. Versioned envelope `{ version: 1, queryString }` preserved from the original impl.

- `loadStoredQueryString(key: string): string | null`
- `saveStoredQueryString(key: string, queryString: string): void`
- `clearStoredQueryString(key: string): void`

Malformed JSON or wrong version → silently returns `null`. Catches all `localStorage` exceptions (private browsing, quota exceeded).

**Layer 2 — URL manager hook** (`apps/web/utils/hooks/useUrlParams.ts`)

Generic. Knows nothing about sort/filter/pagination/etc.

- Constructor: `useUrlParams(storageKey?: string)`. When `storageKey` is provided, persistence is enabled.
- On mount (gated by `hasSeededRef`): if storage has a saved QS that differs from the current URL, calls `window.history.replaceState` synchronously + `router.replace` to seed the URL. Storage wins over URL — preserves the Fix 1 decision.
- After mount: every observed `searchParams` change mirrors to storage.
- `setParams(patch: Record<string, string | null>)`: granular merge into URL. `null` deletes a key, string sets it. Touches only the keys in `patch`. Other keys pass through untouched. Batches multiple calls in a microtask flush, writes once.
- `getQueryString(): string` — returns current `window.location.search` without the leading `?`. Used by orders-tracker `tabParamsCache`.
- `clearOwnedKeys(keys: string[])` — convenience: `setParams` with each key set to `null`.

**Layer 3 — Domain hooks** (refactored)

- `useGridUrlModels(storageKey?: string)` — typed accessors over `useUrlParams`. Each setter calls `setParams` with only its own key.
- `useIntakesTable({ storageKey, initialLocations })` — same pattern. Local `buildIntakesUrlParams` and `writeUrl` are deleted.

The page calls `useGridUrlModels(\`ordersTrackerLastQueryString-${tab}\`)` or `useIntakesTable({ storageKey: 'intakesLastQueryString' })`. Persistence is now an internal detail of the domain hook, not a separate page-level concern.

---

## Decisions

1. **Storage wins over URL on every entry** — preserved from Fix 1. Bookmarked/shared URLs with explicit filters get overwritten by the user's saved state. Documented in `useUrlParams` JSDoc.
2. **Granular per-key writes** — `setParams` patches a fresh `URLSearchParams(window.location.search)` and writes back. No rebuild, no `buildQs`, no `liveStateFromUrl`, no `stateRef`. All that scaffolding is deleted.
3. **`history.replaceState` stays synchronous on seed** — preserved from Fix 2a, so any code reading `window.location` inside the same microtask flush sees the seeded URL. `router.replace` still runs to keep Next.js's router state in sync.
4. **No more `liveStateFromUrl`** — `setParams` reads `window.location.search` directly when applying the patch. There's nothing to rebuild from a snapshot.
5. **`usePersistedRouteQuery` is deleted** — its responsibilities split across `urlPersistence` (storage) and `useUrlParams` (seeding/mirroring). New names match the new architecture; old name lingering would mislead readers.
6. **DataGrid mount-emit sort bug stays out of scope** — this refactor makes filters/search/page survive cleanly. The DataGrid emitting a non-default sort on mount and patching the `sort` key is a separate pre-existing bug at the DataGrid↔page boundary. May be addressed in a follow-up.

---

## Implementation Steps

### Step 1 — Storage primitives

**Files**:
- Create: `apps/web/utils/hooks/urlPersistence.ts`
- Create: `apps/web/utils/hooks/urlPersistence.test.ts`

**🔴 RED**

- `urlPersistence.test.ts`:
  - `"loadStoredQueryString returns null when storage is empty"`
  - `"saveStoredQueryString persists with current version envelope"` — assert raw localStorage value matches `{ version: 1, queryString: 'foo=bar' }`
  - `"loadStoredQueryString round-trips a saved queryString"`
  - `"loadStoredQueryString returns null when JSON is malformed"`
  - `"loadStoredQueryString returns null when version mismatches"`
  - `"loadStoredQueryString returns null when shape is wrong"` (e.g. `{ version: '1', queryString: 'x' }`)
  - `"clearStoredQueryString removes the entry"`
  - `"all functions handle localStorage exceptions silently"` — mock `localStorage.setItem` to throw, assert no throw.

Run from `apps/web/`: `npm run test -- --testPathPattern=urlPersistence` → expected fail (module not found).

**🟢 GREEN**

```ts
const CURRENT_VERSION = 1;
interface StoredEntry { version: number; queryString: string; }

export const loadStoredQueryString = (key: string): string | null => { /* ... */ };
export const saveStoredQueryString = (key: string, queryString: string): void => { /* ... */ };
export const clearStoredQueryString = (key: string): void => { /* ... */ };
```

Run: tests pass.

**🔵 REFACTOR**

- Add JSDoc explaining the version-bump policy.
- Run `npm run test && npm run types:check && npm run lint:fix` from `apps/web/`.

**Commit**: `[MLID-2225] - refactor(url-persistence): extract storage primitives`

---

### Step 2 — `useUrlParams` hook

**Files**:
- Create: `apps/web/utils/hooks/useUrlParams.ts`
- Create: `apps/web/utils/hooks/useUrlParams.test.tsx`

**🔴 RED**

- Mock setup mirrors the existing `usePersistedRouteQuery.test.tsx`: `next/navigation` mocks for `useRouter`, `useSearchParams`, `usePathname`. `localStorage.clear()` and `jest.clearAllMocks()` in `beforeEach`.
- Cases:
  - `"setParams writes a single key without touching other URL params"` — start URL `?a=1&b=2`, call `setParams({ b: '3' })`, await microtask, assert `router.replace` and `history.replaceState` called with target containing both `a=1` and `b=3`.
  - `"setParams with null deletes a key"` — start `?a=1&b=2`, `setParams({ a: null })`, assert target contains `b=2` and not `a=`.
  - `"setParams batches multiple synchronous calls into one flush"` — call `setParams({ a: '1' })` and `setParams({ b: '2' })` in same tick, assert `router.replace` called once.
  - `"setParams skips when patched URL equals current URL"` — `?a=1`, call `setParams({ a: '1' })`, assert `router.replace` NOT called.
  - `"seeds URL from storage on mount when storage has a saved QS and storage differs from URL"` — pre-seed storage with `{ version: 1, queryString: 'a=1' }`, render with empty URL, assert `history.replaceState` and `router.replace` called with `?a=1`.
  - `"seeds even when URL has different params (storage wins)"` — pre-seed storage with `a=stored`, render with `?a=url`, assert seed runs.
  - `"does not seed when storage matches current URL"` — both `?a=1`, assert no replace.
  - `"does not seed when storageKey is undefined"` — render with no key, no replace.
  - `"does not seed twice on re-render"` — render, advance mocks, assert `router.replace` called only once for seed (subsequent re-renders don't re-seed).
  - `"mirrors searchParams changes to storage after seed completes"` — render, change mock searchParams to `?a=2`, assert `localStorage.getItem(key)` updated.
  - `"does not mirror to storage before seed has completed"` — guards against wiping storage with the empty mount URL on render 1. Verified by ordering: assert that on the very first effect tick, `localStorage.getItem(key)` is unchanged from its pre-render value.
  - `"clearOwnedKeys deletes the listed keys and leaves others"` — `?a=1&b=2&c=3`, `clearOwnedKeys(['a','c'])`, assert URL contains only `b=2`.
  - `"getQueryString returns current window.location.search without leading ?"`.

Run: expected fail (module not found).

**🟢 GREEN**

```ts
'use client';
import { usePathname, useRouter, useSearchParams } from 'next/navigation';
import { useCallback, useEffect, useRef } from 'react';
import {
  loadStoredQueryString,
  saveStoredQueryString,
} from './urlPersistence';

interface UseUrlParamsReturn {
  setParams: (patch: Record<string, string | null>) => void;
  clearOwnedKeys: (keys: string[]) => void;
  getQueryString: () => string;
}

export const useUrlParams = (storageKey?: string): UseUrlParamsReturn => {
  const router = useRouter();
  const pathname = usePathname();
  const searchParams = useSearchParams();

  const hasSeededRef = useRef(false);
  const pendingRef = useRef<Record<string, string | null>>({});
  const flushScheduledRef = useRef(false);

  // --- seed from storage (Layer 1 → Layer 2) ----------------------------
  useEffect(() => {
    if (hasSeededRef.current) return;
    hasSeededRef.current = true;
    if (!storageKey) return;

    const stored = loadStoredQueryString(storageKey);
    if (!stored) return;

    const currentQs = searchParams?.toString() ?? '';
    if (stored === currentQs) return;

    const target = `${pathname}?${stored}`;
    if (typeof window !== 'undefined') {
      window.history.replaceState(null, '', target);
    }
    router.replace(target);
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []);

  // --- mirror URL → storage (Layer 2 → Layer 1) -------------------------
  useEffect(() => {
    if (!hasSeededRef.current || !storageKey) return;
    saveStoredQueryString(storageKey, searchParams?.toString() ?? '');
  }, [searchParams, storageKey]);

  // --- granular setParams ------------------------------------------------
  const flush = useCallback(() => {
    flushScheduledRef.current = false;
    const patch = pendingRef.current;
    pendingRef.current = {};

    if (typeof window === 'undefined') return;
    const next = new URLSearchParams(window.location.search);
    for (const [key, value] of Object.entries(patch)) {
      if (value === null) next.delete(key);
      else next.set(key, value);
    }
    const nextQs = next.toString();
    const currentQs = window.location.search.replace(/^\?/, '');
    if (nextQs === currentQs) return;
    const target = `${pathname}${nextQs ? '?' + nextQs : ''}`;
    window.history.replaceState(null, '', target);
    router.replace(target, { scroll: false });
  }, [pathname, router]);

  const setParams = useCallback(
    (patch: Record<string, string | null>) => {
      Object.assign(pendingRef.current, patch);
      if (flushScheduledRef.current) return;
      flushScheduledRef.current = true;
      queueMicrotask(flush);
    },
    [flush]
  );

  const clearOwnedKeys = useCallback(
    (keys: string[]) => {
      const patch: Record<string, string | null> = {};
      for (const k of keys) patch[k] = null;
      setParams(patch);
    },
    [setParams]
  );

  const getQueryString = useCallback(
    (): string =>
      typeof window === 'undefined'
        ? ''
        : window.location.search.replace(/^\?/, ''),
    []
  );

  return { setParams, clearOwnedKeys, getQueryString };
};
```

Run: tests pass.

**🔵 REFACTOR**

- JSDoc on the hook describing the seed-on-mount + mirror-after-seed contract and the storage-wins-over-URL semantic.
- Export from `apps/web/utils/hooks/index.ts`.
- Run `npm run test && npm run types:check && npm run lint:fix` from `apps/web/`.

**Commit**: `[MLID-2225] - refactor(url-params): add useUrlParams with granular setParams and storage seeding`

---

### Step 3 — Refactor `useGridUrlModels`

**Files**:
- Modify: `apps/web/app/orders-tracker/hooks/useGridUrlModels.ts`
- Modify: `apps/web/app/orders-tracker/hooks/useGridUrlModels.test.ts` (if exists; if not, the existing page tests continue to cover behavior)

**🔴 RED — adapt existing tests + add granularity tests**

- Confirm whether a dedicated `useGridUrlModels.test.ts` exists; the orders-tracker page test mocks the hook wholesale so no direct test changes there.
- New focused tests (in a new `useGridUrlModels.test.tsx` if absent):
  - `"setSortModel patches only the sort key"` — start `?filter={...}&search=foo`, call `setSortModel(...)`, assert URL still has `filter` and `search` and now also `sort`.
  - `"setFilterModel patches only the filter key"` — symmetric.
  - `"setSearch with empty string deletes search param"`.
  - `"setActiveQuickFilter null deletes the param"`.
  - `"clearModels deletes only the keys the hook owns"` — start with the hook's keys + an unrelated `?other=keep`, call `clearModels`, assert `other=keep` survives.

Run: expected fail (current implementation rebuilds whole QS).

**🟢 GREEN**

Replace the body. Skeleton:

```ts
const HOOK_KEYS = ['page', 'pageSize', 'sort', 'filter', 'search', 'quickFilter'];

export function useGridUrlModels(storageKey?: string) {
  const { setParams, clearOwnedKeys, getQueryString } = useUrlParams(storageKey);
  const searchParams = useSearchParams();

  const paginationModel = useMemo(...);  // parse from searchParams
  const sortModel = useMemo(...);
  const filterModel = useMemo(...);
  const search = searchParams?.get('search') ?? '';
  const activeQuickFilter = searchParams?.get('quickFilter') ?? null;

  const setPaginationModel = useCallback((m: GridPaginationModel) => {
    setParams({
      page: m.page === DEFAULT_PAGINATION.page ? null : String(m.page),
      pageSize:
        m.pageSize === DEFAULT_PAGINATION.pageSize ? null : String(m.pageSize),
    });
  }, [setParams]);

  const setSortModel = useCallback((m: GridSortModel) => {
    setParams({
      sort: isDefault(m, DEFAULT_SORT) ? null : JSON.stringify(m),
    });
  }, [setParams]);

  // ...filterModel, search, activeQuickFilter, updateModels (multi-key patch)

  const clearModels = useCallback(() => clearOwnedKeys(HOOK_KEYS), [clearOwnedKeys]);
  const getCurrentQs = useCallback(() => {
    const qs = getQueryString();
    return qs ? `?${qs}` : '';
  }, [getQueryString]);

  return { paginationModel, sortModel, filterModel, search, activeQuickFilter,
           setPaginationModel, setSortModel, setFilterModel, setSearch,
           setActiveQuickFilter, updateModels, getCurrentQs, clearModels };
}
```

`updateModels` becomes a single `setParams` call with all overrides combined.

`buildQs`, `liveStateFromUrl`, `stateRef`, `pendingRef` deleted from this file (live in `useUrlParams` now).

**🔵 REFACTOR**

- Drop the export of `DEFAULT_PAGINATION`/`DEFAULT_SORT`/`DEFAULT_FILTER` if no other module imports them; otherwise keep for callers.
- Run `npm run test && npm run types:check && npm run lint:fix` from `apps/web/`.

**Commit**: `[MLID-2225] - refactor(orders-tracker): useGridUrlModels patches keys granularly via useUrlParams`

---

### Step 4 — Refactor `useIntakesTable`

**Files**:
- Modify: `apps/web/app/intakes/hooks/useIntakesTable.ts`
- Modify: `apps/web/app/intakes/hooks/useIntakesTable.test.tsx`

**🔴 RED — augment existing tests**

- Add granularity assertions:
  - `"setSelectedLocations patches only loc, leaves other params"` — start `?tab=Needs+Review&search=jones`, call setter, assert search/tab survive and `loc` is added.
  - `"setSelectedTypes patches only types"`.
  - `"setSearchQuery debounce patches only search"`.

Run: existing tests should still pass with new impl; new tests fail until impl lands.

**🟢 GREEN**

- Replace `writeUrl` and `buildIntakesUrlParams` with `setParams` from `useUrlParams`.
- Keep the URL-read initializers (`parseCsvList`, `parsePagination`, etc.) — they're already correct.
- Each setter becomes a one-liner:
  ```ts
  const setSelectedLocations = useCallback((next: string[]) => {
    setSelectedLocationsState(next);
    setParams({ loc: next.length > 0 ? next.join(',') : null });
  }, [setParams]);
  ```
- `handleTabChange` wipes the hook's owned keys but keeps `tab`:
  ```ts
  setParams({
    loc: null, types: null, search: null,
    page: null, pageSize: null, sort: null, filter: null,
    tab: encodeURIComponent(tab),  // actually URLSearchParams handles encoding
  });
  ```
  (Replace the current `router.replace('/intakes?tab=...')` call.)

**🔵 REFACTOR**

- Drop the `searchParamsRef`/`routerRef` indirection — `setParams` from `useUrlParams` is referentially stable.
- Drop the `isDefaultSort`/`isEmptyFilter` helpers if they're only used inside `buildIntakesUrlParams` (which is being deleted).
- Run `npm run test && npm run types:check && npm run lint:fix` from `apps/web/`.

**Commit**: `[MLID-2225] - refactor(intakes): useIntakesTable patches keys granularly via useUrlParams`

---

### Step 5 — Drop `usePersistedRouteQuery` from pages

**Files**:
- Modify: `apps/web/app/orders-tracker/[category]/page.tsx`
- Modify: `apps/web/app/orders-tracker/[category]/page.test.tsx`
- Modify: `apps/web/app/intakes/IntakesPageNew.tsx`
- Modify: `apps/web/app/intakes/page.test.tsx` (intakes uses this for `IntakesPageNew`)

**🔴 RED — adapt tests**

- Orders page: drop the `usePersistedRouteQuery` mock; instead assert `useGridUrlModels` is invoked with the correct per-tab storage key. The Step-2 page tests (`"usePersistedRouteQuery is called with the correct storage key"`) get rewritten to assert the storage key reaches `useGridUrlModels`.
- Intakes page: same — drop mock, assert `useIntakesTable` is invoked with `{ storageKey: 'intakesLastQueryString', ... }`.

**🟢 GREEN**

- Orders page: delete `usePersistedRouteQuery` import and the `persistedQueryString` declaration. Pass storage key into `useGridUrlModels`:
  ```ts
  const { paginationModel, sortModel, ... } = useGridUrlModels(
    `ordersTrackerLastQueryString-${tab}`
  );
  ```
  In `clearFilters`, drop the `persistedQueryString.clear()` call (storage clears automatically when URL clears via `clearModels`).
- Intakes page: same shape, pass `storageKey` to `useIntakesTable`.

**🔵 REFACTOR**

- Confirm `clearFilters` semantics: `clearModels()` deletes the hook's keys → URL changes → storage mirrors empty QS automatically. No explicit storage clear needed.
- Run `npm run test && npm run types:check && npm run lint:fix` from `apps/web/`.

**Commit**: `[MLID-2225] - refactor(pages): persistence is now internal to domain hooks`

---

### Step 6 — Delete `usePersistedRouteQuery`

**Files**:
- Delete: `apps/web/utils/hooks/usePersistedRouteQuery.ts`
- Delete: `apps/web/utils/hooks/usePersistedRouteQuery.test.tsx`
- Modify: `apps/web/utils/hooks/index.ts` — remove the re-export.

**Verify** with grep that no callers remain. Run full test/types/lint.

**Commit**: `[MLID-2225] - refactor(persistence): remove usePersistedRouteQuery (replaced by useUrlParams)`

---

### Step 7 — Verification gate (no commit)

Run from `apps/web/`:
- `npm run test`
- `npm run types:check`
- `npm run lint:fix`

Stop here. Wait for browser-side visual verification before any commit beyond the per-step commits above (or, if the per-step commits are also gated, stop before each).

---

## Files Affected

| File | Action |
|---|---|
| `apps/web/utils/hooks/urlPersistence.ts` | Create |
| `apps/web/utils/hooks/urlPersistence.test.ts` | Create |
| `apps/web/utils/hooks/useUrlParams.ts` | Create |
| `apps/web/utils/hooks/useUrlParams.test.tsx` | Create |
| `apps/web/utils/hooks/usePersistedRouteQuery.ts` | Delete |
| `apps/web/utils/hooks/usePersistedRouteQuery.test.tsx` | Delete |
| `apps/web/utils/hooks/index.ts` | Modify (drop usePersistedRouteQuery re-export, add useUrlParams + urlPersistence) |
| `apps/web/app/orders-tracker/hooks/useGridUrlModels.ts` | Major refactor (drop buildQs/liveStateFromUrl/stateRef) |
| `apps/web/app/orders-tracker/hooks/useGridUrlModels.test.tsx` | Create or augment |
| `apps/web/app/orders-tracker/[category]/page.tsx` | Drop usePersistedRouteQuery; pass storageKey to useGridUrlModels |
| `apps/web/app/orders-tracker/[category]/page.test.tsx` | Adapt mocks |
| `apps/web/app/intakes/hooks/useIntakesTable.ts` | Major refactor (drop writeUrl/buildIntakesUrlParams) |
| `apps/web/app/intakes/hooks/useIntakesTable.test.tsx` | Augment with granularity tests |
| `apps/web/app/intakes/IntakesPageNew.tsx` | Drop usePersistedRouteQuery; pass storageKey to useIntakesTable |
| `apps/web/app/intakes/page.test.tsx` | Adapt mocks |

---

## Manual Verification

After Step 6 (and Step 7 passes), repeat the original verification matrix:

**Orders Tracker**
1. Set filter A on `/orders-tracker/new`, navigate away, return → filter restored.
2. Clear filters, navigate away, return → no filters.
3. Per-tab isolation: New filter A vs Maintenance filter B both restore independently.
4. Storage wins over URL: pasting a URL with explicit filter loads the saved one.

**Intakes**
1. Set location/search/tab, navigate away, return → restored.
2. Clear All, navigate away, return → no filters.

**Granularity (new for this refactor)**
1. Open dev tools, set `?sort=...&filter=...&someExternalParam=keepme` manually. Trigger a filter change. Confirm `someExternalParam=keepme` survives (currently would be wiped by `buildQs`).

---

## Out of Scope

- DataGrid mount-emit clobbering the saved sort. Pre-existing in develop. Tracked as a follow-up — fix is either dropping `createdAt` from `DEFAULT_SORT` or adding a "first emission swallowed" guard at the page boundary.
- Bonus column-order persistence (Step 5 of original plan). Already shipped as-is; this refactor does not touch `Orders.tsx`.

---

## Post-implementation adjustments

The following changes happened during execution and weren't called out in the plan above. Captured here so the plan reflects what actually shipped.

### useUrlParams — mirror reads `window.location`, not `searchParams`

**Why:** the plan's mirror effect read `searchParams?.toString() ?? ''`. The "does not wipe pre-seeded storage on the very first mount tick" test failed because, on render 1, `searchParams` (from `useSearchParams()`) is still the pre-seed value — `router.replace` is deferred. The mirror was writing the empty pre-seed `searchParams` over the just-restored localStorage entry. Switched to `window.location.search.replace(/^\?/, '')` since the seed updates `window.location` synchronously via `replaceState`. The dep array still uses `searchParams` so the effect re-runs on URL changes going forward.

### useUrlParams — `routerRef` / `pathnameRef` defensive pattern

**Why:** Step 4 surfaced a cascade in the intakes pagination-reset effect. In the test mock, `useRouter()` returns a fresh `{ replace: mockReplace }` object every render, which would otherwise churn `setParams`'s `useCallback` identity → cascade into `setPaginationModel` identity → re-fire the pagination-reset effect on every render. Wrapped `router` and `pathname` in refs and made `setParams` `useCallback([])` to lock its identity. Real Next.js's `useRouter()` already returns a stable object, so this is purely test-stability defense (production behavior unchanged).

### Test infra changes

- **`useIntakesTable.test.tsx` — `setupUrlTracking` helper syncs `window.location`.** The original helper only updated `mockUseSearchParams` when `mockReplace` was called. Once `useUrlParams.setParams` started reading from `window.location` at flush time, tests needed both kept in lockstep.
- **3 intakes tests use `await act(async)` instead of `act()`.** `setParams` queues a microtask; synchronous `act()` returns before the flush. The "writes loc param", "removes loc param", and "strips all filter params on tab change" tests all required `await act(async () => { ...; await Promise.resolve(); })` to flush before asserting on the captured URL.
- **`orders-tracker/[category]/page.test.tsx` — `jest.restoreAllMocks()` added to `beforeEach`.** The "Custom filter visibility" and "Filter chip label formatting" describes use `jest.spyOn(require('../hooks/useGridUrlModels'), 'useGridUrlModels').mockReturnValue(...)` to inject specific filter models. Without restoring spies, the new "Filter persistence" describe (which asserts on the default `mockUseGridUrlModels` jest.fn call args) saw the spy's `mockReturnValue` instead of the wrapper, and the call counter never incremented.

### Intakes page test — persistence describe deleted, not modified

Step 5 of the plan said "Adapt mocks". In practice the entire `IntakesPageNew filter persistence (usePersistedRouteQuery)` describe block had no equivalent in the new architecture — the contract it tested (storage-key wiring + `clear()` propagation) is now covered at the hook level by `useIntakesTable.test.tsx` and `useUrlParams.test.tsx`. The block was removed wholesale.

### DataGrid filter mount-emit guard (orders-tracker page)

**Originally Out of Scope (sort variant).** During browser verification of the refactor, the user reported a related but distinct symptom: after seeding the URL with a 2-item filter and hard-refreshing the page, only one item survived in the URL and storage. Browser console traces (added temporarily via `console.trace` inside `setParams`) pinpointed MUI X's `useGridFilter.js:217` mount-time normalization emitting `onFilterModelChange` with a stripped filter, on a render cycle after the existing `hasMounted` ref had flipped to `true`. The handler then forwarded the truncated value through `setParams`, the URL was wiped, and the mirror saved the wipe to storage.

**Fix in `[category]/page.tsx`'s `handleFilterModelChange`:**

```ts
if (m.items.length < filterModel.items.length) return;
```

Reject emits that strictly reduce item count — that's normalization noise, not user intent. **Trade-off:** the DataGrid filter panel's per-row trash icon stops working as a removal path; users remove via the active-filter chip's `onDelete` (`removeFilterItem`), which bypasses `handleFilterModelChange` entirely. Chips are always rendered when filters are active, so the chip-removal UX is the primary path users see.

The diagnostic `console.trace`/`console.log` calls were removed before commit. **No dedicated test was added** — the `[category]/page.test.tsx` mocks `useGridUrlModels` wholesale, and the surrounding test setup doesn't surface MUI X's mount-time `onFilterModelChange` cascade through the real component tree. Flagged as a known follow-up in the PR doc.

### Intakes seed-race fix (separate commit `bc1b62c1`)

**Bug:** `useIntakesTable` uses `useState` with URL-derived initializers — those initializers run once at mount with the pre-seed (clean) URL. After `useUrlParams.seed` populated the URL from storage, the local state never re-synced — the page rendered as "no filters" even when the URL had them. Worse: the search and filter debounce `useEffect`s scheduled 300ms timers on mount with the captured empty mount-state, and 300ms later called `setParams({search: null, filter: null})` — wiping the seeded values from the URL. The mirror then saved the wiped URL to storage. The user observed storage reduced to just `sort=[]`.

`useGridUrlModels` doesn't have this bug because it uses `useMemo` derived from `searchParams` — auto-resyncs on URL changes.

**Fix in `useIntakesTable.ts`:**

1. **`isFirstSearchDebounceRef` and `isFirstFilterDebounceRef`** — skip-first-tick guards on the two debounce useEffects so the empty mount-state never reaches `setParams`. First user keystroke triggers normal debounce.

2. **Per-key URL→state sync useEffects** for `loc`, `types`, `search`, `page`/`pageSize`, `sort`, `filter`. Each watches its URL string; when it changes externally (e.g. after seed), the local state catches up via functional `setState` (skips re-render when the value already matches, so no churn during normal user-driven updates).

**Tests added:**

- `"syncs local state when URL changes externally after mount (simulates useUrlParams seed)"` — mounts with empty URL, swaps the `searchParams` mock to a populated URL, asserts local state catches up.
- `"does not write empty mount-state to URL on the first debounce tick"` — mounts with a tab-only URL, advances 1000ms with no user input, asserts `mockReplace` was never called and the URL is unchanged.

### Chip alignment fix on intakes (separate commit `e4519448`)

UI designer spec: filter chips should be right-aligned, matching the orders-tracker layout. `IntakesFilterBar` rendered chips inside a flexbox `<Box>` with no justify; orders-tracker uses `<Layout justify="end">`. Added `justifyContent: 'flex-end'` to the `Box`'s `sx`. No test changes (existing tests assert on chip presence, not layout).

---

## Final commit history

| Hash | Message |
|------|---------|
| `dbeb1b8b` | `feat(filter-persistence): cross-session filter and column-order restore` (original implementation, predates this refactor plan) |
| `29527049` | `refactor(url-state): three-layer architecture for filter persistence` (this plan, plus the DataGrid mount-emit guard added during verification) |
| `bc1b62c1` | `fix(intakes): preserve URL state across mount-time seed by useUrlParams` (intakes seed-race fix) |
| `e4519448` | `fix(intakes): right-align active filter chips per UI designer spec` (chip alignment) |

PR doc: `docs/agomez/PR/MLID-2225.md`.
