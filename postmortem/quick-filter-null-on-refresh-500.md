# Postmortem: Orders Tracker 500 on Refresh After Applying "Active Orders by Due Date" Quick Filter

**Status**: In progress. Hotfix branch `hotfix/MLID-2225-quick-filter-null-on-refresh` opened against `develop`.
**Date detected**: 2026-05-18 (local, by the engineer)
**Related ticket**: [MLID-2225](https://localinfusion.atlassian.net/browse/MLID-2225) — the Saving Filters & Column Ordering work whose persistence layer turned a latent client bug into a reproducible 500.
**Related files**:

- `apps/web/app/orders-tracker/[category]/page.tsx` — quick filter definition
- `apps/web/utils/api/orders/gridToMongoQuery.ts` — server-side filter translator
- `apps/web/app/orders-tracker/Orders/Orders.tsx` — column → `fieldTypes` derivation and the mount-time fetch effect
- `apps/web/app/orders-tracker/Orders/NewOrders.tsx` — `status` column lives behind the `ORDERS_TRACKER_IG_COLUMNS` feature flag
- `apps/web/app/orders-tracker/hooks/useGridUrlModels.ts` — `JSON.stringify` of the filter model into the URL
- `apps/web/utils/hooks/useUrlParams.ts` — storage ↔ URL mirror

---

## Summary (high level)

### What was the error?

After clicking the **"Active Orders by Due Date"** quick filter on `/orders-tracker/new`, navigating away (e.g. `/intakes`), navigating back, and hard-refreshing the browser (F5 / Cmd+R), the orders table failed to load with a "No rows" empty state. The browser console showed:

```
GET /api/orders/new?...&filter=...&fieldTypes=... 500 (Internal Server Error)
[ERROR] Error fetching new orders: Error: Failed to fetch new orders: 500
```

Server log:

```
TypeError: Cannot read properties of null (reading 'replace')
    at escapeRegex (utils/api/orders/gridToMongoQuery.ts:13:28)
    at Array.map (<anonymous>)
    at stringPredicate (utils/api/orders/gridToMongoQuery.ts:117:48)
    at gridFilterToMongoQuery (utils/api/orders/gridToMongoQuery.ts:501:26)
    at GET (app/api/orders/new/route.ts:77:131)
```

### Why did this happen?

Three independent defects line up:

1. **Quick-filter authoring bug.** The "Active Orders by Due Date" preset seeds `undefined` as a value to represent the "no status" bucket of orders. `JSON.stringify` of `[undefined, "Not Started", ...]` produces `'[null, "Not Started", ...]'` — `undefined` inside an array is replaced with literal `null` by spec. Once the model is round-tripped through the URL (and through localStorage by the MLID-2225 persistence layer), the `undefined` is gone forever; only `null` survives.

2. **fieldTypes / columns timing race.** The `status` column is only added to `columns` when the `ORDERS_TRACKER_IG_COLUMNS` feature flag resolves to `true`. `useFeatureFlag` returns `false` while the flag is loading. The `Orders` component derives `fieldTypes` from `columns` and fires its initial fetch (`Orders.tsx:247`) on mount — _before_ the flag finishes resolving. The first fetch goes out with a `fieldTypes` payload that has no `status` key.

3. **Server fragility.** `gridFilterToMongoQuery` falls back to `'string'` when a field is missing from `fieldTypes`. For the string `isAnyOf` branch, the value array is fed through `escapeRegex` without dropping `null`. `escapeRegex(null)` reads `null.replace(...)` and throws.

### Why does the first click work, but the refresh crash?

This is the most counter-intuitive part of the bug, and the part that masked it during the original MLID-2225 dev cycle. The same `null` is sent in **both** the first-click flow and the refresh flow, but only the refresh routes through the regex code path that can't handle it.

```
First click (page has been open long enough for flags to resolve):
  isIgColumsEnabled = true
    → columns includes { field: 'status', type: 'singleSelect' }
    → fieldTypes = { ..., status: 'singleSelect' }
    → server picks the singleSelect branch in gridFilterToMongoQuery
    → builds { status: { $in: [null, "Not Started", ...] } }
    → valid MongoDB — `null` is just a value to match
    → 200 OK ✅

Hard refresh (cold mount, flags still loading):
  isIgColumsEnabled = false (initial value)
    → columns excludes the IG-flag block (no status, dueDate, coverageStatus, ...)
    → fieldTypes = { ..., (no 'status') }
    → Orders.tsx mount effect fires fetch with this incomplete fieldTypes
    → server fallback: t = fieldTypes['status'] ?? 'string'
    → server picks the string branch
    → coerced.map(escapeRegex) → escapeRegex(null) → 💥 500
```

The user-visible repro of "navigate away then refresh" is just the simplest path that:
(a) puts the bad filter in localStorage (via the MLID-2225 mirror), then
(b) reloads cold so the feature flag hasn't resolved when the first fetch goes out.

A bare F5 right after clicking the chip the first time produces the same crash. The intermediate navigation is not load-bearing.

---

## Technical detail

### 1. The literal `undefined` in the quick filter definition

`apps/web/app/orders-tracker/[category]/page.tsx:440-450`:

```tsx
filterModel: {
  items: [
    {
      id: 1,
      field: 'status',
      operator: 'isAnyOf',
      value: [
        undefined,                              // <-- intent: include orders with no status
        ...Object.values(OrderStatus).filter(
          (s) =>
            ![
              OrderStatus.ReadyToSchedule,
              OrderStatus.Scheduled,
              OrderStatus.NoGo,
            ].includes(s)
        ),
      ],
    },
  ],
},
```

The rest of the codebase already has the correct primitive for "no value":

- `NO_VALUE_SENTINEL = '__LI_NO_VALUE__'` (`utils/api/orders/gridToMongoQuery.ts:6`)
- `withNoValueOption(...)` (`app/orders-tracker/utils/enumValueOptions.ts:12`) — used by every regular column filter
- Chip-label rendering in the same `page.tsx` (line 557) already maps the sentinel to `NO_VALUE_LABEL`

This quick filter is the lone caller that uses `undefined` instead of the sentinel.

### 2. The persistence round-trip strips `undefined` to `null`

`apps/web/app/orders-tracker/hooks/useGridUrlModels.ts:63-66`:

```ts
if (overrides.filterModel !== undefined) {
  patch.filter = isDefault(overrides.filterModel, DEFAULT_FILTER)
    ? null
    : JSON.stringify(overrides.filterModel); // <-- undefined in array becomes "null"
}
```

`JSON.stringify([undefined, "x"])` is specified to produce `'[null,"x"]'`. After this point, the `undefined` cannot be recovered. `useUrlParams` then mirrors the URL into localStorage (`useUrlParams.ts:89-94`), so the persisted entry permanently contains `null` in the array.

### 3. The columns / fieldTypes race

`apps/web/app/orders-tracker/Orders/Orders.tsx:174-183`:

```ts
const fieldTypes = useMemo(
  () =>
    columns.reduce<Record<string, GridColType>>((acc, c) => {
      acc[c.field] = c.type ?? 'string';
      return acc;
    }, {}),
  [columns]
);
const fieldTypesRef = useRef(fieldTypes);
fieldTypesRef.current = fieldTypes;
```

`columns` comes in as a prop, built by `getNewOrdersColumns({ isIgColumsEnabled, ... })` in the parent. When the flag is still loading (`useFeatureFlag` → `false`), the IG-block of columns is omitted:

```ts
// apps/web/app/orders-tracker/Orders/NewOrders.tsx:356
...(isIgColumsEnabled
  ? [
      { field: 'coverageStatus', type: 'singleSelect', ... },
      { field: 'status',          type: 'singleSelect', ... },
      { field: 'liPharmacyStatus', type: 'singleSelect', ... },
      { field: 'dueDate', type: 'date', ... },
      // ... 5 more columns
    ]
  : []),
```

So a cold-mount `fieldTypes` payload looks like (decoded from the captured 500 URL):

```json
{
  "actions": "actions",
  "assignedToUser.name|assignedTo": "string",
  "displayId": "string",
  "drug.name|drug": "string",
  "id": "string",
  "lastUpdated": "dateTime",
  "orderType": "singleSelect",
  "patient.full_name|patient.name": "string",
  "provider.first_name,provider.last_name|provider": "string",
  "providerOffice.name": "string",
  "providerPracticeFax": "string",
  "received": "date",
  "site.location_name|site.city,site.state": "string",
  "updatedByUser.name": "string"
}
```

— exactly the top-level columns minus the IG-flag-gated block. `status` is absent.

The mount-time fetch effect (`Orders.tsx:247-269`) depends on `fieldTypesRef`, not on the live `fieldTypes` memo. So the first fetch goes out with whatever snapshot the ref happened to capture before the flag resolved. Subsequent fetches after the flag flips would carry the full `fieldTypes`, but the damage is already done if the user reloads on a saved-filter URL.

### 4. The server's regex branch can't take `null`

`apps/web/utils/api/orders/gridToMongoQuery.ts:84-127`:

```ts
if (op === 'isAnyOf') {
  const arr = Array.isArray(item.value) ? item.value : [item.value];
  const hasSentinel = arr.some(isNoValueSentinel);
  const coerced = arr
    .filter((v) => !isNoValueSentinel(v))
    .map((v) => coerceValue(fieldType, v))
    .filter((v) => v !== undefined);            // <-- does NOT drop null

  // ...

  if (fieldType === 'string') {
    const stringPredicate = (field: string) => ({
      $expr: {
        $anyElementTrue: {
          $map: {
            input: coerced.map(escapeRegex),    // <-- escapeRegex(null) → TypeError
            // ...
```

`coerceValue` short-circuits on `null` and returns `null` unchanged (line 16). The final `filter(v => v !== undefined)` lets `null` through. Then `escapeRegex` (`s.replace(...)`, line 11) throws because `null.replace` does not exist.

The `singleSelect` branch of the same function does not call `escapeRegex` at all — it produces `{ [field]: { $in: coerced } }`, which is valid MongoDB even when the array contains `null`. That's why the flag-resolved (first-click) path is fine.

---

## Fix plan

Three layers, in increasing order of architectural impact. Layers 1 and 3 land in this hotfix. Layer 2 is filed as follow-up.

| #   | Layer                   | Change                                                                                             | Status           |
| --- | ----------------------- | -------------------------------------------------------------------------------------------------- | ---------------- |
| 1   | Client root cause       | `page.tsx:441` — `undefined` → `NO_VALUE_SENTINEL`                                                 | In this hotfix   |
| 3   | Server defense in depth | `gridToMongoQuery.ts:90` — `.filter(v => v !== undefined && v !== null)`                           | In this hotfix   |
| 2   | Race condition          | Defer the initial fetch in `Orders.tsx:247` until `fieldTypes` reflects the resolved feature flags | Follow-up ticket |

**Why all three matter:**

- Fix **1 alone** stops _this specific_ quick filter from emitting `null`, but the next person who reaches for `undefined` (or who introduces a saved filter referencing a no-longer-loaded field) reopens the same crash. The sentinel pattern is the team's existing answer; we should use it.
- Fix **3 alone** stops the 500 but masks the underlying defect — the server would silently drop the "no value" bucket from the result, and the filter would no longer match the chip the user sees. We want the user's intent preserved, not swallowed.
- Fix **2 alone** would also stop the 500 (because the singleSelect branch handles `null` fine), but leaves layer 1 as an ergonomic landmine for future quick filters and layer 3 as a class of latent server crashes triggered by _any_ future column that goes behind a feature flag.

Together, 1 + 3 close the immediate hotfix surface area; 2 closes the architectural class of bug.

---

## Lessons

- **`JSON.stringify` lies about `undefined` in arrays.** Anywhere user state crosses an `JSON.stringify`/`JSON.parse` boundary (URL, localStorage, network), `undefined` is _not_ a faithful representation of a "no value" bucket. The codebase already had the right pattern (`NO_VALUE_SENTINEL`) — the quick filter was the lone deviation. Reviewing for `undefined` inside literals that flow through `JSON.stringify` is a cheap lint.
- **`fieldTypes` is a contract between client column defs and the server filter translator.** When the client's column definitions are async (feature-flag-gated, role-gated, async-data-driven), the contract becomes time-dependent. Either the server should not depend on a per-request schema at all (use a static fields-and-types registry server-side), or the client must guarantee a stable schema before issuing the first fetch. Today it does neither.
- **MLID-2225 persistence is doing exactly what it was designed to do.** This bug is not in the persistence layer — it's in the value that the persistence layer was asked to persist. But persistence _exposed_ an asymmetry that was already present: pre-persistence, a user clicking the quick filter never lived through a cold mount with that filter in flight; with persistence, every refresh is a cold mount with that filter in flight.
- **Browser-tab-cache vs cold-mount divergence is a recurring failure mode.** The same family of feature-flag-load-order bugs has shown up before in this codebase. Worth a dedicated check on future feature-flag-gated columns.

---

## Verification (after fix)

Manual:

- [ ] On a fresh local dev session, clear orders-tracker localStorage. Click the **Active Orders by Due Date** chip. Network panel: `/api/orders/new` returns 200, table shows the expected filtered rows.
- [ ] Hard-refresh the browser. Network panel: same 200, same rows. (Pre-fix this is the failing step.)
- [ ] Navigate to `/intakes`, navigate back, hard-refresh. 200, same rows.
- [ ] Open the filter panel manually, add `Status is any of (No Value)`. Apply. 200, table shows only orders with no status.
- [ ] Apply a custom `Status is any of [Not Started, (No Value)]`. Refresh. 200, table shows orders matching either bucket.

Automated:

- New unit test in `gridToMongoQuery.test.ts` — `isAnyOf` with a `null` in the value array does not throw and produces a sane predicate.
- Existing test for the quick filter apply (page.test.tsx) updated to assert `NO_VALUE_SENTINEL` rather than `undefined` in the emitted filter value.

---

## Follow-up

- **[MLID-2409](https://localinfusion.atlassian.net/browse/MLID-2409)** (filed 2026-05-18, Sprint 28, 3 SP): Defer the `Orders.tsx` initial fetch until `fieldTypes` reflects the resolved feature flags. Options to evaluate: (a) gate the effect on a "all relevant feature flags settled" boolean from the FeatureFlagContext; (b) compute `fieldTypes` from a static registry that doesn't depend on column visibility; (c) have the server own the schema and stop trusting the client's `fieldTypes` payload at all. Option (c) is the longest-lived answer but the largest change.
