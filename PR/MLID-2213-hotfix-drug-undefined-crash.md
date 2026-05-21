# [MLID-2213] Hotfix: Guard drug column against undefined `row.drug` on `/orders-tracker/new`

## Summary

- Detected on staging while testing MLID-2213 (the `(No Value)` enum filter): selecting `Order Type` → operator `is` → value `(No Value)` crashed the page with `TypeError: Cannot read properties of undefined (reading 'name')` at `Drug.tsx:94`.
- Root cause is **not** the filter — it's a pre-existing landmine since 2026-03-09. The Drug column's `renderCell` in `NewOrders.tsx` and `MaintenanceOrders.tsx` only guarded `typeof params.row.drug === 'string'`, then unconditionally rendered `<Drug drug={params.row.drug} />`. When `row.drug` is `undefined`, the `typeof` check returns `false`, the `<Drug>` branch runs, and `Drug.tsx` blows up on `drug.name`.
- Rows with `drug = undefined` are a small minority of legacy/incomplete records that also tend to have `orderType` missing. The default `received desc` sort buries them on deep pages where users rarely scroll. The new `(No Value)` filter is the first UI affordance that concentrates them onto page 1 — exposing the latent crash.
- Fix at the call site: guard `if (!params.row.drug) return null` before the `<Drug>` render. Keeps `Drug`'s `LIDrug | string` prop contract honest.

**Branch:** `hotfix/MLID-2213-drug-undefined-crash` → `develop`

**Reproduction (pre-fix):**

1. Navigate to `/orders-tracker/new`.
2. Open filter panel → column **Order Type** → operator **is** → value **(No Value)**.
3. Page crashes with `TypeError: Cannot read properties of undefined (reading 'name')` (visible in browser console, app shell blank).

The same crash also triggers from **Maintenance Orders** under any enum `(No Value)` filter (e.g., `changeType is (No Value)`), since `MaintenanceOrders.tsx` has the identical unsafe `renderCell` shape.

---

## Why only the `(No Value)` filter triggered it

The bug is data-driven, not filter-driven:

```
Before MLID-2213:
  Orders DB
    ├── 9,9xx well-formed orders     ← shown on pages 1..N (no crash)
    └── few incomplete orders        ← buried on deep pages (rarely rendered)

After MLID-2213 + "(No Value)" filter:
  Orders DB
    └── only incomplete orders       ← ALL on page 1 → 💥
```

`gridToMongoQuery.ts:224` translates the `__LI_NO_VALUE__` sentinel for `singleSelect + is` into `emptyPredicate(field)` — a `$or` of `$exists:false / null / ''`. So Mongo returns exclusively the incomplete records, and those same records also happen to have `drug = undefined`. Any other repro path (deep pagination, sort that floats nulls forward, etc.) would have hit the same crash.

---

## Changes

| File | Change |
|------|--------|
| `apps/web/app/orders-tracker/Orders/NewOrders.tsx` | Drug column `renderCell`: early-return `null` when `params.row.drug` is falsy, before the `typeof === 'string'` branch. |
| `apps/web/app/orders-tracker/Orders/MaintenanceOrders.tsx` | Same fix — the file had the identical unsafe shape. |
| `apps/web/app/orders-tracker/Orders/NewOrders.test.tsx` | Added `should render null when row.drug is undefined` to the Drug column block. Verified RED before the fix, GREEN after. |
| `apps/web/app/orders-tracker/Orders/MaintenanceOrders.test.tsx` | Same test added to the parallel block. |

### NOT touched

- `Drug.tsx` — its `LIDrug | string` prop contract excludes `undefined` and the right fix is at the caller, not by widening Drug's contract. Two other unsafe accesses (`drug._id`, `drug.is_treatment`, `drug.biosimilar`) sit in the same else-branch and would crash identically if a caller ever passed `undefined`, but guarding all of them inside Drug would only encourage callers to keep passing bad data.
- `LeadOrder.tsx` — its drug column is `type: 'string'` rendered as plain text; no `<Drug>` component involvement.
- `gridToMongoQuery.ts` — the `(No Value)` → `emptyPredicate` translation is correct; the filter feature itself works as designed.
- Backend / Mongo data — incomplete records remain in the collection. Cleaning them up is a separate decision (data-quality, not safety).

---

## Verification

| Check | Result |
|-------|--------|
| `npx jest "Orders/(NewOrders|MaintenanceOrders).test.tsx"` | 93/93 passed (+2 new) |
| `npm run types:check` (apps/web) | Clean |
| `eslint --max-warnings 0` on 4 changed files | Clean |
| `prettier` on 4 changed files | Unchanged |
| Branch coverage on changed files | 90.99% |

### RED-before-GREEN evidence

Pre-fix run of the two new tests (showing the unguarded `<Drug>` is rendered with `"drug": undefined` — i.e., the same call shape that crashes in production):

```
● should render null when row.drug is undefined
  expect(received).toBeNull()
  Received: <Drug ... params={{"row": {..., "drug": undefined, ...}}} />
```

---

## Test Plan

- [ ] On staging or local: navigate to `/orders-tracker/new`, open the filter panel, select **Order Type / is / (No Value)** → page renders the result list without crashing; rows missing a drug show an empty Drug cell.
- [ ] Same flow on `/orders-tracker/maintenance` with **Change Type / is / (No Value)**.
- [ ] Sanity: existing rows with a populated drug still render the Drug component (link, biosimilar switch chip, etc.) unchanged.
- [ ] Sanity: filtering by `(No Value)` on other enum columns (`Audit`, `Coverage Status`, `Order Status`, `LI Pharm Status`) does not crash even if the resulting rows have `drug = undefined`.
- [ ] Sanity: the inline Drug cell edit (biosimilar autocomplete) still opens for editable rows where drug *is* populated.
