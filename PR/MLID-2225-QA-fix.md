# [MLID-2225] fix(intakes): persist filter state per tab to prevent cross-tab wipe

## Summary

Hotfix for the previously-merged MLID-2225 (PR #1530). QA rejected the original implementation because filter state on `/intakes` did not survive a tab change: applying a filter on *Needs Review*, switching to *Processed*, and switching back showed an empty filter instead of the previous one.

The original implementation used a single shared storage key `intakesLastQueryString` and `handleTabChange` wiped every owned URL key before switching tabs. The mirror effect inside `useUrlParams` then wrote that wiped URL into storage, so the previous tab's state was destroyed — not hidden. Coming back to a tab had nothing to restore.

This change moves intakes to **per-tab storage** (one bucket per `IntakeTab`), aligning with the orders-tracker pattern. The fix is scoped entirely to `useIntakesTable.ts`; the URL shape (`?tab=…` as a query param) is unchanged.

**Branch:** `hotfix/MLID-2225-per-tab-filter-persistence` → `develop`

**Jira:** [MLID-2225](https://localinfusion.atlassian.net/browse/MLID-2225)

---

## Root Cause

In `useIntakesTable.handleTabChange` (pre-fix):

```ts
setParams({ tab, loc: null, types: null, search: null, page: null, pageSize: null, sort: null, filter: null });
```

This wiped every URL key owned by intakes before switching tabs. `useUrlParams`'s mirror effect then observed the wiped URL and wrote it into the single `intakesLastQueryString` storage key — overwriting whatever the old tab had built up. There was no per-tab namespace, so any state was global; any tab switch was destructive.

Orders-tracker did not suffer from this because each tab is a separate path (`/orders-tracker/new|maintenance|lead`), with its own page mount and its own storage key (`ordersTrackerLastQueryString-<tab>`). Intakes uses a single mount with `?tab=…`, so the persistence layer has to be tab-aware.

---

## Architecture

```
LocalStorage (one bucket per tab)
   intakesLastQueryString-Needs Review
   intakesLastQueryString-Processed
   intakesLastQueryString-System Processing
   intakesLastQueryString-Failed
   intakesLastQueryString-Skipped
   intakesLastQueryString-All
        |
        v  (seed for the current tab on mount; re-seed on tab change)
URL params (single page mount, `tab` is just another param)
        |
        v  (parse via useMemo / per-key sync useEffects)
UI state
        |
        ^  (user change)
        |
URL params  ── granular setParams patches
        |
        v  (mirror to bucket for current tab on every change)
LocalStorage
```

`useUrlParams` is now called without `storageKey` from `useIntakesTable`, so its seed and mirror are dormant. All storage logic lives inside `useIntakesTable` and is tab-aware. `useUrlParams`, `urlPersistence`, `useGridUrlModels`, and the orders-tracker page are untouched.

---

## Changes Overview

- **Files changed:** 2
- **Lines added:** +422
- **Lines removed:** -59

## Files Changed

| File | Change | Description |
|---|---|---|
| `apps/web/app/intakes/hooks/useIntakesTable.ts` | Modified | Per-tab seed, mirror, and tab-change snapshot/restore logic using `urlPersistence` primitives directly. Adds debounce-flush in `handleTabChange`. |
| `apps/web/app/intakes/hooks/useIntakesTable.test.tsx` | Modified | New `per-tab persistence` describe block (8 scenarios). Existing URL serialization, tab-change, and debounce tests unchanged. |

---

## What Changed and Why

### 1. `options.storageKey` is now a per-tab prefix

```ts
const storagePrefix = options?.storageKey ?? null;
const keyForTab = useCallback(
  (tab: IntakeTab): string | null =>
    storagePrefix ? `${storagePrefix}-${tab}` : null,
  [storagePrefix]
);
```

`IntakesPageNew.tsx` continues to call `useIntakesTable({ storageKey: 'intakesLastQueryString' })` — no page-level change needed. The string `intakesLastQueryString` is now a prefix, and each tab gets its own key (e.g. `intakesLastQueryString-Needs Review`).

### 2. Mount-time seed reads the URL-active tab's bucket only

```ts
const hasSeededRef = useRef(false);
useEffect(() => {
  if (hasSeededRef.current) return;
  hasSeededRef.current = true;

  const key = keyForTab(validTabFromUrl);
  if (!key) return;
  const stored = loadStoredQueryString(key);
  if (!stored) return;
  // … sync window.history.replaceState + router.replace, then apply to local state
}, []);
```

`window.history.replaceState` is called synchronously so the mirror effect (which runs in the same effects flush) reads the seeded URL from `window.location` rather than the pre-seed URL. `router.replace` is also called so Next.js's `useSearchParams` propagates the change for subsequent renders. Both are necessary — same reasoning as Layer 2 of the original PR.

Local state is applied directly (not via the per-key sync `useEffect`s) because those depend on `searchParams`, which lags one render behind `router.replace`.

### 3. Mirror writes the current URL into the current tab's bucket

```ts
useEffect(() => {
  if (!hasSeededRef.current) return; // never wipe before seeding
  const key = keyForTab(activeTab);
  if (!key || typeof window === 'undefined') return;
  const currentQs = window.location.search.replace(/^\?/, '');
  saveStoredQueryString(key, currentQs);
}, [searchParams, activeTab, keyForTab]);
```

Reads `window.location` (not `searchParams`) for the same one-render lag reason.

### 4. `handleTabChange` flushes, snapshots, restores

```ts
const handleTabChange = useCallback((newTab: IntakeTab) => {
  if (newTab === activeTabRef.current) return;

  // 1. Flush pending debounces so un-URL'd values are captured in the snapshot.
  if (searchTimerRef.current) { clearTimeout(searchTimerRef.current); searchTimerRef.current = null; }
  if (filterTimerRef.current) { clearTimeout(filterTimerRef.current); filterTimerRef.current = null; }

  // 2. Snapshot the old tab's state into its bucket, including the just-flushed values.
  const oldKey = keyForTab(activeTabRef.current);
  if (oldKey) {
    const currentSp = new URLSearchParams(window.location.search);
    if (currentSearch) currentSp.set('search', currentSearch); else currentSp.delete('search');
    if (!isEmptyFilter(currentFilter)) currentSp.set('filter', JSON.stringify(currentFilter));
    else currentSp.delete('filter');
    saveStoredQueryString(oldKey, currentSp.toString());
  }

  // 3. Load the new tab's bucket (or empty defaults).
  const newKey = keyForTab(newTab);
  const stored = newKey ? loadStoredQueryString(newKey) : null;
  const params = stored ? new URLSearchParams(stored) : new URLSearchParams();

  // 4. Apply to local state directly + 5. Sync URL via setParams.
});
```

Local state is set directly (not waiting for the per-key sync `useEffect`s) to avoid a transient render with stale state. Refs (`searchQueryRef`, `filterModelRef`, `activeTabRef`, `routerRef`) hold the latest values so the callback dep array stays small and the callback identity is stable across keystrokes.

### 5. Trade-offs preserved from the original PR

- **Storage wins over URL on entry** — pasting `/intakes?tab=Processed&loc=xyz` is overwritten by the Processed bucket on mount. Same product call as the original.
- **No migration of orphan `intakesLastQueryString` entries** — pre-fix storage keys sit harmlessly in users' localStorage. Cleanup is a low-priority follow-up.

---

## Commits

| Hash | Message |
|---|---|
| `3698672a8` | `[MLID-2225] - fix(intakes): persist filter state per tab to prevent cross-tab wipe` |

---

## Test Plan

### Automated (Jest)

All 51 tests in `useIntakesTable.test.tsx` pass, including the new `per-tab persistence` block:

- Mount seeds from the URL-active tab's bucket only.
- Mount with a bucket for a different tab is ignored.
- Tab switch round-trip preserves filters.
- Tab switch to an unseen tab uses empty defaults.
- Tab switch saves the old tab's state to the old bucket.
- Search debounce flush on tab change.
- Filter debounce flush on tab change.
- URL-to-bucket mirror updates on every URL change.

Quality gate:
- Full Jest suite passes (15560/15563; 2 unrelated pre-existing failures on `develop`).
- `tsc --noEmit` clean.
- Prettier clean, ESLint clean.
- Coverage on `useIntakesTable.ts`: 97.45% lines, 100% functions.

### Manual (browser)

- [x] Apply a filter on *Needs Review*, switch to *Processed*, switch back → filter restored.
- [x] Apply a filter on *Needs Review*, a different filter on *Processed*, switch between them several times → each tab keeps its own filter.
- [x] Hard reload while on *Processed* with a filter → filter restored after reload.
- [x] Hard reload while on *Needs Review* with a filter → filter restored after reload (no cross-tab leak).
- [x] Type a search query, immediately click *Processed*, click back → search value restored.
- [x] Click *Clear all filters* on *Needs Review* → only *Needs Review* clears; *Processed* untouched.
- [x] Paste `/intakes?tab=Processed&loc=xxx` when *Processed* bucket has saved state → bucket wins.
- [x] New user (no localStorage) → all tabs load clean, persistence starts as soon as a filter is set.

---

## Jira

- [MLID-2225](https://localinfusion.atlassian.net/browse/MLID-2225)
