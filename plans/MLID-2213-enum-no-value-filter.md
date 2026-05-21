# MLID-2213 — Orders Filter: "(No Value)" option for enum columns

## Task Reference

- **Jira**: [MLID-2213](https://localinfusion.atlassian.net/browse/MLID-2213)
- **Type**: Story
- **Parent epic**: MLID-1745 (no local epic plan; treat as standalone)
- **Story Points (proposed)**: 2
- **Branch**: `feature/MLID-2213-enum-no-value-filter`
- **Base Branch**: `develop`
- **Status**: Done — branch `feature/MLID-2213-enum-no-value-filter`, 4 commits ready, manual UI verification passed (2026-05-06). PR doc at `docs/agomez/PR/MLID-2213.md`.

---

## Summary

Extend every `singleSelect` (enum) column on the Orders tracker so the filter dropdown shows an extra `(No Value)` option that filters to rows where the field is missing, `null`, or `''`. The `(No Value)` option appears only in the filter UI — never in the inline-edit dropdown — by adopting MUI X v8's function-form `valueOptions`. The server translates a sentinel value into the existing `emptyPredicate` / `notEmptyPredicate` helpers in `gridToMongoQuery.ts`. No new MUI X operators are introduced; we only extend the value list as the PO requested.

The shared `NO_VALUE_SENTINEL` constant lives directly in `gridToMongoQuery.ts` (where the server logic that interprets it lives) and the client helper imports it from there — no separate constants module. The chip label change is made inline in `page.tsx`; tests live in the existing `page.test.tsx`. No extracted helper file.

---

## Codebase Analysis

- **Affected enum columns (10 total)**: `orderType`, `coverageStatus`, `status`, `liPharmacyStatus`, `audit` in `NewOrders.tsx`; `changeType`, `coverageStatus`, `status` in `MaintenanceOrders.tsx`; `orderType`, `status` in `LeadOrder.tsx`. All use bare `valueOptions: Object.values(SomeEnum)` today.
- **Server filter pipeline (`apps/web/utils/api/orders/gridToMongoQuery.ts`)** already has `emptyPredicate` / `notEmptyPredicate` helpers (lines 47–61). For `singleSelect`, `is`/`equals`/`=` produce `{ field: v }`, `not`/`doesNotEqual`/`!=` produce `{ field: { $ne: v } }`, and `isAnyOf` produces `{ field: { $in: coerced } }` (lines 79–106, 193–210). `coerceValue` returns `singleSelect` values unchanged.
- **MUI X v8.27.3** supports the `valueOptions` function form `(params) => Array<ValueOptions>`. The function is invoked without `row` for the filter panel and with `row` for inline edit — the canonical way to differentiate the two contexts. No existing usage in `apps/web/`; this PR introduces it.
- **Chip label formatter** lives inline in `apps/web/app/orders-tracker/[category]/page.tsx` (lines 497–529). It quotes the raw value, so a sentinel string would render verbatim unless we map it. The existing `page.test.tsx` already has a `'Filter chip label formatting'` describe block (around lines 677–701) with a `renderWithFilters` helper — new chip tests slot into that block; no extraction required.
- **Existing column-test pattern**: `getNewOrdersColumns(opts)` / `getMaintenanceOrdersColumns(opts)` / `getLeadOrdersColumns(opts)` factories already exist and accept feature-flag toggles like `isIgColumsEnabled`. Asserting the function-form contract is straightforward: `const fn = (column as GridSingleSelectColDef).valueOptions as Function; expect(fn({ field })).toContain({ value: NO_VALUE_SENTINEL, label: '(No Value)' });`.
- **`gridToMongoQuery.test.ts`** has zero coverage of `singleSelect`, `isEmpty`, `isNotEmpty`, or `isAnyOf` today. We add focused tests for sentinel handling only — we do not refactor the existing string-based tests.
- **Out of scope (pre-existing behavior)**: the "Active Orders by Due Date" quick filter (page.tsx ~line 403–415) inserts `undefined` into an `isAnyOf` array hoping to match null status; the server's `.filter((v) => v !== undefined)` silently drops it. Note as a follow-up; do not touch in this PR.

---

## Resolved Decisions

| Decision | Choice |
|---|---|
| New operators? | **No.** Do not add `isEmpty` / `isNotEmpty` to UI columns. Extend the value list only. |
| Filter-only visibility | **Function-form `valueOptions`.** When invoked without `row` → return `[...enum, sentinel]`; with `row` → return `[...enum]`. |
| Sentinel constant | `NO_VALUE_SENTINEL = '__LI_NO_VALUE__'`, declared and exported directly from `apps/web/utils/api/orders/gridToMongoQuery.ts` (where the server logic that interprets it lives). The client helper imports from there. **No separate constants module.** |
| `NO_VALUE_LABEL` | `'(No Value)'`, defined in the client helper `enumValueOptions.ts` (UI concern only — server doesn't need it). |
| Dropdown label | Exactly `(No Value)` (parens included), per PO spec. |
| Server: `is` + sentinel | Substitute `emptyPredicate(field)`. |
| Server: `not` + sentinel | Substitute `notEmptyPredicate(field)`. |
| Server: `isAnyOf` containing sentinel | Build `$or: [{ field: { $in: realValues } }, emptyPredicate(field)]`. If only the sentinel is present → just `emptyPredicate(field)`. If the array has zero real values **and** no sentinel → existing behavior (skip filter, return `null`). |
| Chip label | Add a sentinel→label mapping in `formatFilterChipLabel` so single and array paths render `(No Value)` instead of `__LI_NO_VALUE__`. Existing operator labels and truncation behavior unchanged. |
| `audit`/`Audit.NA` semantics | Default: `(No Value)` does **not** match `audit === 'N/A'` (only true null/missing/`''`). Surface as the only Open Question. |
| Fallback if MUI v8.27.3 calls function-form for inline edit too | Defensive `valueSetter` on enum columns no-oping the sentinel. We commit to the function-form approach; this fallback is documented but not implemented unless impl reveals a v8.27.3 bug. |

---

## Implementation Steps

Four commits, each a complete red-green-refactor cycle. Plan ≤ 3 files per commit.

### Commit 1 — Server: sentinel translation in `gridToMongoQuery`

- **Files**:
  - `apps/web/utils/api/orders/gridToMongoQuery.ts` (Modify)
  - `apps/web/utils/api/orders/gridToMongoQuery.test.ts` (Modify)

- **RED — failing tests** (add to `describe('gridFilterToMongoQuery', ...)`):
  1. `'singleSelect "is" with the sentinel value substitutes emptyPredicate on _filterKey0'`
  2. `'singleSelect "not" with the sentinel value substitutes notEmptyPredicate on _filterKey0'`
  3. `'singleSelect "isAnyOf" containing only the sentinel substitutes emptyPredicate'`
  4. `'singleSelect "isAnyOf" mixing sentinel and real values yields $or: [{ $in: [...real] }, emptyPredicate]'`
  5. `'singleSelect "is" with a real enum value still produces { _filterKey0: value }'` (regression)
  6. `'singleSelect "isAnyOf" with only real values still produces { _filterKey0: { $in: [...] } }'` (regression)

  All use the existing `JSON.stringify(result) === JSON.stringify([...])` style. Field type passed as `{ status: 'singleSelect' }`.

- **GREEN — implementation** (all in `gridToMongoQuery.ts`):
  - Add at module top: `export const NO_VALUE_SENTINEL = '__LI_NO_VALUE__';`. The constant lives where the server logic that interprets it lives — no separate module.
  - In the `isAnyOf` branch (lines 79–106), partition values: `const hasSentinel = arr.some(v => v === NO_VALUE_SENTINEL); const realCoerced = arr.filter(v => v !== NO_VALUE_SENTINEL).map(...).filter(v => v !== undefined);`. If `hasSentinel && realCoerced.length === 0` → return `emptyPredicate`. If `hasSentinel && realCoerced.length > 0` → return `(field) => ({ $or: [{ [field]: { $in: realCoerced } }, emptyPredicate(field)] })`. Otherwise existing path.
  - In the `singleSelect` branch (lines 193–210): if `item.value === NO_VALUE_SENTINEL`, then `is/equals/=` returns `emptyPredicate`, `not/doesNotEqual/!=` returns `notEmptyPredicate`. Otherwise existing path.

- **REFACTOR**:
  - Keep both new branches behind a small helper `isNoValueSentinel(v: unknown): v is typeof NO_VALUE_SENTINEL` if it improves readability.
  - Run `npm run test -- gridToMongoQuery`, `npm run types:check`, `npm run lint:fix` from `apps/web/`.

- **Commit message**: `[MLID-2213] - feat(orders-filter): translate (No Value) sentinel to empty predicates in singleSelect/isAnyOf`

### Commit 2 — Client: `withNoValueOption` helper + unit tests

- **Files**:
  - `apps/web/app/orders-tracker/utils/enumValueOptions.ts` (Create — `utils/` folder is new under `orders-tracker/`)
  - `apps/web/app/orders-tracker/utils/enumValueOptions.test.ts` (Create)

- **Signature**:
  ```ts
  import type { ValueOptions } from '@mui/x-data-grid-pro';
  import { NO_VALUE_SENTINEL } from '@/utils/api/orders/gridToMongoQuery';

  export const NO_VALUE_LABEL = '(No Value)';

  export function withNoValueOption<T extends string | number>(
    enumValues: readonly T[]
  ): (params: { field: string; id?: unknown; row?: unknown }) => ValueOptions[];
  ```
  - When `params.row` is `undefined` → return `[...enumValues, { value: NO_VALUE_SENTINEL, label: NO_VALUE_LABEL }]`.
  - When `params.row` is defined → return `[...enumValues]`.

- **RED — failing tests** (Arrange-Act-Assert, one behavior per `it`):
  1. `'returns enum values plus the (No Value) sentinel option in filter context (no row)'`
  2. `'returns only the enum values in edit context (row defined)'`
  3. `'sentinel option uses the (No Value) label'`
  4. `'preserves the original enum value order'`
  5. `'is a stable reference identity per call (helper does not mutate input)'`

- **GREEN — implementation**: small pure function, no closures over enums; spread input into a fresh array each invocation.

- **REFACTOR**: extract the literal `(No Value)` to import `NO_VALUE_LABEL` only — no other refactor.

- **Note**: This is the **first usage of MUI X v8 function-form `valueOptions` in `apps/web/`**. Call this out in the PR description. During implementation, manually verify in the running app that the sentinel does NOT appear in the inline-edit dropdown of `orderType` (the simplest enum column to inspect).

- **Commit message**: `[MLID-2213] - feat(orders-filter): add withNoValueOption helper for filter-only enum dropdown option`

### Commit 3 — Wire `withNoValueOption` into all 10 enum columns

- **Files**:
  - `apps/web/app/orders-tracker/Orders/NewOrders.tsx` (Modify — 5 columns)
  - `apps/web/app/orders-tracker/Orders/MaintenanceOrders.tsx` (Modify — 3 columns)
  - `apps/web/app/orders-tracker/Orders/LeadOrder.tsx` (Modify — 2 columns)
  - `apps/web/app/orders-tracker/Orders/NewOrders.test.tsx` (Modify — add 1 test)
  - `apps/web/app/orders-tracker/Orders/MaintenanceOrders.test.tsx` (Modify — add 1 test)
  - `apps/web/app/orders-tracker/Orders/LeadOrder.test.tsx` (Modify — add 1 test)

  This commit touches 6 files but they are tightly coupled (each column file paired with its test). If the user prefers smaller commits we can split into one commit per column file; default is one commit for the wiring.

- **RED — failing tests** (one per column file, smoke level):
  - `'orderType column exposes (No Value) in filter-context valueOptions'` (NewOrders)
  - `'changeType column exposes (No Value) in filter-context valueOptions'` (MaintenanceOrders)
  - `'orderType column exposes (No Value) in filter-context valueOptions'` (LeadOrder)

  Pattern (matches existing test style):
  ```ts
  const cols = getNewOrdersColumns({ onDeleteClick: jest.fn() });
  const orderType = cols.find(c => c.field === 'orderType') as GridSingleSelectColDef;
  const fn = orderType.valueOptions as (p: { field: string }) => ValueOptions[];
  expect(fn({ field: 'orderType' })).toEqual(
    expect.arrayContaining([{ value: NO_VALUE_SENTINEL, label: '(No Value)' }])
  );
  ```

- **GREEN — implementation**: replace `valueOptions: Object.values(SomeEnum)` with `valueOptions: withNoValueOption(Object.values(SomeEnum))` on all 10 columns. Mechanical, single-line per column.

- **REFACTOR**: none — just verify all existing column tests still pass.

- **Manual verification (BLOCK before committing this commit per `feedback_visual_test_before_commit`)**:
  1. `npm run dev:web`, navigate to `/orders-tracker/new`.
  2. Open the filter panel for "Order Type" column → confirm `(No Value)` appears at the bottom of the value list.
  3. Click the cell to inline-edit "Order Type" → confirm `(No Value)` does NOT appear in the edit dropdown.
  4. Repeat spot-check on Maintenance tab (`changeType`) and Lead tab (`orderType`).
  5. Apply `Order Type is (No Value)` → results should be orders with missing/null/empty `orderType`.
  6. Apply `Order Type is any of [SomeRealValue, (No Value)]` → results union of matching real value plus null orders.

- **Commit message**: `[MLID-2213] - feat(orders-filter): show (No Value) option on all enum filter dropdowns across new/maintenance/lead orders`

### Commit 4 — Chip label mapping (inline in page.tsx)

- **Files**:
  - `apps/web/app/orders-tracker/[category]/page.tsx` (Modify — update inline `formatFilterChipLabel`)
  - `apps/web/app/orders-tracker/[category]/page.test.tsx` (Modify — add tests inside the existing `'Filter chip label formatting'` describe block at lines 677–701, reusing the existing `renderWithFilters` helper)

- **Behavior change** (in the inline `formatFilterChipLabel` at `page.tsx` lines 497–529):
  - Import `NO_VALUE_SENTINEL` from `@/utils/api/orders/gridToMongoQuery` and `NO_VALUE_LABEL` from `@/app/orders-tracker/utils/enumValueOptions`.
  - **Single-value path**: if `item.value === NO_VALUE_SENTINEL`, render value as `(No Value)` (still wrapped in quotes per current style → produces `Order Status is "(No Value)"`).
  - **Array path**: in the existing `nonEmpty.filter((v) => v !== '' && v != null)` step, map any sentinel entry to `NO_VALUE_LABEL` BEFORE truncation/join. Truncation and `+N more` behavior unchanged.

- **RED — failing tests** (added inside the existing `'Filter chip label formatting'` describe in `page.test.tsx`):
  1. `'renders single sentinel value as (No Value) literal in chip text'`
  2. `'renders array containing sentinel with (No Value) substituted, others preserved'`
  3. `'renders array of mixed sentinel + 3 real values with truncation: "A, B +2 more"'` (verifies sentinel counts toward visible/remaining like any other value)
  4. `'renders real enum value unchanged (regression)'`
  5. `'renders date ISO value formatted (regression)'`
  6. `'renders unknown column field with field name as fallback (regression)'`

- **GREEN — implementation**: add sentinel mapping at the top of the array filter step and the single-value branch in the inline function. No extraction.

- **REFACTOR**: none.

- **Manual verification (BLOCK before committing per visual-test feedback)**:
  1. Apply `Order Status is (No Value)` filter → confirm chip reads `Order Status is "(No Value)"` (not `__LI_NO_VALUE__`).
  2. Apply `Order Status is any of [Approved, (No Value)]` → confirm chip reads `Order Status is any of "Approved, (No Value)"`.
  3. Confirm previously-working chips (`contains "john"`, date filters) render unchanged.

- **Commit message**: `[MLID-2213] - feat(orders-filter): render (No Value) sentinel as readable label in active-filter chips`

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/utils/api/orders/gridToMongoQuery.ts` | Modify | Declare and export `NO_VALUE_SENTINEL`. Translate sentinel to `emptyPredicate`/`notEmptyPredicate` for `singleSelect` `is`/`not` and to mixed `$or` for `isAnyOf`. |
| `apps/web/utils/api/orders/gridToMongoQuery.test.ts` | Modify | Add 6 focused `singleSelect` + sentinel tests. |
| `apps/web/app/orders-tracker/utils/enumValueOptions.ts` | Create | `withNoValueOption(enumValues)` returning function-form `valueOptions` that hides sentinel during inline edit. Defines `NO_VALUE_LABEL`. |
| `apps/web/app/orders-tracker/utils/enumValueOptions.test.ts` | Create | Unit tests for filter vs edit context, label, ordering. |
| `apps/web/app/orders-tracker/Orders/NewOrders.tsx` | Modify | Wrap `valueOptions` for `orderType`, `coverageStatus`, `status`, `liPharmacyStatus`, `audit`. |
| `apps/web/app/orders-tracker/Orders/MaintenanceOrders.tsx` | Modify | Wrap `valueOptions` for `changeType`, `coverageStatus`, `status`. |
| `apps/web/app/orders-tracker/Orders/LeadOrder.tsx` | Modify | Wrap `valueOptions` for `orderType`, `status`. |
| `apps/web/app/orders-tracker/Orders/NewOrders.test.tsx` | Modify | One smoke test asserting `(No Value)` is in filter-context options for an enum column. |
| `apps/web/app/orders-tracker/Orders/MaintenanceOrders.test.tsx` | Modify | One smoke test (same pattern). |
| `apps/web/app/orders-tracker/Orders/LeadOrder.test.tsx` | Modify | One smoke test (same pattern). |
| `apps/web/app/orders-tracker/[category]/page.tsx` | Modify | Update inline `formatFilterChipLabel` to map sentinel → `(No Value)` for single and array paths. |
| `apps/web/app/orders-tracker/[category]/page.test.tsx` | Modify | Add 6 tests inside the existing `'Filter chip label formatting'` describe block. |

---

## Testing Strategy

### Unit tests (Jest)
- **Server (`gridToMongoQuery.test.ts`)**: 6 new cases covering `singleSelect` × `{is, not, isAnyOf-only-sentinel, isAnyOf-mixed, is-real-regression, isAnyOf-real-regression}`.
- **Client helper (`enumValueOptions.test.ts`)**: 5 cases — filter context, edit context, label correctness, ordering preservation, no input mutation.
- **Column wiring (3 files)**: 1 smoke test per column file confirming the function-form contract on a representative enum column.
- **Chip label (existing `page.test.tsx`)**: 6 cases added to the `'Filter chip label formatting'` describe block — sentinel single, sentinel array, sentinel + truncation, real enum regression, date regression, unknown field fallback.

### Integration / E2E
- No automated E2E in scope. Browser visual verification (see Manual section) covers the cross-cutting flow.

### Manual verification (must pass before commits 3 and 4)
- [ ] `(No Value)` appears in the filter dropdown for every enum column on all three tabs (New / Maintenance / Lead) — at minimum spot-check `orderType`, `changeType`, `coverageStatus`, `status`, `audit`.
- [ ] `(No Value)` does NOT appear in the inline-edit dropdown for any enum column.
- [ ] Filter "Order Status is (No Value)" returns rows with missing/null/empty status.
- [ ] Filter "Order Status is not (No Value)" excludes those rows.
- [ ] Filter "Order Status is any of [SomeReal, (No Value)]" returns the union.
- [ ] Active-filter chips render `(No Value)` (not `__LI_NO_VALUE__`).
- [ ] Existing string/date filters and chips render unchanged (no regression).

### Quality gates
- `npm run test` (workspace), `npm run types:check`, `npm run lint:fix` — all green from `apps/web/`.
- Coverage on modified files at or above existing thresholds (80%).

---

## Security Considerations

- **Input validation**: `NO_VALUE_SENTINEL` is a server-controlled constant imported by the filter translator; it is not user-supplied free text. The client helper emits it only inside the dropdown's `valueOptions`. The server compares with strict equality (`v === NO_VALUE_SENTINEL`) before substituting predicates, so external callers passing arbitrary strings cannot accidentally trigger the empty/non-empty path.
- **Authorization**: no new endpoints, no new feature flags; existing orders-tracker RBAC and feature-flag gating (`isIgColumsEnabled`) are unchanged.
- **PHI**: enum fields (`orderType`, `coverageStatus`, `status`, `liPharmacyStatus`, `audit`, `changeType`) are not PHI. No PHI leakage paths added.
- **MongoDB injection**: sentinel substitution uses the existing `emptyPredicate` / `notEmptyPredicate` helpers — no string interpolation introduced.

---

## Open Questions

1. **`audit` column + `Audit.NA` semantics.** The `audit` column has `valueFormatter: (_, row) => row.audit === Audit.NA ? '' : row.audit` — cells with `Audit.NA = 'N/A'` LOOK blank but the underlying value is the string `'N/A'`. Should `(No Value)` filtering on `audit` match `'N/A'` rows too?
   - **Recommended default (this plan implements this)**: NO. `(No Value)` matches only true null/missing/`''`. Rows with `audit === 'N/A'` are excluded from `(No Value)` and included in `is not (No Value)`.
   - **If PO disagrees**, implementation cost is one extra branch in the server (`audit`-specific predicate that also matches `Audit.NA`) — but this would couple `gridToMongoQuery` to a domain enum, which we'd rather avoid; alternative is to add an `audit`-column-specific filter operator or normalize `Audit.NA` to `null` on read. Decide before merge.

### Follow-ups (not part of this PR)
- "Active Orders by Due Date" quick filter inserts `undefined` into an `isAnyOf` array hoping to match null status; the server silently drops it. Pre-existing behavior, separate ticket.
