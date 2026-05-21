# Postmortem: Orders Tracker Page Crash on `(No Value)` Enum Filter

**Status**: Resolved. Hotfix branch `hotfix/MLID-2213-drug-undefined-crash` opened against `develop`.
**Date detected**: 2026-05-11 (staging)
**Related ticket**: [MLID-2213](https://localinfusion.atlassian.net/browse/MLID-2213) — the `(No Value)` enum-filter feature whose acceptance testing surfaced this crash.
**Related files**:

- `apps/web/app/orders-tracker/Orders/NewOrders.tsx` (drug column `renderCell`)
- `apps/web/app/orders-tracker/Orders/MaintenanceOrders.tsx` (same shape, same bug)
- `apps/web/app/orders-tracker/components/Drug/Drug.tsx` (the component that threw)
- `apps/web/utils/api/orders/gridToMongoQuery.ts` (sentinel → Mongo predicate translation)

---

## Summary (high level)

### What was the error?

Visiting `/orders-tracker/new`, opening the filter panel, choosing **Order Type / is / (No Value)** caused the page to crash. Browser console:

```
TypeError: Cannot read properties of undefined (reading 'name')
    at Drug.tsx (94:21)
```

The grid blanked out, the React error boundary did not recover the page, and the user had to refresh to escape the state.

### Why did this happen?

The `Drug` cell component dereferences `drug.name` without first verifying `drug` is defined. A small number of orders in `neworders` have `drug = undefined` (legacy or partially-imported records). When the grid tries to render one of those rows, the `Drug` component throws.

The bug had existed in the code for ~2 months before being observed. It went undetected because the broken rows were a small minority, scattered across deep pages of the default `received desc` listing, where they were rarely rendered.

### Why did this error only happen when we select the new `(No Value)` item in filters?

The `(No Value)` filter does not introduce the broken rows — it concentrates them. Selecting `Order Type is (No Value)` translates server-side into a Mongo predicate that returns **only** rows missing `orderType`. Those same rows tend to also be missing `drug` (incomplete records are incomplete across multiple fields, not just one). The filter therefore guarantees page 1 of the result is full of the exact rows that trigger the latent crash.

Mental model:

```
Before (No Value) filter existed:
  Orders DB
    ├── thousands of well-formed orders   ← shown on pages 1..N, no crash
    └── handful of incomplete orders       ← buried on deep pages, rarely rendered

After (No Value) filter:
  Orders DB
    └── only the incomplete orders         ← all on page 1 → 💥
```

The same crash is reproducible without the filter by any path that surfaces a `drug === undefined` row (deep pagination, a sort that floats nulls forward, etc.). `(No Value)` is just the most reliable trigger.

---

## Technical detail

### 1. The throwing line

`apps/web/app/orders-tracker/components/Drug/Drug.tsx`, in the component body that turns a row's `drug` into the rendered cell:

```tsx
let drugName: string;
let drugHref: string | undefined;
let isTreatment = false;
let hasBiosimilar = false;
let currentBiosimilar: ... | undefined;

if (typeof drug === 'string') {
  drugName = drug;
  drugHref = undefined;
} else {
  drugName = drug.name;                                       // ← throws when drug is undefined
  drugHref = `/drug-admin-panel/drugs/${drug._id}`;           // (and these would too)
  isTreatment = Boolean(drug.is_treatment);
  hasBiosimilar = Boolean(drug.biosimilar);
  currentBiosimilar = drug.biosimilar?.drugs.find(
    (b) => b.li_drug_id === drug._id
  );
}
```

The component's declared prop type is `drug: LIDrug | string`, which excludes `undefined`. The `else` branch correctly relies on that contract — but the contract is **enforced only at compile time**. At runtime, if a caller passes `undefined`, the type check has nothing to say and `drug.name` evaluates `undefined.name`, producing the `TypeError`.

### 2. The unsafe call site

`apps/web/app/orders-tracker/Orders/NewOrders.tsx`, in the `drug.name|drug` column definition (pre-fix):

```tsx
renderCell: (params) =>
  typeof params.row.drug === 'string' ? (
    params.row.drug
  ) : (
    <Drug
      drug={params.row.drug}
      orderProtocol={params.row.orderProtocol}
      params={params}
      /* ... */
    />
  ),
```

The conditional has two implicit assumptions:

1. The interesting case to special-case is `drug` being a **string** (a legacy denormalized field).
2. Anything else is a valid `LIDrug` object.

Assumption 2 silently absorbs `undefined`. `typeof undefined === 'string'` is `false`, so the `<Drug>` branch runs with `drug={undefined}`. The grid then crashes inside `Drug.tsx` rather than at the call site.

The identical shape exists in `MaintenanceOrders.tsx` for the maintenance tab. (The lead tab's drug column is `type: 'string'` with no `<Drug>` involvement and is unaffected.)

### 3. The data shape on the broken rows

A representative broken record (sanitized):

```js
{
  _id: ObjectId("…"),
  displayId: "…",
  received: ISODate("2024-…"),
  patient: { _id: ObjectId("…"), full_name: "…" },
  site: { _id: ObjectId("…"), location_name: "…", city: "…", state: "…" },
  assignedTo: "…",
  // orderType: <missing>
  // drug: <missing>
  // provider: <missing>
  category: "new",
  createdAt: ISODate("2024-…"),
}
```

These records typically come from partial intakes or legacy imports that landed before the current intake flow enforced the full field set. They are valid documents — they just lack the fields the UI assumes are populated.

### 4. How the `(No Value)` filter routes those rows to page 1

`apps/web/utils/api/orders/gridToMongoQuery.ts`, the sentinel translation:

```ts
const NO_VALUE_SENTINEL = '__LI_NO_VALUE__';

function emptyPredicate(field: string) {
  return {
    $or: [
      { [field]: { $exists: false } },
      { [field]: null },
      { [field]: '' },
    ],
  };
}

// inside itemToMongoPredicate for fieldType === 'singleSelect':
switch (op) {
  case 'is':
  case 'equals':
  case '=':
    return sentinel ? emptyPredicate : (field) => ({ [field]: v });
  // …
}
```

For `{ field: 'orderType', operator: 'is', value: '__LI_NO_VALUE__' }`, the resulting `$match` stage is:

```js
{
  $match: {
    $or: [
      { orderType: { $exists: false } },
      { orderType: null },
      { orderType: '' },
    ],
  },
}
```

This is correct behavior — it is exactly the semantic the feature was designed to provide. But because field-level missingness in this dataset is **correlated** across fields, the rows it returns are precisely the rows that also have `drug = undefined`. The query is fine; the **rendering** of its results is what blows up.

### 5. Why the crash hid for so long

Three factors kept the latent bug from being observed before MLID-2213 shipped:

1. **Frequency**: broken rows are a small minority of total orders.
2. **Default ordering**: the grid sorts `received desc` by default. Incomplete legacy rows tend to have older `received` timestamps and land on later pages.
3. **Pagination cost**: server-side pagination + 25-row default page size means users have to actively navigate to a page containing a broken row to render one. In practice, almost no one does.

Without an affordance that says *"show me only the broken rows,"* the crash was statistically improbable per session. `(No Value)` is that affordance.

---

## Fix

### Where the guard goes

The fix is at the **call site**, not inside `Drug`. Rationale:

- `Drug`'s prop type already excludes `undefined`. Widening it to accept `undefined` would silently legitimize bad inputs and shift the integrity-check burden to a leaf component.
- The call site already handles the string-vs-object dichotomy; the missing case is "neither — there is nothing to render." That belongs alongside the existing conditional.
- A defensive `if (!drug) return null` inside `Drug` would also work, but it (a) trains future callers to assume Drug can absorb anything, and (b) leaves the same risk on the four other unguarded dereferences in the same `else` branch (`drug._id`, `drug.is_treatment`, `drug.biosimilar`, `drug.biosimilar?.drugs.find`).

### The change

In both `NewOrders.tsx` and `MaintenanceOrders.tsx`, the drug column's `renderCell` gains an early-return for falsy `drug`:

```tsx
renderCell: (params) => {
  if (!params.row.drug) return null;
  return typeof params.row.drug === 'string' ? (
    params.row.drug
  ) : (
    <Drug
      drug={params.row.drug}
      /* … */
    />
  );
},
```

Net effect: a row with `drug === undefined` now renders an empty Drug cell instead of crashing the page. Rows with a populated drug are unaffected.

### Tests

A regression test was added to each grid's existing **Drug column** block. The test:

1. Builds the columns.
2. Locates the Drug column.
3. Constructs a row with `drug: undefined as unknown as FullOrder<'new'>['drug']`.
4. Calls `drugColumn.renderCell(params)` directly.
5. Asserts the return is `null`.

Both tests fail RED before the guard (the unmocked output shows `<Drug … drug={undefined} … />`, which is exactly the call shape that crashes in production) and pass GREEN after.

### Verification

| Check | Result |
|-------|--------|
| `npx jest "Orders/(NewOrders|MaintenanceOrders).test.tsx"` | 93/93 passed (+2 new) |
| `npm run types:check` (apps/web) | Clean |
| ESLint `--max-warnings 0` on the 4 changed files | Clean |
| Prettier on the 4 changed files | No changes |
| Branch coverage on the two changed components | 90.99% |

---

## Lessons

- **A prop type that excludes `undefined` is a compile-time contract, not a runtime guarantee.** The closer a leaf component is to data-of-uncertain-completeness (e.g., a Mongo document with optional fields), the less safe it is to rely on type narrowing alone. Either enforce at the boundary (the call site, ideally with an explicit guard) or use defensive coding inside the component.
- **New UI affordances can surface latent rendering bugs without introducing them.** A feature that legitimately exposes a previously-rare data shape is doing its job; the crash is owned by the rendering path that assumed that shape would never arrive. Treat new "show me the edge cases" filters as an opportunity to audit the cells that render those cases.
- **Correlated missingness in legacy data.** Records that lack one optional field tend to lack several. A filter for "missing X" is implicitly also a filter for "likely missing Y, Z." Worth keeping in mind whenever a `(No Value)`-style affordance is added to a new column.
- **`typeof x === 'string'` is not a not-null check.** A two-branch conditional like `typeof x === 'string' ? A : B` reads as if it covers all cases, but the `else` branch absorbs `undefined`, `null`, numbers, arrays, plain objects — anything non-string. When the type contract is `T | string`, the contract is what makes the `else` safe; the `typeof` check by itself does not.
- **Server-side pagination hides rendering bugs that local pagination would expose.** With everything on the client, a single render pass over every row would have surfaced this crash long ago. With server-side paging, the bug is invisible until the query selects the bad rows.

---

## Followups (out of scope for the hotfix)

- Audit the other dereferences in `Drug.tsx`'s `else` branch (`drug._id`, `drug.is_treatment`, `drug.biosimilar`, `drug.biosimilar?.drugs.find`). They are protected from the immediate crash by the new call-site guard, but the leaf-level brittleness remains.
- Survey the rest of the orders-tracker columns for the same shape: `typeof x === 'string' ? string-render : <Component x={x} />` with no falsy-guard. Patient and Provider columns already use optional chaining; verify the remaining columns do too.
- Consider whether incomplete `neworders` documents should be backfilled (data cleanup) or whether the UI should continue treating them as legitimate "empty cell" rows. This is a product decision, not an engineering one.
