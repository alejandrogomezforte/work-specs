# MLID-2225 — Saving Users Filters & Column Ordering

## Task Reference

- **Jira**: [MLID-2225](https://localinfusion.atlassian.net/browse/MLID-2225)
- **Story Points**: (see Jira)
- **Branch**: `feature/MLID-2225-persist-filters-column-order`
- **Base Branch**: `develop`
- **Status**: In Development
- **Parent**: MLID-2226 (Order Tracker - User Enhancements) — no epic plan file

---

## Summary

Persist the last-used filter/sort/search state for the Orders Tracker and Intakes pages across browser sessions using localStorage. The hook `usePersistedRouteQuery` mirrors the URL query string to localStorage and restores it on next page entry — **storage wins over URL** (see Post-implementation Fix 1). Intakes requires a prerequisite URL-serialization refactor of `useIntakesTable` before the hook can be wired in. A bonus final step persists column drag-order on the Orders Tracker.

---

## Codebase Analysis

**`apps/web/utils/hooks/`** is the confirmed home for shared hooks. `useMRU.ts` establishes the pattern: typed localStorage read/write helpers at module level, `try/catch` on every access, no `localStorage` calls outside effects or `useRef` initializers (SSR safety). The new hook lives here and is exported from `apps/web/utils/hooks/index.ts`.

**Orders Tracker URL state** is already complete via `useGridUrlModels` (`apps/web/app/orders-tracker/hooks/useGridUrlModels.ts`). All six params (`page`, `pageSize`, `sort`, `filter`, `search`, `quickFilter`) are serialized to the URL on every change. `getCurrentQs()` returns the current full query string. `clearModels()` calls `router.replace(pathname)`. The hook is mocked wholesale in `page.test.tsx` (lines 51–68) — the new localStorage layer sits above it in `page.tsx`, not inside it.

**`tabParamsCache`** at `page.tsx:70–74` is a module-level `Record<OrderCategory, string>` for in-session tab switching. It is read in `handleTabChange` (line 540) and cleared in `clearFilters` (line 483). Keep it. The cross-session restore is layered on top.

**`clearFilters`** at `page.tsx:481–485`: calls `setQuickSearchValue('')`, writes `tabParamsCache[tab] = ''`, calls `clearModels()`. Step 2 adds `persisted.clear()` here.

**Intakes filter state** lives entirely in React state inside `useIntakesTable` (`apps/web/app/intakes/hooks/useIntakesTable.ts:155–178`). Only `activeTab` is URL-synced. The stale comment at lines 295–297 references MLID-2125 which was the intakes redesign itself (deployed 2026-05-05) — not a separate URL-sync ticket. Step 3 fixes this comment and migrates all filter state to URL params. The hook's debounced values (`debouncedSearch`, `debouncedFilterModel`) drive fetch — Step 3 persists those, not the raw state, to avoid URL thrash and history pollution.

**Intakes column visibility** uses Pattern B (sync read in `useState` initializer, inline write in `handleColumnVisibilityChange`). Storage key is `intakesColumnVisibilityModel` from `apps/web/app/intakes/constants.ts:112`. No conflict with new keys.

**MUI X column ordering**: `onColumnOrderChange` prop fires after drag. `apiRef.current.getAllColumns().map(c => c.field)` yields the new order as `string[]`. `initialState.columns.orderedFields` seeds the order on mount. `Orders.tsx` already receives `gridApiRef` as a prop, so no new prop needed on `Orders`.

**Test patterns**:
- `useIntakesTable.test.tsx`: `mockUseSearchParams` factory + `localStorage.clear()` in `beforeEach`. Add the new URL params to the mock factory.
- `page.test.tsx` (orders tracker): mocks `useGridUrlModels` entirely. The new `usePersistedRouteQuery` hook will need its own mock there.
- `Orders.test.tsx`: mocks `@repo/ui`'s `DataGrid` as a `div` — `onColumnOrderChange` won't fire through the mock. Column order tests go in `Orders.test.tsx` with a partial DataGrid mock that captures the callback.

---

## Implementation Steps

### Step 1 — Build `usePersistedRouteQuery` hook

**Branch state**: fresh `feature/MLID-2225-persist-filters-column-order` off `develop`

**Files**:
- Create: `apps/web/utils/hooks/usePersistedRouteQuery.ts`
- Create: `apps/web/utils/hooks/usePersistedRouteQuery.test.ts`
- Modify: `apps/web/utils/hooks/index.ts`

---

**🔴 RED — failing test**

- Test file: `apps/web/utils/hooks/usePersistedRouteQuery.test.ts`
- Mock setup:
  ```ts
  const mockReplace = jest.fn();
  const mockSearchParams = jest.fn(() => new URLSearchParams(''));
  jest.mock('next/navigation', () => ({
    useRouter: () => ({ replace: mockReplace }),
    useSearchParams: (...args: unknown[]) => mockSearchParams(...args),
    usePathname: () => '/orders-tracker/new',
  }));
  ```
  Reset `localStorage.clear()` and `jest.clearAllMocks()` in `beforeEach`.

- Test cases:
  - `"should write current QS to localStorage when search params are non-empty"` — render hook with `storageKey='test-key'`, set mock searchParams to `?filter=foo`, advance timers; assert `localStorage.getItem('test-key')` equals `JSON.stringify({ version: 1, queryString: 'filter=foo' })`.
  - `"should restore saved QS via router.replace when URL is empty and storage has a value"` — pre-seed `localStorage.setItem('test-key', JSON.stringify({ version: 1, queryString: 'filter=bar' }))`; render hook with empty search params; assert `mockReplace` called with `/orders-tracker/new?filter=bar`.
  - `"should not restore when URL already has params (deep link protection)"` — pre-seed storage; render with `?filter=existing`; assert `mockReplace` NOT called.
  - `"should discard stored value when version does not match"` — pre-seed with `{ version: 0, queryString: 'old=data' }`; render with empty URL; assert `mockReplace` NOT called.
  - `"should discard stored value when JSON is malformed"` — pre-seed with `'not-json'`; render with empty URL; assert `mockReplace` NOT called, no thrown error.
  - `"clear() removes the stored value from localStorage"` — render hook, write something to storage, call `result.current.clear()`; assert `localStorage.getItem('test-key')` is null.
  - `"should not clobber existing URL with storage when hasLoaded is false during SSR guard"` — assert `mockReplace` is only called once (on mount, not again when searchParams change after restore).

- Run: `npm run test -- --testPathPattern=usePersistedRouteQuery` from `apps/web/` → expected failure: module not found.

---

**🟢 GREEN — minimal implementation**

- File: `apps/web/utils/hooks/usePersistedRouteQuery.ts`
- Shape:
  ```ts
  'use client';

  import { usePathname, useRouter, useSearchParams } from 'next/navigation';
  import { useCallback, useEffect, useRef } from 'react';

  const CURRENT_VERSION = 1;

  type StoredEntry = { version: number; queryString: string };

  function readEntry(key: string): StoredEntry | null { /* try/catch JSON.parse, validate shape */ }
  function writeEntry(key: string, queryString: string): void { /* try/catch localStorage.setItem */ }
  function clearEntry(key: string): void { /* try/catch localStorage.removeItem */ }

  export interface UsePersistedRouteQueryReturn {
    clear: () => void;
  }

  export function usePersistedRouteQuery(storageKey: string): UsePersistedRouteQueryReturn {
    const router = useRouter();
    const pathname = usePathname();
    const searchParams = useSearchParams();
    const hasRestoredRef = useRef(false);

    // Effect 1: restore on empty URL (runs once on mount)
    useEffect(() => {
      if (hasRestoredRef.current) return;
      const currentQs = searchParams?.toString() ?? '';
      if (currentQs) {
        hasRestoredRef.current = true; // URL already populated — skip restore
        return;
      }
      const entry = readEntry(storageKey);
      if (entry && entry.queryString) {
        router.replace(`${pathname}?${entry.queryString}`);
      }
      hasRestoredRef.current = true;
    }, []); // intentionally empty — runs exactly once

    // Effect 2: persist current QS to localStorage whenever it changes
    useEffect(() => {
      if (!hasRestoredRef.current) return;
      const queryString = searchParams?.toString() ?? '';
      if (typeof window === 'undefined') return;
      writeEntry(storageKey, queryString);
    }, [searchParams, storageKey]);

    const clear = useCallback(() => {
      clearEntry(storageKey);
    }, [storageKey]);

    return { clear };
  }
  ```
- Run: `npm run test -- --testPathPattern=usePersistedRouteQuery` → expected pass.

---

**🔵 REFACTOR**

- Tighten `readEntry`: validate `typeof entry.version === 'number'` and `typeof entry.queryString === 'string'` before returning.
- Add JSDoc comment block explaining the `hasRestoredRef` gate and the `CURRENT_VERSION` bump policy.
- Export from `apps/web/utils/hooks/index.ts`: append `export * from './usePersistedRouteQuery';`.
- Run: `npm run test && npm run types:check && npm run lint:fix` from `apps/web/`.

**Commit**: `[MLID-2225] - feat(persist-filters): add usePersistedRouteQuery hook`

---

### Step 2 — Wire `usePersistedRouteQuery` into Orders Tracker

**Branch state**: Step 1 merged into feature branch.

**Files**:
- Modify: `apps/web/app/orders-tracker/[category]/page.tsx`
- Modify: `apps/web/app/orders-tracker/[category]/page.test.tsx`

---

**🔴 RED — failing test**

- Test file: `apps/web/app/orders-tracker/[category]/page.test.tsx`
- Add mock at top of file (alongside existing mocks):
  ```ts
  const mockPersistedClear = jest.fn();
  jest.mock('@/utils/hooks/usePersistedRouteQuery', () => ({
    usePersistedRouteQuery: jest.fn(() => ({ clear: mockPersistedClear })),
  }));
  ```
- Test cases (add to existing `describe('OrdersTrackerPage')`):
  - `"clearFilters calls persisted.clear()"` — render page, click "Clear Filters" button (or call `clearFilters` via test handle); assert `mockPersistedClear` was called once.
  - `"usePersistedRouteQuery is called with the correct storage key for the new tab"` — render with `usePathname` returning `/orders-tracker/new`; assert `usePersistedRouteQuery` was called with `'ordersTrackerLastQueryString-new'`.
  - `"usePersistedRouteQuery is called with the correct storage key for the maintenance tab"` — same for `/orders-tracker/maintenance` → `'ordersTrackerLastQueryString-maintenance'`.
  - `"usePersistedRouteQuery is called with the correct storage key for the lead tab"` — same for `/orders-tracker/lead` → `'ordersTrackerLastQueryString-lead'`.

- Run: `npm run test -- --testPathPattern="orders-tracker/\[category\]/page"` → expected failure: `usePersistedRouteQuery` not imported, `mockPersistedClear` never called.

---

**🟢 GREEN — minimal implementation**

- File: `apps/web/app/orders-tracker/[category]/page.tsx`
- Changes:
  1. Import `usePersistedRouteQuery` from `@/utils/hooks`.
  2. Derive storage key from `tab`: `` const persistedQueryStringKey = `ordersTrackerLastQueryString-${tab}`; `` (after line 101 where `tab` is derived).
  3. Call the hook: `const persistedQueryString = usePersistedRouteQuery(persistedQueryStringKey);`
  4. In `clearFilters` (line 481): add `persistedQueryString.clear();` after `tabParamsCache[tab] = '';`.
- No other changes in this step.

- Run: `npm run test -- --testPathPattern="orders-tracker/\[category\]/page"` → expected pass.

---

**🔵 REFACTOR**

- Confirm `persistedQueryStringKey` derivation is co-located with `tab` derivation for readability.
- Run: `npm run test && npm run types:check && npm run lint:fix` from `apps/web/`.

**Commit**: `[MLID-2225] - feat(orders-tracker): wire usePersistedRouteQuery for cross-session filter restore`

---

### Step 3 — URL-serialize intakes filters in `useIntakesTable`

**Branch state**: Steps 1–2 merged into feature branch.

**Files**:
- Modify: `apps/web/app/intakes/hooks/useIntakesTable.ts`
- Modify: `apps/web/app/intakes/hooks/useIntakesTable.test.tsx`

---

**🔴 RED — failing test**

- Test file: `apps/web/app/intakes/hooks/useIntakesTable.test.tsx`
- Extend the existing mock setup: add `usePathname: () => '/intakes'` and ensure `mockUseSearchParams` covers the new QS params.
- Test cases (add to existing `describe('useIntakesTable')`):
  - `"should read selectedLocations from URL on mount"` — set `mockUseSearchParams` to `new URLSearchParams('tab=Needs+Review&loc=SiteA%2CSiteB')`; render; assert `result.current.selectedLocations` equals `['SiteA', 'SiteB']`.
  - `"should read selectedTypes from URL on mount"` — set QS `?tab=Needs+Review&types=Type1%2CType2`; assert `result.current.selectedTypes` equals `['Type1', 'Type2']`.
  - `"should read searchQuery from URL on mount"` — set QS `?tab=Needs+Review&search=jones`; assert `result.current.searchQuery` equals `'jones'`.
  - `"should read paginationModel from URL on mount"` — set QS `?tab=Needs+Review&page=2&pageSize=50`; assert `result.current.paginationModel` equals `{ page: 2, pageSize: 50 }` (URL is 0-based, matches orders tracker convention).
  - `"should read sortModel from URL on mount"` — set QS `?tab=Needs+Review&sort=%5B%7B%22field%22%3A%22patientName%22%2C%22sort%22%3A%22asc%22%7D%5D`; assert `result.current.sortModel` equals `[{ field: 'patientName', sort: 'asc' }]`.
  - `"should read filterModel from URL on mount"` — set QS with `filter=<JSON-encoded>`; assert `result.current.filterModel.items` has the expected item.
  - `"should call router.replace with loc param when setSelectedLocations is called"` — render with default QS; act `result.current.setSelectedLocations(['SiteA'])`; assert `mockReplace` called with URL containing `loc=SiteA`.
  - `"should remove loc param from URL when setSelectedLocations is called with empty array"` — assert URL does NOT contain `loc=`.
  - `"should sync debouncedSearch to URL, not raw searchQuery"` — act `setSearchQuery('abc')`, advance timer 300ms; assert `mockReplace` called with `search=abc` only after the debounce fires.
  - `"handleTabChange removes all filter params except tab"` — set QS with `loc=SiteA&search=jones`; act `handleTabChange('Processing')`; assert `mockReplace` called with only `?tab=Processing`.

- Run: `npm run test -- --testPathPattern=useIntakesTable` → expected failure: assertions on `result.current.selectedLocations` fail (state not populated from URL).

---

**🟢 GREEN — minimal implementation**

- File: `apps/web/app/intakes/hooks/useIntakesTable.ts`
- Key changes:
  1. Add `usePathname` to the `next/navigation` import.
  2. Replace `useState` initializers for all filter state with URL-read initializers:
     - `selectedLocations`: read `searchParams?.get('loc')`, split on comma, filter empty strings.
     - `selectedTypes`: read `searchParams?.get('types')`, split on comma.
     - `searchQuery`: read `searchParams?.get('search') ?? ''`.
     - `paginationModel`: read `page` (0-based from URL, no conversion) and `pageSize`. The existing API conversion (`page + 1`) stays inside `buildParams`.
     - `sortModel`: `safeJsonParse(searchParams?.get('sort'), DEFAULT_SORT)` — inline a `safeJsonParse` helper or import from `useGridUrlModels` (cannot import — that file is not exported; define locally).
     - `filterModel`: `safeJsonParse(searchParams?.get('filter'), { items: [] })`.
  3. Replace each raw setter with a URL-writing wrapper:
     - `setSelectedLocations`: calls `router.replace` with updated params (comma-joined, deleted when empty).
     - `setSelectedTypes`: same.
     - `setSearchQuery`: updates local `useState` only (debounce writes to URL in the debounce effect).
     - `setPaginationModel`: writes `page` (0-based, no conversion) and `pageSize` to URL.
     - `setSortModel`: writes JSON-encoded `sort` to URL.
     - `setFilterModel`: writes JSON-encoded `filter` to URL.
  4. Debounce write: after `debouncedSearch` settles (existing 300ms effect), call `router.replace` with `search=<value>` (or delete `search` param if empty). Same for `debouncedFilterModel`.
  5. `handleTabChange` (line 298): strip all filter params — keep only `tab` in the replace call. Current code only sets `tab`; confirm `page`, `loc`, `types`, `search`, `sort`, `filter` are not inherited.
  6. Fix stale comment at lines 295–297: replace with `// All filter state is persisted to the URL. usePersistedRouteQuery (MLID-2225) restores the last-used QS on next page entry.`
  7. Existing `setActiveTab` via `validTabFromUrl` sync effect (line 291) is unchanged.

- Run: `npm run test -- --testPathPattern=useIntakesTable` → expected pass.

---

**🔵 REFACTOR**

- Extract a `buildIntakesUrlParams(current: URLSearchParams, overrides: IntakesUrlOverrides): URLSearchParams` pure function above the hook — makes each setter a one-liner and keeps the hook body readable.
- Define `type IntakesUrlOverrides` for the override shape (no `any`).
- Confirm pagination is 0-based on the URL (matches orders tracker convention).
- Run: `npm run test && npm run types:check && npm run lint:fix` from `apps/web/`.

**Commit**: `[MLID-2225] - refactor(intakes): URL-serialize all filter state in useIntakesTable`

---

### Step 4 — Wire `usePersistedRouteQuery` into Intakes page

**Branch state**: Step 3 merged into feature branch.

**Files**:
- Modify: `apps/web/app/intakes/IntakesPageNew.tsx`
- Modify (or create if test file doesn't exist): `apps/web/app/intakes/IntakesPageNew.test.tsx`

---

**🔴 RED — failing test**

- Test file: `apps/web/app/intakes/IntakesPageNew.test.tsx`
- Confirm the file exists (glob found `IntakesPageNew.tsx` — check for corresponding test). If none exists, create it.
- Mock setup mirrors `page.test.tsx` pattern — mock `useIntakesTable`, `usePersistedRouteQuery`, `next/navigation`.
  ```ts
  const mockPersistedClear = jest.fn();
  jest.mock('@/utils/hooks/usePersistedRouteQuery', () => ({
    usePersistedRouteQuery: jest.fn(() => ({ clear: mockPersistedClear })),
  }));
  ```
- Test cases:
  - `"usePersistedRouteQuery is called with 'intakesLastQueryString'"` — render `IntakesPageNew`; assert `usePersistedRouteQuery` called with `'intakesLastQueryString'`.
  - `"clearAllFilters calls persisted.clear()"` — render; find Clear All button (or call `clearAllFilters` captured from mock); assert `mockPersistedClear` called once.

- Run: `npm run test -- --testPathPattern=IntakesPageNew` → expected failure: hook not imported.

---

**🟢 GREEN — minimal implementation**

- File: `apps/web/app/intakes/IntakesPageNew.tsx`
- Changes:
  1. Import `usePersistedRouteQuery` from `@/utils/hooks`.
  2. Call: `const persistedQueryString = usePersistedRouteQuery('intakesLastQueryString');`
  3. In `clearAllFilters` (line 124): add `persistedQueryString.clear();` before or after the existing state resets.

- Run: `npm run test -- --testPathPattern=IntakesPageNew` → expected pass.

---

**🔵 REFACTOR**

- No structural changes needed — the hook call is a single line.
- Run: `npm run test && npm run types:check && npm run lint:fix` from `apps/web/`.

**Commit**: `[MLID-2225] - feat(intakes): wire usePersistedRouteQuery for cross-session filter restore`

---

### Step 5 (Bonus) — Persist column ordering on Orders Tracker

**Branch state**: Steps 1–4 merged into feature branch.

**Files**:
- Modify: `apps/web/app/orders-tracker/Orders/Orders.tsx`
- Modify: `apps/web/app/orders-tracker/Orders/Orders.test.tsx`

This step is cleanly skippable. If scope creeps, stop after Step 4. The bonus does not affect any of the previous steps.

---

**🔴 RED — failing test**

- Test file: `apps/web/app/orders-tracker/Orders/Orders.test.tsx`
- Extend the existing `@repo/ui` DataGrid mock to capture `onColumnOrderChange` and `initialState`:
  ```ts
  let capturedOnColumnOrderChange: ((params: unknown) => void) | null = null;
  let capturedInitialState: { columns?: { orderedFields?: string[] } } | null = null;
  // Inside mock DataGrid:
  capturedOnColumnOrderChange = props.onColumnOrderChange ?? null;
  capturedInitialState = props.initialState ?? null;
  ```
- Test cases:
  - `"persists column order to localStorage when onColumnOrderChange fires"` — render `Orders` with `orderCategory='new'`; retrieve `capturedOnColumnOrderChange`; call it with a mock `GridApiPro` that returns `['col1', 'col2']` from `getAllColumns()`; assert `localStorage.getItem('ordersTrackerColumnOrder-new')` equals `JSON.stringify(['col1', 'col2'])`.
  - `"reads saved column order from localStorage and passes as initialState.columns.orderedFields"` — seed `localStorage.setItem('ordersTrackerColumnOrder-new', JSON.stringify(['col2', 'col1']))`; render; assert `capturedInitialState?.columns?.orderedFields` equals `['col2', 'col1']`.
  - `"falls back to undefined orderedFields when localStorage is empty"` — render with clean localStorage; assert `capturedInitialState?.columns?.orderedFields` is `undefined`.
  - `"handles malformed JSON in localStorage without throwing"` — seed `'not-json'`; render; assert no thrown error and `capturedInitialState?.columns?.orderedFields` is `undefined`.

- Run: `npm run test -- --testPathPattern="Orders/Orders"` → expected failure: `onColumnOrderChange` not wired, `initialState` not set.

---

**🟢 GREEN — minimal implementation**

- File: `apps/web/app/orders-tracker/Orders/Orders.tsx`
- Changes (inside the `Orders` component, after the existing `columnVisibilityStorageKey` pattern):
  1. Derive storage key: `` const columnOrderStorageKey = `ordersTrackerColumnOrder-${orderCategory}`; ``
  2. Read on mount (same two-effect Pattern A as column visibility):
     ```ts
     const [columnOrderedFields, setColumnOrderedFields] = useState<string[] | undefined>(undefined);
     const [hasLoadedColumnOrder, setHasLoadedColumnOrder] = useState(false);
     useEffect(() => {
       try {
         const raw = window.localStorage.getItem(columnOrderStorageKey);
         if (raw) setColumnOrderedFields(JSON.parse(raw));
       } catch { /* ignore */ } finally { setHasLoadedColumnOrder(true); }
     }, [columnOrderStorageKey]);
     ```
  3. Write on change:
     ```ts
     useEffect(() => {
       if (!hasLoadedColumnOrder) return;
       if (!columnOrderedFields) return;
       try { window.localStorage.setItem(columnOrderStorageKey, JSON.stringify(columnOrderedFields)); }
       catch { /* ignore */ }
     }, [columnOrderedFields, columnOrderStorageKey, hasLoadedColumnOrder]);
     ```
  4. Wire `onColumnOrderChange` on the `DataGrid`:
     ```ts
     onColumnOrderChange={() => {
       if (!gridApiRef.current) return;
       const fields = gridApiRef.current.getAllColumns().map(c => c.field);
       setColumnOrderedFields(fields);
     }}
     ```
  5. Pass `initialState={{ columns: { orderedFields: columnOrderedFields } }}` to `DataGrid`.

- Run: `npm run test -- --testPathPattern="Orders/Orders"` → expected pass.

---

**🔵 REFACTOR**

- Confirm `initialState` is only passed when `hasLoadedColumnOrder` is true — otherwise the initial render with `undefined` fields may conflict with the DataGrid's own default order. Conditionally spread: `...(hasLoadedColumnOrder && columnOrderedFields ? { initialState: { columns: { orderedFields: columnOrderedFields } } } : {})`.
- Confirm `GridApiPro` type is already imported (it is — line 21).
- Run: `npm run test && npm run types:check && npm run lint:fix` from `apps/web/`.

**Commit**: `[MLID-2225] - feat(orders-tracker): persist column drag order to localStorage`

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/utils/hooks/usePersistedRouteQuery.ts` | Create | Shared hook: mirrors URL QS to localStorage, restores on empty-URL entry, exposes `clear()` |
| `apps/web/utils/hooks/usePersistedRouteQuery.test.ts` | Create | Unit tests for all hook behaviors |
| `apps/web/utils/hooks/index.ts` | Modify | Export `usePersistedRouteQuery` |
| `apps/web/app/orders-tracker/[category]/page.tsx` | Modify | Derive per-category storage key; call `usePersistedRouteQuery`; call `persisted.clear()` in `clearFilters` |
| `apps/web/app/orders-tracker/[category]/page.test.tsx` | Modify | Mock `usePersistedRouteQuery`; assert `clear()` called on filter clear; assert correct storage key per tab |
| `apps/web/app/intakes/hooks/useIntakesTable.ts` | Modify | URL-serialize all filter state; fix stale MLID-2125 comment at lines 295–297 |
| `apps/web/app/intakes/hooks/useIntakesTable.test.tsx` | Modify | Tests for URL read on mount and URL write on setter calls for all filter fields |
| `apps/web/app/intakes/IntakesPageNew.tsx` | Modify | Call `usePersistedRouteQuery('intakesLastQueryString')`; call `persisted.clear()` in `clearAllFilters` |
| `apps/web/app/intakes/IntakesPageNew.test.tsx` | Create/Modify | Assert hook called with correct key; assert `clear()` called on filter clear |
| `apps/web/app/orders-tracker/Orders/Orders.tsx` | Modify (Bonus Step 5) | Read/write column order from localStorage; wire `onColumnOrderChange`; pass `initialState.columns.orderedFields` |
| `apps/web/app/orders-tracker/Orders/Orders.test.tsx` | Modify (Bonus Step 5) | Assert localStorage read seeds `orderedFields`; assert drag fires write |
| `apps/web/app/orders-tracker/hooks/useGridUrlModels.ts` | Modify (Post-impl Fix 2) | `buildQs` reads URL state from `window.location` via new `liveStateFromUrl()` helper instead of stale `stateRef.current` |

---

## Testing Strategy

**Unit tests**:
- `usePersistedRouteQuery.test.ts`: version mismatch discards entry; malformed JSON discards entry; empty URL triggers restore; non-empty URL skips restore; `clear()` removes entry; write effect fires when searchParams change.
- `useIntakesTable.test.tsx`: all six filter fields read from URL on mount; each setter writes the correct URL param; `debouncedSearch` (not raw) written to URL; `handleTabChange` strips all filter params.

**Integration behavior** (no separate integration test needed — covered by unit tests with jsdom localStorage and mocked `router.replace`):
- `page.test.tsx`: `usePersistedRouteQuery` called with correct key per tab; `clear()` called by `clearFilters`.
- `IntakesPageNew.test.tsx`: same shape.

**Manual verification** (see section below).

**Coverage**: `usePersistedRouteQuery.ts` — 100% (all branches exercised by the seven test cases). `useIntakesTable.ts` — existing coverage maintained; new URL paths covered by new test cases. `Orders.tsx` column order — 100% of the localStorage read/write code paths covered.

---

## Security Considerations

- **Input validation**: `readEntry` validates `version` is a number and `queryString` is a string before use. Malformed JSON and wrong-version entries are silently discarded.
- **Authorization**: No authorization concerns — this is a purely client-side UX feature. No server calls added.
- **Feature flags**: None — ships directly, no flag needed.
- **PHI handling**: The URL query string for orders tracker already contains only filter parameters (field names, enum values, sort fields, search text). No PHI is stored in the query string. The intakes URL will contain `search=<text>` which could be a patient name substring — this is the same data already in the URL during an active session and already visible in the browser address bar. The storage is per-browser, same caveat as existing column-visibility persistence. No additional PHI exposure beyond what is already present.
- **localStorage is per-browser, not per-user**: if multiple users share a machine, one user's filters will be restored for the next user until they clear. This is the same limitation as the existing column-visibility keys and is accepted.

---

## Manual Verification

Perform these steps after Step 4 (and again after Step 5 if the bonus is included):

**Orders Tracker — filter restore**
1. Navigate to `/orders-tracker/new`.
2. Set a column filter (e.g., Status = "Urgent") and a quick search value.
3. Navigate to a patient detail page (or any other page).
4. Use browser Back or navigate directly back to `/orders-tracker/new`.
5. Verify the filter and search are restored — the URL should contain the same params as step 2.
6. Verify the data grid shows the filtered results.

**Orders Tracker — clear persists**
1. While filters are active (from above), click "Clear Filters".
2. Navigate away and return to `/orders-tracker/new`.
3. Verify the page loads with no filters (empty URL, no filter chips).

**Orders Tracker — tab switching**
1. Set filters on the "New" tab.
2. Click the "Maintenance" tab, set different filters.
3. Navigate away entirely, then return to `/orders-tracker/new`.
4. Verify "New" tab filters are restored.
5. Switch to "Maintenance" tab — verify maintenance filters are also restored.

**Orders Tracker — storage wins over URL (priority flip)**
1. Save filter A on `/orders-tracker/new` (set status filter, navigate away, come back to confirm restored).
2. Manually paste a URL with different explicit params: `/orders-tracker/new?filter={"items":[{"field":"status","operator":"is","value":"Urgent"}]}`.
3. Verify the page loads with **filter A** (the user's saved value), not the explicit URL filter.
4. URL in the address bar should be rewritten to filter A's query string.

**Intakes — filter restore**
1. Navigate to `/intakes`.
2. Select a location filter, set a search query, change the tab to "Processing".
3. Navigate to another page and return to `/intakes`.
4. Verify the same tab, location filter, and search query are restored.

**Intakes — clear persists**
1. While filters are active, click "Clear All".
2. Navigate away and return to `/intakes`.
3. Verify the page loads with no filters.

**Bonus Step 5 — column ordering**
1. Navigate to `/orders-tracker/new`.
2. Drag a column to a new position.
3. Reload the page (hard reload, not SPA navigation).
4. Verify the column appears in the position it was dragged to.
5. Verify other tabs are unaffected (drag a column on "Maintenance" and confirm "New" still shows its order).

---

## Decisions

The following questions were raised during planning and resolved before implementation:

1. **localStorage envelope shape** — Storage keys stay **independent** of the existing column-visibility keys. The new `{ version: 1, queryString: '...' }` envelope is namespaced under `ordersTrackerLastQueryString-{cat}` / `intakesLastQueryString`, separate from `ordersTrackerColumnVisibilityModel-{cat}` and `intakesColumnVisibilityModel`. No migration of existing visibility data.

2. **Bonus column ordering — separate vs. merged storage key** — **Separate keys**: `ordersTrackerColumnOrder-new`, `-maintenance`, `-lead`. Visibility keys remain unchanged.

3. **Intakes URL pagination convention** — **0-based** on the URL, matching the orders tracker convention (`useGridUrlModels.ts` line 104). The existing API conversion (`page + 1` inside `buildParams`) is unchanged.

4. **Intakes array encoding** — **Comma-separated** (`loc=A,B`, `types=Type1,Type2`). Shorter, URL-friendly, consistent with the `tab=` style. Empty values strip the param entirely.

5. **`sortModel` and `filterModel` on intakes URL** — **JSON-encoded** matching the orders tracker convention (`sort=<JSON>`, `filter=<JSON>`). Stored and restored transparently by `usePersistedRouteQuery` without needing to understand their shape.

---

## Post-implementation Fixes

Two adjustments were made after Steps 1–5 were merged into the feature branch and the user manually verified the flow in the browser. Both stay in scope for MLID-2225.

### Fix 1 — Priority flip: storage wins over URL on entry

**Trigger.** Manual testing surfaced a UX question: if the user sets filters today, navigates away, and comes back tomorrow, they expect the same filters back — regardless of what's currently in the address bar. The original Step 1 design treated the URL as authoritative ("if URL has any params on entry, skip restore — URL wins"), which only restored on entry when the URL was completely empty. Product call from the user: localStorage is the user's intent; URL is just a transient view. Storage should win on every entry.

**What changed.** `usePersistedRouteQuery.ts` Effect 1 was inverted. It no longer skips restore when the URL is non-empty. Instead, on every mount, it reads storage and — if storage holds a non-empty `queryString` and that value differs from the current URL — calls `router.replace` to push the saved value onto the URL.

```ts
useEffect(() => {
  if (hasRestoredRef.current) return;
  const entry = readEntry(storageKey);
  if (entry && entry.queryString) {
    const currentQueryString = searchParams?.toString() ?? '';
    if (entry.queryString !== currentQueryString) {
      // ... call router.replace (see Fix 2 for the synchronous variant)
    }
  }
  hasRestoredRef.current = true;
}, []);
```

**Trade-off.** Bookmarked or shared URLs with explicit filters get **overwritten** by the user's saved state on entry. If users paste filtered URLs to each other, the recipient sees their own saved filters instead. Accepted by the user — see the JSDoc on the hook for the documented behavior.

**Test changes.** The original test `"should not restore when URL already has params (deep link protection)"` was renamed to `"restores saved query string even when URL already has different params (storage wins over URL)"` and its assertion inverted (`mockReplace` IS called now). A new test was added: `"does not call router.replace when storage matches the current URL exactly"` — guards against unnecessary navigation when storage and URL already agree.

**Manual verification.** The "Orders Tracker — storage wins over URL (priority flip)" entry in the Manual Verification section reflects this change.

### Fix 2 — Wipe bug: flush race against external `router.replace`

**Trigger.** With Fix 1 in place, manual testing exposed a separate bug. Reproduction:

1. Set `orderType is "New Start"` filter on `/orders-tracker/new`. URL and localStorage both show `?sort=...&filter=...`.
2. Navigate to `/intakes`.
3. Click "Order Tracker" in the main menu. Lands on `/orders-tracker/new` (no query params).

Expected: filter restored from storage, URL shows `?sort=...&filter=...`. Actual: URL ended at `?sort=...` (filter dropped) and localStorage was clobbered to match.

**Root cause.** A subtle race between three things on the same render cycle:

1. `usePersistedRouteQuery`'s Effect 1 calls `router.replace(?sort=...&filter=...)`. In the Next.js App Router, `router.replace` is wrapped in a transition and is **deferred** — it does not synchronously update `window.location.search`.
2. Mount-time effects in `page.tsx` call `updateModels` with a single field — most notably `setSearch('')` from the debounce effect at line 127, plus DataGrid's mount-time `onSortModelChange` emission. Each call accumulates into `pendingRef.current`.
3. `useGridUrlModels`'s `scheduleFlush` runs as a microtask. `buildQs` reads `stateRef.current` to fill in fields not in `overrides`. But `stateRef.current` was set during Render 1 (URL empty) — it lags one render behind any external `router.replace`.

The flush rebuilds the URL using stale `stateRef` values (URL-empty defaults: `filterModel = {items:[]}`, `sortModel = DEFAULT_SORT`). Since `stateRef.filterModel` is empty and matches the default, the `filter=` param is dropped from the rebuilt query string. The compare-and-replace check sees the rebuilt QS differs from `window.location.search` (which is also still empty because Effect 1's `router.replace` hasn't propagated), so a fresh `router.replace` is issued — and that one *does* eventually win, leaving the URL with only `?sort=...` and no filter. Effect 2 of `usePersistedRouteQuery` then mirrors the wiped URL back into localStorage on the next render, so the saved value is lost.

**Diagnostic confirmation.** A temporary `console.log` inside the flush microtask printed `window.location.search` along with `liveStateFromUrl()`. The output showed `window.location.search = ''` at flush time even after Effect 1 had called `router.replace` with the restored URL — confirming `window.location` had not yet been updated when the flush ran.

**Two-part fix.**

**(a)** `usePersistedRouteQuery.ts` Effect 1 now calls `window.history.replaceState(null, '', target)` **synchronously** alongside `router.replace(target)`. The browser URL updates immediately, so any code reading `window.location.search` inside the same microtask flush sees the restored value. `router.replace` still runs to keep Next.js's router state in sync for `useSearchParams()` so subsequent renders observe the new URL.

```ts
const target = `${pathname}?${entry.queryString}`;
if (typeof window !== 'undefined') {
  window.history.replaceState(null, '', target);
}
router.replace(target);
```

**(b)** `useGridUrlModels.ts` `buildQs` no longer reads from `stateRef.current`. A new module-internal helper `liveStateFromUrl()` parses the current `window.location.search` directly and returns the live state. `buildQs` uses that as its base, applying `overrides` on top. This makes the flush self-correcting against any external URL mutation, regardless of when React's render cycle catches up.

```ts
const liveStateFromUrl = useCallback(() => {
  if (typeof window === 'undefined') {
    return { paginationModel: DEFAULT_PAGINATION, sortModel: DEFAULT_SORT, ... };
  }
  const params = new URLSearchParams(window.location.search);
  return {
    paginationModel: { page: ..., pageSize: ... },
    sortModel: safeJsonParse(params.get('sort'), DEFAULT_SORT),
    filterModel: safeJsonParse(params.get('filter'), DEFAULT_FILTER),
    search: params.get('search') ?? '',
    activeQuickFilter: params.get('quickFilter'),
  };
}, []);

const buildQs = useCallback((overrides) => {
  const s = liveStateFromUrl();
  // ... apply overrides on top of s and emit URLSearchParams
}, [liveStateFromUrl]);
```

Both pieces are necessary. Without (a), `liveStateFromUrl` would still see an empty URL because `router.replace` is deferred. Without (b), even a synchronously-updated `window.location` would not be consulted because the flush would still be reading `stateRef`.

**Why this is in scope.** The race only manifested because Step 1 of MLID-2225 introduced the new pathway where an external hook (`usePersistedRouteQuery`) mutates the URL between renders. `useGridUrlModels` was previously the sole writer of URL state, so `stateRef` and `window.location` could never disagree. Reading from `window.location` instead of `stateRef` is the minimum change that keeps the flush self-consistent under the new architecture.

**Test changes.** A regression test was added in `usePersistedRouteQuery.test.tsx`:

```ts
it('updates window.location synchronously during restore so flushed URL writes from other hooks see the restored state', () => {
  // Spy on window.history.replaceState; assert it is called with the restored URL.
});
```

This pins down the synchronous `history.replaceState` call so a future refactor that drops it will fail loudly.

**Manual verification.** Reproducing the original flow (set filter → navigate to `/intakes` → return via main menu) now leaves the URL and localStorage intact. Confirmed in browser by the user.

### Fix 3 — Architecture refactor: granular per-key URL writes + DataGrid mount-emit guard

**Trigger.** During post-Fix-2 manual verification, two related issues surfaced:

1. **Pre-existing default-sort write on /orders-tracker.** Visiting `/orders-tracker/new` with a clean URL would auto-populate `?sort=[{received,desc}]` on every entry — the DataGrid emits a single-entry sort on mount and `useGridUrlModels.buildQs` writes it (compares to `DEFAULT_SORT`'s 2-entry tiebreaker, finds them different). With persistence layered on top, this auto-write started clobbering the user's saved sort on every visit.
2. **Architecture concern.** Both `useGridUrlModels.buildQs` and `usePersistedRouteQuery` treated the query string as one opaque string blob. Each rebuild from scratch could silently drop any URL param outside its vocabulary — and the two writers raced each other on every mutation.

The user pushed for an architectural fix: each writer should mutate **only the keys it owns** and treat the rest as opaque pass-through. Three layers, one direction of authority.

**What changed.** Implemented per the dedicated plan in [`docs/agomez/plans/MLID-2225-url-architecture-refactor.md`](MLID-2225-url-architecture-refactor.md). High-level summary:

- **Layer 1 — `apps/web/utils/hooks/urlPersistence.ts`** (new). Pure storage primitives: `loadStoredQueryString`, `saveStoredQueryString`, `clearStoredQueryString`. Versioned envelope, all `localStorage` exceptions swallowed.
- **Layer 2 — `apps/web/utils/hooks/useUrlParams.ts`** (new). Generic URL manager. `setParams(patch)` patches only the listed keys via `URLSearchParams.set`/`delete` on a fresh copy of `window.location.search`. Multiple synchronous calls batch into one microtask-flushed `router.replace`. Seeds URL from storage on mount via synchronous `history.replaceState` + `router.replace` (storage wins over URL — preserves Fix 1). Mirrors URL → storage on every observed `searchParams` change.
- **Layer 3 — domain hooks refactored.** `useGridUrlModels` becomes a thin wrapper of `useUrlParams` — `buildQs`/`liveStateFromUrl`/`stateRef`/`pendingRef` (~165 lines) are deleted. Each setter calls `setParams` with a one-key patch. `useIntakesTable` adopts the same model, dropping its local `writeUrl`/`buildIntakesUrlParams` rebuilder.
- **`usePersistedRouteQuery` deleted.** Its responsibilities split across the two new modules.
- **Page wiring simplified.** `orders-tracker/[category]/page.tsx` and `intakes/IntakesPageNew.tsx` no longer call `usePersistedRouteQuery` directly. Persistence is now an internal detail of the domain hook (`useGridUrlModels(\`ordersTrackerLastQueryString-${tab}\`)` / `useIntakesTable({ storageKey })`).

**DataGrid mount-emit guard (added during verification of this refactor).** With granular writes in place, the user reported a still-present symptom on `/orders-tracker/new`: setting a 2-item filter, navigating away, returning, and hard-refreshing left only one item in the URL and storage. Browser console traces (added temporarily inside `setParams`) pinpointed MUI X's `useGridFilter.js:217` mount-time normalization emitting `onFilterModelChange` with one item silently dropped, on a render cycle after the existing `hasMounted` ref had flipped to `true`. The handler forwarded the truncated value through `setParams`, the URL was wiped, and the mirror saved the wipe to storage.

Fix in `[category]/page.tsx`'s `handleFilterModelChange`:

```ts
if (m.items.length < filterModel.items.length) return;
```

Reject emits that strictly reduce item count — that's normalization noise, not user intent. **Trade-off:** the DataGrid filter panel's per-row trash icon stops working as a removal path; users remove via the active-filter chip's `onDelete` (`removeFilterItem`), which bypasses `handleFilterModelChange` entirely. Chips are always rendered when filters are active, so the chip-removal UX is the primary path users see.

The diagnostic `console.trace`/`console.log` calls were removed before commit. **No dedicated test was added** — `[category]/page.test.tsx` mocks `useGridUrlModels` wholesale, so simulating MUI X's mount-time `onFilterModelChange` cascade through the real component tree is non-trivial. Flagged as a follow-up in the PR doc.

**Test infra adjustments** (came up during refactor execution; documented in the dedicated refactor plan's "Post-implementation adjustments" section):

- `useUrlParams` mirror reads `window.location` rather than `searchParams.toString()` (the latter lags one render after seed).
- `useUrlParams` uses `routerRef`/`pathnameRef` to keep `setParams` referentially stable when test mocks return a fresh `useRouter()` object every render.
- `useIntakesTable.test.tsx`'s `setupUrlTracking` helper syncs `window.location` alongside the searchParams mock.
- 3 intakes setter tests use `await act(async)` to flush the microtask queued by `setParams` before asserting on the captured URL.
- `orders-tracker/[category]/page.test.tsx` adds `jest.restoreAllMocks()` to `beforeEach` so spy overrides from earlier describes don't leak into the new persistence assertions.
- The intakes page test's persistence describe was deleted (not modified) — the contracts it tested are now covered at the hook level by `useIntakesTable.test.tsx` and `useUrlParams.test.tsx`.

**Manual verification.** Reproducing the user's full flow (set 2 filters → navigate to `/intakes` → return via main menu → hard refresh → repeat) now leaves both filters intact in URL and storage.

**Commit.** `29527049` — `[MLID-2225] - refactor(url-state): three-layer architecture for filter persistence`. Net +217 / −640 (~420 lines deleted overall, mostly the `usePersistedRouteQuery` hook and the `useGridUrlModels` rebuild scaffolding).

### Fix 4 — Intakes seed-race / mount-time wipe

**Trigger.** After Fix 3 landed, manual verification on `/intakes` exposed a separate bug, distinct from the orders-tracker one. Reproduction:

1. Configure 2 filters on `/intakes`. URL and localStorage both show `?sort=[]&filter={items: [type=fax, status=w]}`.
2. Navigate to `/users-locations`.
3. Return to `/intakes` via the main menu (clean URL).
4. Filters not restored in the UI; storage reduced to `?sort=[]` only.

**Root cause.** Two compounding bugs in `useIntakesTable`, both rooted in its choice of `useState` (rather than `useMemo` derived from `searchParams`):

1. **Local state stuck on empty.** The `useState` initializers run once at mount with the (then-empty) URL. When `useUrlParams.seed` populated the URL afterward, the local `filterModel` / `searchQuery` / etc. never re-synced. The DataGrid and filter chips rendered as "no filters" even when the URL had them.
2. **Mount-time URL wipe.** The search and filter `useEffect` debounces scheduled 300ms timers on mount with the captured empty mount-state values. 300ms later they fired `setParams({search: null, filter: null})` — deleting whatever the seed had just put in the URL. The mirror effect then saved the wiped URL back to storage. That's why the user observed storage reduced to just `sort=[]`.

`useGridUrlModels` doesn't have this bug because it uses `useMemo` derived from `searchParams` — auto-resyncs on URL changes, no useState initializers to capture stale mount state.

**Fix in `useIntakesTable.ts`:**

1. **Skip-first-tick guards** (`isFirstSearchDebounceRef`, `isFirstFilterDebounceRef`) on the two debounce useEffects — the empty mount-state is never written to URL on a 300ms timer. First user keystroke triggers normal debounce.
2. **Per-key URL→state sync useEffects** for `loc`, `types`, `search` (+ `debouncedSearch`), `page`/`pageSize`, `sort`, `filter` (+ `debouncedFilterModel`). Each watches its URL string; when it changes externally (e.g. after seed), local state catches up via functional `setState` (skips re-render when the value already matches).

**Why local state + sync useEffects rather than refactoring to `useMemo` derivation** (matching `useGridUrlModels`): pragmatic choice to keep the existing test surface and the typing UX for the search input without a larger refactor. Full convergence on the `useMemo` pattern is a clean follow-up.

**Tests added** to `useIntakesTable.test.tsx`:

- `"syncs local state when URL changes externally after mount (simulates useUrlParams seed)"` — mounts with empty URL, swaps the `searchParams` mock to a populated URL, asserts local state catches up.
- `"does not write empty mount-state to URL on the first debounce tick"` — mounts with a tab-only URL, advances 1000ms with no user input, asserts `mockReplace` was never called and the URL is unchanged.

**Manual verification.** Repeating the trigger flow now leaves URL and storage intact across menu navigation. Confirmed in browser by the user.

**Commit.** `bc1b62c1` — `[MLID-2225] - fix(intakes): preserve URL state across mount-time seed by useUrlParams`. Net +169 lines (97 in the hook, 70 in tests).

### Fix 5 — Right-align active filter chips on /intakes

**Trigger.** Post-Fix-4 visual review: the filter chip row on `/intakes` (location, type, advanced filter chips under `IntakesFilterBar`) was left-aligned, diverging from the `/orders-tracker` chip layout that the UI designer specified as the reference.

**Fix in `IntakesFilterBar.tsx`.** Added `justifyContent: 'flex-end'` to the chip-row flexbox `<Box>`'s `sx`. Orders-tracker uses `<Layout justify="end">`, which renders the same flexbox property; this one-line change brings intakes into parity. No test changes (existing tests assert on chip presence via testid, not layout).

**Commit.** `e4519448` — `[MLID-2225] - fix(intakes): right-align active filter chips per UI designer spec`. +12 / −2.

---

## Final commit history (this branch → develop)

| Hash | Message |
|------|---------|
| `dbeb1b8b` | `feat(filter-persistence): cross-session filter and column-order restore` (Steps 1–5 of the original plan + Fixes 1 & 2) |
| `29527049` | `refactor(url-state): three-layer architecture for filter persistence` (Fix 3 — architecture refactor + DataGrid mount-emit guard) |
| `bc1b62c1` | `fix(intakes): preserve URL state across mount-time seed by useUrlParams` (Fix 4) |
| `e4519448` | `fix(intakes): right-align active filter chips per UI designer spec` (Fix 5) |

PR doc: [`docs/agomez/PR/MLID-2225.md`](../PR/MLID-2225.md). Refactor plan (Fix 3 details): [`docs/agomez/plans/MLID-2225-url-architecture-refactor.md`](MLID-2225-url-architecture-refactor.md).
