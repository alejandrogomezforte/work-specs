# MLID-2011 — Documents tab: show order dashboard above the table

> **Status:** PROPOSED — awaiting approval before implementation.

## Goal

Render the order's 3-column dashboard (PATIENT / PROVIDER & INSURANCE / FULFILLMENT) at the top of the Documents tab, above the existing "Order Documents" + Mark-all-as-Read strip + table.

**Scope constraint (2026-04-27):** The team is actively working on the Order Dashboard tab and the order layout. This plan therefore touches **only** files under `documents/` plus a new shared component file — no edits to `page.tsx` (Dashboard route) or `layout.tsx`. A future refactor will deduplicate and move the dashboard rendering into the layout for both tabs once the other work lands.

## Branch

`fix/MLID-2011-documents-tab-show-dashboard-card`

Off `epic/MLID-2011-order-document-notifications` (current HEAD `96c9f572`).

## Visual change

### Today (Documents tab)

```
┌─ Card (from layout) ───────────────────────────────────────────────┐
│ Cabenuva · LISA Order #050y · Order Type New Start  · Status: N/A  │ ← layout title row
│ ───────────────────────────────────────────────────────────────────│
│ Order Documents                              [ Mark all as Read ]  │ ← documents/page.tsx
│ ───────────────────────────────────────────────────────────────────│
│ Document Name        Received Date   Category   Status   View      │
│ QA Test              04/03/2026      Insurance  ...      View      │
│ ...                                                                │
└────────────────────────────────────────────────────────────────────┘
```

### After (interim — everything inside the layout's Card)

```
┌─ Card (from layout, unchanged) ────────────────────────────────────┐
│ Inflectra · LISA Order #319308 · Order Type New Start  · Status: ✦│ ← layout title row
│ ───────────────────────────────────────────────────────────────────│
│ PATIENT          │ PROVIDER & INSURANCE     │ FULFILLMENT          │ ← NEW (3-col dashboard)
│ Patient: ...     │ Provider: ...            │ Coverage: ...        │
│ DOB: ...         │ Insurance: ...           │ LI Pharmacy: ...     │
│ ───────────────────────────────────────────────────────────────────│
│ Order Documents                              [ Mark all as Read ]  │ ← existing strip (unchanged)
│ ───────────────────────────────────────────────────────────────────│
│ Document Name        Received     Category                  View   │ ← existing table (unchanged)
│ Insurance Card …     10/8/25      Insurance Card            View   │
│ ...                                                                │
└────────────────────────────────────────────────────────────────────┘
```

### Spec target (achievable later, when layout refactor is unblocked)

The design spec (`Screenshot 2026-04-27 123209.png`) shows the dashboard inside one Card and the strip + table as separate sections **outside** that Card. Splitting the table out of the Card requires modifying `layout.tsx` (move `{children}` for Documents to render below `</Card>`), which the scope constraint forbids today. The interim "everything inside one card" rendering preserves the data and the relative ordering; the future layout refactor will produce the perfect split visual without further changes to `documents/page.tsx`.

**Mark-all-as-Read button placement:** stays in its existing flex strip with `justifyContent: 'space-between'`. No markup change to that strip.

## Files to touch

| File | Change |
|---|---|
| `apps/web/app/orders-tracker/[category]/[orderId]/components/OrderDashboardContent.tsx` | **NEW.** Contains the 3-column dashboard JSX. Mirrors the JSX currently inlined in `page.tsx` (acceptable duplication for now — future refactor will dedupe). |
| `apps/web/app/orders-tracker/[category]/[orderId]/components/OrderDashboardContent.test.tsx` | **NEW.** Component tests covering the 3-column rendering and feature-flag/loading/error guards. |
| `apps/web/app/orders-tracker/[category]/[orderId]/documents/page.tsx` | Render `<OrderDashboardContent />` above the existing section header strip + table. No removal of existing markup. |
| `apps/web/app/orders-tracker/[category]/[orderId]/documents/page.test.tsx` | Add a case asserting `<OrderDashboardContent />` is mounted (mock + presence assertion). |

**Explicitly NOT touched:**

- `apps/web/app/orders-tracker/[category]/[orderId]/page.tsx` (Order Dashboard route — keeps its inlined 3-column JSX).
- `apps/web/app/orders-tracker/[category]/[orderId]/layout.tsx` (Card structure, tab routing, `tabContentInsideCard` wrapper — all unchanged).
- `apps/web/app/orders-tracker/[category]/[orderId]/page.test.tsx`, `layout.test.tsx` (no test changes for those files).

## Approach

