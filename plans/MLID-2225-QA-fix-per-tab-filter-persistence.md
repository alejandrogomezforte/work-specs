# [MLID-2225] QA fix — per-tab filter persistence on /intakes

> **Status:** Done — committed `3698672a8`, PR doc at `docs/agomez/PR/MLID-2225-QA-fix.md`
> **Branch:** `hotfix/MLID-2225-per-tab-filter-persistence` (off `develop`)
> **Base:** `develop` (the original feature already merged via PR #1530)
> **Approach:** C (per-tab storage keys, scoping logic in `useIntakesTable`; `useUrlParams` untouched)

---

## Problem

QA rejected MLID-2225 because filter persistence on `/intakes` doesn't survive a tab change.

Repro:
1. On `/intakes?tab=Needs Review`, apply any filter (location, type, search, advanced filter).
2. Click another tab (e.g. *Processed*).
3. Click back to *Needs Review*.
4. **Expected:** filters from step 1 are restored.
5. **Actual:** filters are gone.

### Root cause

Today `/intakes` uses a single storage key `intakesLastQueryString` and `handleTabChange` wipes every owned URL key before switching tabs:

```ts
setParams({ tab, loc: null, types: null, search: null, page: null, pageSize: null, sort: null, filter: null });
```

`useUrlParams`'s mirror effect then writes that wiped URL into storage, so the previous tab's state is **destroyed**, not hidden. Coming back to the tab has nothing to restore.

Orders-tracker avoids this because each tab is a separate path (`/orders-tracker/new|maintenance|lead`) with its own page-mount and its own storage key (`ordersTrackerLastQueryString-<tab>`).

Aligning the URL shape between intakes and orders-tracker is a follow-up the PO is aware of — out of scope for this fix.

---

## Approach C — per-tab storage keys in `useIntakesTable`

Stop letting `useUrlParams` manage storage for intakes. Instead, `useIntakesTable` owns per-tab storage explicitly. Each tab gets its own bucket; `useUrlParams` stays a generic URL manager with no domain knowledge.

### Decisions taken

1. **Default tab on cold entry** — when the user lands on `/intakes` with no `?tab=…`, default to *Needs Review* (current behavior). We do **not** remember "last active tab" across full navigations. Keeps the URL the source of truth for which tab.
2. **Debounce flush on tab change** — flush pending search/filter debounces into the old tab's bucket before switching, so a half-typed search isn't lost on tab switch.
3. **Storage wins over URL still applies per tab** — pasting `/intakes?tab=Processed&loc=xyz` is overwritten by the Processed bucket. Same trade-off the original PR accepted, scoped to one tab.
4. **No migration of the old single-key entry** — `intakesLastQueryString` (the pre-fix key) is abandoned. Bucket entries are populated fresh as users interact. The orphan key sits harmlessly in localStorage for users who used the broken feature.
5. **Storage key shape** — `intakesLastQueryString-<IntakeTab>` (e.g. `intakesLastQueryString-Needs Review`). Spaces are fine in localStorage keys. Matches the orders-tracker naming pattern at the prefix level.

---

## Architecture

```
LocalStorage (one key per tab)
   intakesLastQueryString-Needs Review
   intakesLastQueryString-Processed
   intakesLastQueryString-System Processing
   intakesLastQueryString-Failed
   intakesLastQueryString-Skipped
   intakesLastQueryString-All
        │
        ▼ (seed for the *current* tab on mount; re-seed on tab change)
URL params (single page mount, `tab` is just another param)
        │
        ▼ (parse via useMemo / per-key sync useEffects — already exists)
UI state
        │
        ▲ (user change)
        │
URL params  ── granular setParams patches (already exists)
        │
        ▼ (mirror to bucket for *current* tab on every change)
LocalStorage
```

`useUrlParams` continues to provide `setParams` / `clearOwnedKeys` / `getQueryString`. The `storageKey` arg is **not** passed in from `useIntakesTable` anymore (or is explicitly `undefined`) — so the seed and mirror inside `useUrlParams` are dormant. All storage work moves into `useIntakesTable`.

---

## Implementation plan

### Files to change

| File | Change | Notes |
|------|--------|-------|
| `apps/web/app/intakes/hooks/useIntakesTable.ts` | Modify | Drop `storageKey` from being forwarded to `useUrlParams`. Add per-tab seed, mirror, and tab-change snapshot logic using `urlPersistence` primitives directly. Add debounce flush helpers used by `handleTabChange`. |
| `apps/web/app/intakes/IntakesPageNew.tsx` | No change | Continues to call `useIntakesTable({ storageKey: 'intakesLastQueryString' })`. The option becomes a *prefix* internally. |
| `apps/web/app/intakes/hooks/useIntakesTable.test.tsx` | Modify | Add per-tab persistence tests (see Test Plan). Update any existing test that asserted single-key behavior. |

`useUrlParams.ts`, `urlPersistence.ts`, `useGridUrlModels.ts`, and the orders-tracker page are untouched.

### Detailed changes inside `useIntakesTable.ts`

1. **Stop passing `storageKey` to `useUrlParams`.**
   ```ts
   const { setParams } = useUrlParams(); // no arg — generic URL manager only
   ```
   Keep `options.storageKey` as the option name (so the page-level call site doesn't change), but treat it as a **prefix** internally:
   ```ts
   const storagePrefix = options?.storageKey; // e.g. "intakesLastQueryString"
   const keyForTab = useCallback(
     (tab: IntakeTab) => (storagePrefix ? `${storagePrefix}-${tab}` : null),
     [storagePrefix]
   );
   ```

2. **Mount-time seed for the current tab.**
   Use one `useEffect` gated by a `hasSeededRef`. Read the URL's `tab` (validated via the existing `validTabFromUrl` derivation), load the bucket for that tab, and push every owned key into `setParams`. The existing per-key sync `useEffect`s in this hook will rehydrate local state.
   ```ts
   const hasSeededRef = useRef(false);
   useEffect(() => {
     if (hasSeededRef.current) return;
     hasSeededRef.current = true;
     const key = keyForTab(validTabFromUrl);
     if (!key) return;
     const stored = loadStoredQueryString(key);
     if (!stored) return;
     const currentQs = window.location.search.replace(/^\?/, '');
     if (stored === currentQs) return;
     const params = new URLSearchParams(stored);
     setParams({
       tab: validTabFromUrl,
       loc: params.get('loc'),
       types: params.get('types'),
       search: params.get('search'),
       page: params.get('page'),
       pageSize: params.get('pageSize'),
       sort: params.get('sort'),
       filter: params.get('filter'),
     });
   }, []); // mount-only, ESLint disabled with comment
   ```
   `setParams(null)` deletes the key, so absent values in the bucket cleanly clear stale URL values.

3. **Mirror URL → current tab's bucket.**
   ```ts
   useEffect(() => {
     if (!hasSeededRef.current) return; // never wipe before seeding
     const key = keyForTab(activeTab);
     if (!key) return;
     const currentQs = window.location.search.replace(/^\?/, '');
     saveStoredQueryString(key, currentQs);
   }, [searchParams, activeTab, keyForTab]);
   ```
   Reads from `window.location` (not `searchParams`) for the same reason as the existing architecture — `useSearchParams` lags one render after a `router.replace`.

4. **Tab change: flush, snapshot, restore.**
   Rewrite `handleTabChange` to be per-tab aware. Pseudocode:
   ```ts
   const handleTabChange = useCallback((newTab: IntakeTab) => {
     if (newTab === activeTab) return;

     // 1. Flush pending search/filter debounces so unsaved keystrokes land in the
     //    old tab's bucket. Use clearTimeout + apply value immediately.
     if (searchTimerRef.current) {
       clearTimeout(searchTimerRef.current);
       setDebouncedSearch(searchQuery);
       setParams({ search: searchQuery ? searchQuery : null });
     }
     if (filterTimerRef.current) {
       clearTimeout(filterTimerRef.current);
       setDebouncedFilterModel(filterModel);
       setParams({ filter: isEmptyFilter(filterModel) ? null : JSON.stringify(filterModel) });
     }

     // 2. Snapshot current URL into old tab's bucket (mirror usually has this,
     //    but defensive — the setParams microtask above hasn't flushed yet).
     const oldKey = keyForTab(activeTab);
     if (oldKey) {
       // Use a fresh URLSearchParams that reflects the just-flushed values:
       const sp = new URLSearchParams(window.location.search);
       if (searchTimerRef.current === null && searchQuery) sp.set('search', searchQuery);
       if (filterTimerRef.current === null && !isEmptyFilter(filterModel)) {
         sp.set('filter', JSON.stringify(filterModel));
       }
       saveStoredQueryString(oldKey, sp.toString());
     }

     // 3. Load new tab's bucket; null entries mean "delete that URL key".
     const newKey = keyForTab(newTab);
     const stored = newKey ? loadStoredQueryString(newKey) : null;
     const params = stored ? new URLSearchParams(stored) : new URLSearchParams();

     // 4. Apply via setParams. Per-key sync useEffects rehydrate local state.
     setParams({
       tab: newTab,
       loc: params.get('loc'),
       types: params.get('types'),
       search: params.get('search'),
       page: params.get('page'),
       pageSize: params.get('pageSize'),
       sort: params.get('sort'),
       filter: params.get('filter'),
     });

     // 5. Update local activeTab — sync useEffect on `validTabFromUrl` would
     //    catch this after the URL flushes, but we set it directly to avoid a
     //    transient render with stale activeTab.
     setActiveTab(newTab);
   }, [activeTab, searchQuery, filterModel, setParams, keyForTab]);
   ```
   Note: the existing `handleTabChange` *also* reset local state directly (`setSelectedLocationsState([])`, etc.). We **remove** those direct sets — local state is rehydrated by the existing per-key sync `useEffect`s once the URL settles. This avoids a flash of empty state before the rehydrate.

5. **Keep the rest of the hook intact.** The `isFirstSearchDebounceRef`/`isFirstFilterDebounceRef` guards, the per-key sync `useEffect`s, `buildParams`, `loadData`, etc. — all unchanged.

### Why this is safe with the existing mount-emit guard on orders-tracker

This change is scoped entirely to `useIntakesTable.ts`. The orders-tracker DataGrid mount-emit guard in `apps/web/app/orders-tracker/[category]/page.tsx` is untouched and continues to defend the orders-tracker page only.

---

## Test plan

### Automated (Jest)

Add these cases to `apps/web/app/intakes/hooks/useIntakesTable.test.tsx`:

1. **Mount seeds from the URL-active tab's bucket only.**
   Pre-seed `intakesLastQueryString-Processed` with `loc=abc`, mount with `?tab=Processed` → URL becomes `?tab=Processed&loc=abc`.
2. **Mount with a bucket for a different tab is ignored.**
   Pre-seed `intakesLastQueryString-Needs Review` with `loc=xyz`, mount with `?tab=Processed` (no Processed bucket) → URL stays `?tab=Processed`.
3. **Tab switch round-trip preserves filters.**
   Mount with `?tab=Needs Review`, set `loc=abc`, call `handleTabChange('Processed')`, call `handleTabChange('Needs Review')` → URL ends up `?tab=Needs Review&loc=abc`.
4. **Tab switch to an unseen tab clears all filter keys.**
   Mount with `?tab=Needs Review`, set `loc=abc`, call `handleTabChange('Skipped')` (no Skipped bucket) → URL is `?tab=Skipped`, nothing else.
5. **Tab switch saves the old tab's state to the old bucket.**
   Set `loc=abc` on *Needs Review*, switch to *Processed* → `intakesLastQueryString-Needs Review` in localStorage contains `loc=abc` (and any other live keys).
6. **Search debounce flush on tab change.**
   Type into search but don't wait for debounce. Call `handleTabChange('Processed')`. Then switch back → search value is restored on *Needs Review*.
7. **Filter debounce flush on tab change.**
   Same as (6) but for the DataGrid filter model.
8. **No mount-time wipe.** Pre-seed *Needs Review* bucket; mount; observe that mirror does not fire with empty QS before seed runs.

Update / replace any existing test that asserts on the single `intakesLastQueryString` key.

Keep the existing `useUrlParams.test.tsx`, `urlPersistence.test.tsx`, `useGridUrlModels.test.tsx`, and orders-tracker `page.test.tsx` suites passing untouched.

### Manual (Playwright MCP — verify before commit per user prefs)

1. Apply a filter on *Needs Review*, switch to *Processed*, switch back → filter restored.
2. Apply a filter on *Needs Review*, *different* filter on *Processed*, switch between them several times → each tab keeps its own filter.
3. Hard reload while on *Processed* with a filter → filter restored after reload.
4. Hard reload while on *Needs Review* with a filter → filter restored after reload (no cross-tab leak from a previously-active *Processed*).
5. Type a search query, immediately click *Processed*, click back → search restored.
6. Click *Clear all filters* on *Needs Review* → only *Needs Review* clears; *Processed* is untouched.
7. Paste `/intakes?tab=Processed&loc=xxx` when *Processed* bucket has saved state → bucket wins (matches the existing PR's storage-wins-over-URL contract).
8. New user (no localStorage) → all tabs load clean, persistence kicks in as soon as a filter is set.

---

## Rollout

- Branch: `hotfix/MLID-2225-per-tab-filter-persistence` (off `develop`).
- Commit style: `[MLID-2225] - fix(intakes): persist filter state per tab to prevent cross-tab wipe`.
- PR doc written to `docs/agomez/PR/MLID-2225-QA-fix.md` (separate from the original merged PR doc). Open the PR manually.
- No Jira transition.

---

## Open questions / follow-ups (out of scope for this hotfix)

- **Unify tab URL shape between `/intakes` and `/orders-tracker`.** PO has been asked. Once decided, either intakes moves to a `[tab]` path or orders-tracker moves to `?tab=…`. Whichever direction, the per-tab storage primitive stays useful.
- **Storage cleanup.** Old `intakesLastQueryString` orphans linger in user localStorage. Could be cleared in a follow-up cleanup pass. Low priority; entries are small.
- **Eventually generalize.** If a third surface needs tab-scoped persistence, the per-tab snapshot/restore pattern in `useIntakesTable` could be lifted into a small `useTabScopedUrlPersistence(prefix, scopeParam)` helper. Premature for one consumer.