1. **Create the component.** `components/OrderDashboardContent.tsx` is a `'use client'` component that mirrors the 3-column JSX from `page.tsx` but **without** the `useFeatureFlag(FeatureFlag.ORDER_DETAILS)` gate. The `ORDER_DETAILS` flag is a visual UI-version toggle (old vs. new style) added by Alberto for [MLID-1661](https://localinfusion.atlassian.net/browse/MLID-1661); we are not perpetuating that pattern in new components.
   - Reads `useOrderDetails()` and returns `null` while `isLoading`, on `error`, or when `data` is missing.
   - Renders the same `<Layout direction="row">` with three `<LayoutItem>` columns and the same field labels and accessors.
2. **Render in the Documents page.** `documents/page.tsx` adds `<OrderDashboardContent />` as the first child inside its existing `<div style={{ display: 'flex', flexDirection: 'column', gap: 16 }}>` wrapper:
   ```tsx
   return (
     <div style={{ display: 'flex', flexDirection: 'column', gap: 16 }}>
       <OrderDashboardContent />
       {sectionHeader}
       <table className={styles.documentsTable}>…</table>
     </div>
   );
   ```
   The section header + table render below it unchanged.
3. **Empty-state handling.** The current "no documents" early return inside `documents/page.tsx` should also render the dashboard above the empty-state message — i.e., `<OrderDashboardContent />` precedes the `sectionHeader` + `<p>No documents available…</p>` block.
4. **No feature-flag gating in the new component.** The `ORDER_DETAILS` flag is a UI-version toggle Alberto added for the original Order Details Header rollout — its purpose was to show a pre-redesign layout when the new one was off. New work in this area should not replicate that pattern; the dashboard renders unconditionally (subject to data being present).

### Why a separate component (vs. inlining the JSX again in `documents/page.tsx`)

- **Future refactor will be a 2-line change.** When the team unblocks layout edits, swapping `page.tsx`'s inline JSX for `<OrderDashboardContent />` is mechanical, and the `documents/page.tsx` import already points at the canonical place.
- **Test surface is smaller.** The 3-column rendering logic gets its own test file rather than being mixed into both pages' tests.
- **No risk to the Dashboard tab.** Because `page.tsx` is untouched, no tests there flip; no chance of accidental render-tree change.

## Alternatives considered (and why rejected)

1. **Modify the layout to render the dashboard for both tabs and split the table outside the Card.** Cleanest end state but violates today's scope constraint (layout edits forbidden).
2. **Inline the 3-column JSX directly into `documents/page.tsx`.** No new file but creates a third copy of the same JSX (Dashboard inline + Documents inline + future layout dedupe). Picking the component approach now means the future refactor only deletes 2 inline copies.
3. **Wrap the table in its own Card to hint at the future visual split.** Adds a Card-in-Card nesting which is visually awkward today and adds a markup pattern we'd remove during the refactor anyway.

## TDD steps

Strict red → green → refactor.

### Phase 1 — `OrderDashboardContent` component

1. **RED:** Create `components/OrderDashboardContent.test.tsx` asserting the 3-column rendering and the loading / error / missing-data null cases. Run — fails (component doesn't exist).
2. **GREEN:** Create `components/OrderDashboardContent.tsx` with the JSX mirrored from `page.tsx`. Run — passes.
3. **REFACTOR:** None expected (mechanical mirror).

### Phase 2 — Wire it into the Documents page

4. **RED:** Add a test in `documents/page.test.tsx` mocking `<OrderDashboardContent />` (e.g., `jest.mock` returning a marker `<div data-testid="dashboard-content" />`) and asserting the marker appears above the existing section header / table. Run — fails.
5. **GREEN:** Import and render `<OrderDashboardContent />` at the top of `documents/page.tsx`'s return tree (both branches — empty-state and populated-table). Run — passes.
6. **REFACTOR:** None expected.

### Phase 3 — Verification

7. Run the affected test files:
   ```
   cd apps/web
   npx jest "app/orders-tracker/\[category\]/\[orderId\]/(components/OrderDashboardContent|documents/page).test.tsx"
   ```
8. `npx tsc --noEmit` clean.
9. `npx prettier --write` on all changed files.
10. Manual QA in the browser:
    - Documents tab on a real order — dashboard renders at top, strip + table below.
    - Empty-documents order — dashboard renders, then the empty-state message.
    - `ORDER_DOCUMENT_NOTIFICATIONS` toggling still gates the New chip + per-row Mark-as-Read button.
    - Order Dashboard tab — visually identical to before.
    - Order History tab — unaffected.

## Risks

| Risk | Mitigation |
|---|---|
| Future layout refactor will need to delete duplicated 3-column JSX from `page.tsx`. | Acknowledged in scope; documented in this plan. The refactor PR will be 1 file delete + 1 import change in `page.tsx`. |
| `<OrderDashboardContent />` and `page.tsx`'s inline JSX could drift if someone edits one but not the other. | Add a comment in `page.tsx` (in the future cleanup PR, not now) once the team's parallel work lands. For this PR, drift risk is low — both copies start identical. |
| Empty-state rendering on Documents tab could look odd with a dashboard above an "No documents available" message. | Manual QA covers this; the visual is "dashboard summary + empty state below," which is reasonable. |

## Out of scope

- No change to `page.tsx` (Order Dashboard route).
- No change to `layout.tsx`.
- No change to `OrderExpandPanel` or the Edit Order button on the Documents tab (Edit Order is rendered by the layout — adding it on Documents requires layout edits, deferred).
- No change to the Order History tab.
- No change to the API or data layer.
- No changes to the documents table itself (columns, sorting, pagination).
- No new feature flag.

## Open questions for approval

1. **Empty-state ordering — dashboard above the empty-state message?** Defaulting to **yes** (described in §3 above). If you prefer the empty state to be a standalone page (no dashboard), say so.

> **Resolved:** The `ORDER_DETAILS` flag question is dropped — that flag is a visual UI-version toggle Alberto added for [MLID-1661](https://localinfusion.atlassian.net/browse/MLID-1661) (old vs. new style). Per Alejandro on 2026-04-27, new components in this area should not replicate that gating pattern. `<OrderDashboardContent />` renders unconditionally (subject to data being present).

## Acceptance criteria

- [ ] Documents tab renders the 3-column dashboard at the top, above the section header strip and the table.
- [ ] Order Dashboard tab is visually unchanged.
- [ ] Order History tab is unaffected.
- [ ] `page.tsx` and `layout.tsx` are untouched (verified via `git diff`).
- [ ] All pre-existing tests still pass.
- [ ] New tests cover `<OrderDashboardContent />` rendering and the Documents page integration.
- [ ] Coverage on touched files ≥ 80% (per CLAUDE.md).
- [ ] No `any` types introduced.
- [ ] `tsc --noEmit` clean, prettier clean.
- [ ] Manual QA per Phase 3 §10 passes.
