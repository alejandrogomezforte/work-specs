# MLID-2307 — Order Details: Financial Review Calculator

## Task Reference

- **Jira**: [MLID-2307](https://localinfusion.atlassian.net/browse/MLID-2307)
- **Parent epic**: [MLID-2206](https://localinfusion.atlassian.net/browse/MLID-2206) — "Order Details v2 - Show Each Step Under Order Tracker" (no active epic branch; treat as standalone)
- **Story Points**: 3 (Stage 1 only; Stage 2 to be sized separately)
- **Branch**: `feature/MLID-2307-financial-review`
- **Base Branch**: `develop`
- **Status**: **Stage 1 complete (mock UI). Stage 2 (data layer) not started — DO NOT SHIP without Stage 2.**

---

## Delivery Stages

This ticket is split into two stages because the PO confirmed mock-only UI for Stage 1:

| Stage | Scope | Status | Mergeable to develop? |
|---|---|---|---|
| **Stage 1 — UI** | Sub-tab routing, components, feature flag, in-memory seed, full TDD coverage | ✅ Complete (2026-05-20) | ❌ No — mock data must be removed first |
| **Stage 2 — Data Layer** | Understand → analyze → design → build the persistence + retrieval layer; replace mock with real data | ⏳ Not started | ✅ Yes, once Stage 2 lands on top |

Stage 1 is committed on `feature/MLID-2307-financial-review` and held locally. The branch is NOT pushed and a PR is NOT opened until Stage 2 work replaces the mock seed with a real backend.

---

## Summary

Add a **Financial Review** sub-tab to the Order Details v2 dashboard panel. The tab is a patient-responsibility calculator with two sub-views: a **list** of saved calculations for the current order (each expandable in place to reveal its breakdown) and a **new-calculation form** with live estimate preview and a save action. Stage 1 ships mock-only data (in-memory seed, no backend); Stage 2 will replace the seed with a real data layer. The feature is gated behind a `ORDER_FINANCIAL_CALCULATION` feature flag at both the tab-visibility (UI) and route-rendering (URL) levels.

This is a mechanical port of a complete playground implementation (`~/Dev/li-ui-playground/src/pages/FinancialReview/`) into the codebase's established orders-tracker patterns, with TDD applied to the pure calculation logic and all new components.

---

## Codebase Analysis

### Route base
All order-detail routes live under `apps/web/app/orders-tracker/[category]/[orderId]/`. The new sub-route will be `financial-review/{,/new}` — two `page.tsx` files, no `[calculationId]` segment (no standalone detail page exists in the playground's designed flow).

### Dashboard structural quirk — critical
There is no `/dashboard` path segment. The "dashboard" is the index route `[orderId]/page.tsx`. `layout.tsx` (lines 569-588) **ignores `children` on the dashboard path** and instead renders `<OrderClinicalReviewsPanel />` directly. The layout uses `isDashboardPath = pathname === basePath` to gate this. Adding Financial Review requires two independent changes:
1. Adding a new tab to `OrderClinicalReviewsPanel`.
2. Adding a new `isFinancialReviewPath` guard in `layout.tsx` so that navigating to `/financial-review` or `/financial-review/new` renders `{children}` instead of the panel.

### `OrderClinicalReviewsPanel` state
`activeReviewTab` is currently hardcoded to `'clinical'` (line 60). `reviewTabItems` is a module-level constant with one entry. The `Tabs.onChange` only handles `'clinical'`. Both must be updated to support the new tab and URL-driven active state.

### Feature flag pattern
`useFeatureFlag(FeatureFlag.X)` is a client-side hook from `@/utils/contexts/FeatureFlagContext`. Flag enum and `DEFAULT_FEATURE_FLAGS` map both live in `apps/web/utils/featureFlags/featureFlags.ts`. No DB document is created until an admin toggles the flag in `/admin/feature-flags`.

### Session import pattern (for author identity)
`import { useSession } from 'next-auth/react'` with `const { data: session } = useSession()` is the established pattern (see `apps/web/app/orders-tracker/components/EditAssignedTo.tsx`). Use `session?.user?.email` for the author field on fake saves; fall back to `'unknown@mylocalinfusion.com'`.

### Clinical-reviews as structural precedent
`apps/web/app/orders-tracker/[category]/[orderId]/clinical-reviews/` contains `page.tsx`, `new/page.tsx`, `[reviewId]/page.tsx`, and `_components/` with all partials and hooks. All pages are `'use client'`, use `useParams()`, `useOrderDetails()`, and build `basePath` locally. Financial Review mirrors this pattern minus the `[reviewId]` segment.

### `@repo/ui` availability — all confirmed
`Layout` (including `Layout.Item grow="1"`), `Typography`, `Button`, `Card`, `Chip` (with `color="primary" variant="soft"`), `Divider`, `TextField`, `Snackbar`, `IconButton`, and icons `ContentCopyIcon`, `CheckIcon`, `ExpandMoreIcon`, `AddIcon` are all available in `@repo/ui`.

### Calculation algorithm
The `calculate()` function in the playground (`src/data/financialCalculations.ts`) is a pure function with no side effects. It must be ported verbatim. Four seed scenarios in `financialCalculations` give us deterministic expected outputs for TDD fixtures.

### No existing test file for `OrderClinicalReviewsPanel`
The `components/` directory contains `OrderExpandPanel.test.tsx` but no test for `OrderClinicalReviewsPanel`. We will create one as part of the tab-integration step.

---

## Architecture Decisions

### Routes — two pages, no catch-all
`financial-review/page.tsx` (list) and `financial-review/new/page.tsx` (new-calculation form). Each is `'use client'`, calls `useParams()` and `useOrderDetails()`, and delegates to a `_components/` partial. No `[calculationId]/page.tsx` — the playground's detail page is unreachable from the designed UI flow and is out of scope.

### Fake data — in-memory Context, no persistence
A `FinancialCalculationsContext` is created inside `financial-review/layout.tsx`, providing a `useState`-backed list seeded from the static `seed.ts` array. State is shared across the list and new-calculation form through this context. Data is lost on full page refresh — that is the correct behavior for a mock and surfaces clearly to QA. No localStorage, no sessionStorage, no Mongoose model, no API route.

### Feature flag gating — belt-and-suspenders
1. **Tab visibility** in `OrderClinicalReviewsPanel.tsx`: the Financial Review tab item is conditionally included in the tabs array based on `useFeatureFlag(FeatureFlag.ORDER_FINANCIAL_CALCULATION)`.
2. **Route gating** in `layout.tsx`: a new `isFinancialReviewPath = !!basePath && pathname?.startsWith(`${basePath}/financial-review`)` boolean drives a new `{children}` block gated by `isFinancialReviewEnabled && !isLoading && !error`. Navigating to the URL with the flag off renders nothing.

### Active tab — URL-driven derivation
`activeReviewTab` in `OrderClinicalReviewsPanel` is derived from `usePathname()`: if path contains `/clinical-reviews` → `'clinical'`; if path contains `/financial-review` → `'financial'`; else `'clinical'`. `Tabs.onChange` handles both values and routes accordingly. `showNewReviewAction` becomes dependent on the derived active tab.

### Component structure — playground port, adapted

| Playground source | Our destination | Notes |
|---|---|---|
| `FinancialReviewIndex.tsx` (list + `CalcRowCard`) | `_components/FinancialReviewIndex.tsx` + `_components/CalculationCard.tsx` | `CalcRowCard` extracted and renamed |
| `Calculator.tsx` | `_components/Calculator.tsx` | `useNavigate` → `useRouter`; module-level `addCalculation` → context mutation |
| `CalculationBreakdown.tsx` | `_components/CalculationBreakdown.tsx` | Keep compact-mode path only; drop dark-teal headline card (detail page is out of scope) |
| `CalculationDetail.tsx` | **Not ported** | Unreachable in playground UX flow |
| `data/financialCalculations.ts` types + `calculate()` | `_components/calculate.ts` | Pure function + types only |
| `data/financialCalculations.ts` seed array | `_components/seed.ts` | Typed static array |
| `data/financialCalculations.ts` mutations/getters | `_components/FinancialCalculationsContext.tsx` | `useState` + React Context replaces module-level mutation |

---

## Implementation Steps

Each step is one commit. Steps are ordered RED → GREEN → REFACTOR within each commit, and sequentially across commits so earlier commits are always in a passing state.

---

### Step 1 — Feature flag registration

**Files**
- `apps/web/utils/featureFlags/featureFlags.ts` — Modify

**What**
Add `ORDER_FINANCIAL_CALCULATION = 'ORDER_FINANCIAL_CALCULATION'` to the `FeatureFlag` enum (alongside the other order-detail tab flags, after `ORDER_DATES_RESTRICTION`). Add `[FeatureFlag.ORDER_FINANCIAL_CALCULATION]: false` to `DEFAULT_FEATURE_FLAGS` with a comment noting it is disabled until QA sign-off. No DB document is created; the admin panel creates it on first toggle.

**Test cases**
This step has no behavior to unit-test beyond TypeScript compilation. The flag will be exercised in later steps' tests. TypeScript strict mode will catch the missing entry in `DEFAULT_FEATURE_FLAGS` at compile time — that is the test.

**Commit message**
```
[MLID-2307] - chore(feature-flags): register ORDER_FINANCIAL_CALCULATION feature flag, default false
```

---

### Step 2 — Pure calculation function (TDD core)

**Files**
- `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/calculate.ts` — Create
- `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/calculate.test.ts` — Create

**What**
Port `CalculationInputs`, `CalculationResult`, `FinancialCalculation` interfaces and the `calculate()` and `fmt()` functions verbatim from the playground. Types are co-located here (not in `apps/web/types/`) because they are mock-only for now. `getCalculation` (used only by the out-of-scope detail page) is **not** ported.

**RED phase — write these tests first, all failing:**

```
describe('calculate', () => {
  describe('seed scenario c1 — zero deductible, zero coinsurance, assistance exceeds costs', () => {
    it('should return drugResp=0, adminResp=0, totalResp=0')
    it('should include the zero-deductible-zero-coinsurance summary lines')
  })

  describe('seed scenario c2 — partial deductible, 20% coinsurance, drug assistance larger than drug responsibility', () => {
    it('should return drugResp=0, adminResp=64, totalResp=64')
    it('should include deductible-remaining and coinsurance summary lines')
    it('should include copay-assistance summary line')
  })

  describe('seed scenario c3 — deductible fully met, 10% coinsurance, no assistance', () => {
    it('should return drugResp=420, adminResp=32, totalResp=452')
    it('should include deductible-fully-met and coinsurance summary lines')
  })

  describe('seed scenario c4 — partial deductible, 30% coinsurance, both assistance amounts', () => {
    it('should return drugResp=1480, adminResp=0, totalResp=1480')
  })

  describe('OOP cap scaling edge case', () => {
    it('should scale both drug and admin legs proportionally when pre-assistance total exceeds remaining OOP')
    it('should not apply OOP scaling when oopMax is 0')
  })

  describe('assistance overflow edge case', () => {
    it('should clamp admin responsibility to 0 when admin assistance exceeds admin responsibility without moving overflow to drug')
  })

  describe('fmt', () => {
    it('should format a number with two decimal places and comma separators')
    it('should return "0.00" for null')
    it('should return "0.00" for NaN')
  })
})
```

**GREEN phase** — port the function body verbatim from the playground.

**REFACTOR phase** — extract `applyDeductible` and `applyCoinsurance` as private helpers if it improves readability; keep `summaryLines` generation inline.

**Commit message**
```
[MLID-2307] - feat(financial-review): add calculate() pure function with types and fmt() helper

Ports the patient-responsibility calculation algorithm from the playground
verbatim. Tests cover all four seed scenarios plus OOP-cap scaling and
assistance-overflow edge cases to prevent regressions when real data wiring
is added in a follow-up ticket.
```

---

### Step 3 — Static seed data

**Files**
- `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/seed.ts` — Create

**What**
Export the four `FinancialCalculation` objects from the playground's `financialCalculations` array as a typed constant `SEED_CALCULATIONS`. The `orderId` field values from the playground (`'319309'`) will be overridden at runtime by the context (the seed is filtered by the real `orderId` from params). Keep the seed strings as-is to make the data recognizable in QA.

No test needed for a pure static data file — correctness is guaranteed by TypeScript's structural type checking against `FinancialCalculation[]`.

**Commit message**
```
[MLID-2307] - chore(financial-review): add seed calculations for mock data display
```

---

### Step 4 — In-memory data context

**Files**
- `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/FinancialCalculationsContext.tsx` — Create
- `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/FinancialCalculationsContext.test.tsx` — Create
- `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/layout.tsx` — Create

**What**

`FinancialCalculationsContext.tsx` exports:
- `FinancialCalculationsContext` (React Context, initial value `undefined`)
- `FinancialCalculationsProvider` — a `'use client'` component that owns a `useState<FinancialCalculation[]>` seeded from `SEED_CALCULATIONS`. Exposes `calculations`, `addCalculation(calc: FinancialCalculation): void`, and `getCalculationsForOrder(orderId: string): FinancialCalculation[]` through the context value.
- `useFinancialCalculations()` — throws if used outside the provider; returns the context value.

Context value type (not exported from `types/`, lives inline):
```ts
type FinancialCalculationsContextValue = {
  calculations: FinancialCalculation[];
  addCalculation: (calc: FinancialCalculation) => void;
  getCalculationsForOrder: (orderId: string) => FinancialCalculation[];
};
```

`financial-review/layout.tsx` — a minimal `'use client'` layout that wraps `{children}` in `<FinancialCalculationsProvider>`. This ensures state survives navigation between `/financial-review` and `/financial-review/new` within the same browser session.

**RED phase tests:**

```
describe('FinancialCalculationsProvider', () => {
  beforeEach(() => jest.clearAllMocks())

  it('should provide calculations seeded from SEED_CALCULATIONS')
  it('should filter calculations by orderId via getCalculationsForOrder')
  it('should add a new calculation and make it visible in the list')
  it('should prepend the new calculation so it appears first')
})

describe('useFinancialCalculations', () => {
  it('should throw when used outside FinancialCalculationsProvider')
})
```

**Commit message**
```
[MLID-2307] - feat(financial-review): add in-memory FinancialCalculationsContext with layout provider

State is intentionally volatile (in-memory only, lost on page refresh) to
accurately reflect the mock-data scope of this ticket. The context layout
preserves state during client-side navigation between the list and new-
calculation pages within the financial-review subtree.
```

---

### Step 5 — `CalculationBreakdown` component

**Files**
- `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/CalculationBreakdown.tsx` — Create
- `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/CalculationBreakdown.test.tsx` — Create

**What**
Port `CalculationBreakdown.tsx` from the playground. Keep only the **compact-mode** code path (inputs grid + summary panel + drug/admin split bar). Drop the dark-teal headline card branch entirely — it was built for the out-of-scope standalone detail page. The component renders as `'use client'` and receives `inputs: CalculationInputs` and `result: CalculationResult` as props. No `mode` prop — compact is the only mode.

Drug/admin split bar renders only when both `drugResp > 0` and `adminResp > 0`, preventing division by zero.

**RED phase tests:**

```
describe('CalculationBreakdown', () => {
  beforeEach(() => jest.clearAllMocks())

  it('should render all input values in the grid')
  it('should render all summary lines')
  it('should render the drug/admin split bar when both values are non-zero')
  it('should not render the split bar when drugResp is 0')
  it('should not render the split bar when adminResp is 0')
  it('should display formatted currency values via fmt()')
})
```

**Commit message**
```
[MLID-2307] - feat(financial-review): add CalculationBreakdown component (compact mode only)

Omits the dark-teal detail-page mode from the playground source since there
is no standalone calculation detail route in the designed UX flow.
```

---

### Step 6 — `CalculationCard` component

**Files**
- `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/CalculationCard.tsx` — Create
- `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/CalculationCard.test.tsx` — Create

**What**
Extract and port the `CalcRowCard` inline component from the playground's `FinancialReviewIndex.tsx`, renamed to `CalculationCard`. This component owns:
- The collapsed row: date, author, total responsibility, "Latest" chip (shown when `isLatest` prop is `true`), expand/collapse toggle.
- The expanded state: mounts `<CalculationBreakdown inputs={...} result={...} />` below the row.
- The copy-summary action: calls `navigator.clipboard.writeText()` with a multi-line string composed from `result.summaryLines`, then briefly swaps the copy icon for a check icon (1.5 s).

Props:
```ts
type CalculationCardProps = {
  calculation: FinancialCalculation;
  isLatest: boolean;
};
```

**RED phase tests:**

```
describe('CalculationCard', () => {
  beforeEach(() => jest.clearAllMocks())

  it('should render the calculation date and author')
  it('should render the total responsibility formatted as currency')
  it('should show the "Latest" chip when isLatest is true')
  it('should not show the "Latest" chip when isLatest is false')
  it('should not render CalculationBreakdown when collapsed')
  it('should render CalculationBreakdown when expanded after clicking the toggle')
  it('should collapse again when the toggle is clicked a second time')
  it('should call navigator.clipboard.writeText with the summary lines joined by newline on copy')
  it('should briefly show the check icon after copying and restore the copy icon after 1500ms')
})
```

Mock `navigator.clipboard.writeText` via `Object.defineProperty` in the test setup.

**Commit message**
```
[MLID-2307] - feat(financial-review): add CalculationCard component with expand-in-place breakdown

Extracted from the playground's CalcRowCard inline component and given its
own file per naming conventions. The clipboard copy action is tested with a
mocked navigator.clipboard.
```

---

### Step 7 — `FinancialReviewIndex` component (list view)

**Files**
- `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/FinancialReviewIndex.tsx` — Create
- `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/FinancialReviewIndex.test.tsx` — Create

**What**
Port `FinancialReviewIndex.tsx` from the playground. The component:
- Calls `useFinancialCalculations()` and `useParams()` to get the current `orderId`.
- Filters calculations via `getCalculationsForOrder(orderId)`.
- Renders an empty state with a "Start a new calculation" CTA when the list is empty (verbatim from playground design).
- Renders a list of `<CalculationCard>` in order (latest first), passing `isLatest={index === 0}` to the first card.

Props: none (self-contained, reads from context and params).

**RED phase tests:**

```
describe('FinancialReviewIndex', () => {
  beforeEach(() => jest.clearAllMocks())

  it('should render the empty state when no calculations exist for the order')
  it('should render the empty-state CTA that links to the new-calculation page')
  it('should render one CalculationCard per calculation')
  it('should pass isLatest=true only to the first card')
  it('should pass isLatest=false to all subsequent cards')
  it('should show calculations for the current orderId only')
})
```

Mock `useFinancialCalculations` and `useParams` in tests.

**Commit message**
```
[MLID-2307] - feat(financial-review): add FinancialReviewIndex list view component

Renders saved calculations as expandable CalculationCard rows. Empty state
matches playground design verbatim. Filtering by orderId ensures calculations
from seed data do not bleed across different order detail pages.
```

---

### Step 8 — `Calculator` component (new-calculation form)

**Files**
- `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/Calculator.tsx` — Create
- `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/Calculator.test.tsx` — Create

**What**
Port `Calculator.tsx` from the playground. Key adaptations:
- `useNavigate()` from react-router → `useRouter()` from `next/navigation`. After save, call `router.push(`${basePath}`)` to navigate back to the list.
- Module-level `addCalculation()` → `useFinancialCalculations().addCalculation()` from context.
- Author field: `useSession()` from `next-auth/react` → `session?.user?.email ?? 'unknown@mylocalinfusion.com'`.
- ID generation: `crypto.randomUUID()` (available in modern browsers and Node 18+).
- ESC keyboard handler: window-level `keydown` listener that calls `router.push(basePath)` when `event.key === 'Escape'`, cleaned up via `useEffect` return.
- Input masking: each numeric `TextField` uses `onChange` handler that strips non-numeric/non-decimal characters (`value.replace(/[^\d.]/g, '')`).
- Live estimate sidebar: calls `calculate(inputs)` on every input change and renders the result immediately. Disabled (shows dashes) when all inputs are zero.
- Save button: disabled when all input fields are zero/empty.

Props: `basePath: string` passed from the page.

**RED phase tests:**

```
describe('Calculator', () => {
  beforeEach(() => jest.clearAllMocks())

  it('should render all input fields')
  it('should disable the Save button when all inputs are zero')
  it('should enable the Save button when at least one input is non-zero')
  it('should update the live estimate when an input changes')
  it('should call addCalculation with correct data on save')
  it('should navigate to basePath after saving')
  it('should navigate to basePath when ESC key is pressed')
  it('should strip non-numeric characters from numeric inputs')
  it('should use session email as author when session is available')
  it('should fall back to "unknown@mylocalinfusion.com" when session is not available')
})
```

Mock `useRouter`, `useSession`, `useFinancialCalculations`, and `crypto.randomUUID` in tests.

**Commit message**
```
[MLID-2307] - feat(financial-review): add Calculator component with live estimate and save action

Ports the new-calculation form from the playground. State is wired through
FinancialCalculationsContext so saved calculations appear immediately in the
list view on navigation back. ESC key and save button navigate to the list.
Author is drawn from NextAuth session with a safe fallback.
```

---

### Step 9 — Route page shells

**Files**
- `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/page.tsx` — Create
- `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/new/page.tsx` — Create

**What**
Two thin `'use client'` page components following the clinical-reviews pattern:

`financial-review/page.tsx`:
- Calls `useParams<{ category: string; orderId: string }>()`.
- Returns `null` while params resolve.
- Renders `<FinancialReviewIndex />`.

`financial-review/new/page.tsx`:
- Calls `useParams()` and `useOrderDetails()`.
- Builds `basePath` and `financialReviewBasePath`.
- Returns a loading/error guard identical to the clinical-reviews new-page pattern.
- Renders `<Calculator basePath={financialReviewBasePath} />`.

These pages are intentionally thin. No component tests needed for the shells — behavior is tested in the components themselves. Verify via the manual visual walkthrough in Step 11.

**Commit message**
```
[MLID-2307] - feat(financial-review): add page shells for list and new-calculation routes

Mirrors the clinical-reviews page structure. Shells are thin wrappers that
resolve params and delegate entirely to _components/ partials.
```

---

### Step 10 — Tab integration and layout gating

**Files**
- `apps/web/app/orders-tracker/[category]/[orderId]/components/OrderClinicalReviewsPanel.tsx` — Modify
- `apps/web/app/orders-tracker/[category]/[orderId]/components/OrderClinicalReviewsPanel.test.tsx` — Create
- `apps/web/app/orders-tracker/[category]/[orderId]/layout.tsx` — Modify

**What — `OrderClinicalReviewsPanel.tsx`**

Replace the hardcoded `activeReviewTab = 'clinical'` with a `usePathname()`-derived value:
```ts
const pathname = usePathname();
const activeReviewTab = useMemo(() => {
  if (pathname?.includes('/financial-review')) return 'financial';
  return 'clinical';
}, [pathname]);
```

Replace the module-level constant `reviewTabItems` with a `useMemo` that conditionally includes the Financial Review tab based on the feature flag:
```ts
const isFinancialReviewEnabled = useFeatureFlag(FeatureFlag.ORDER_FINANCIAL_CALCULATION);

const reviewTabItems = useMemo(() => [
  { label: 'Clinical Reviews', value: 'clinical' },
  ...(isFinancialReviewEnabled ? [{ label: 'Financial Review', value: 'financial' }] : []),
], [isFinancialReviewEnabled]);
```

Extend `Tabs.onChange` to handle `'financial'`:
```ts
onChange={(value) => {
  if (value === 'clinical') router.push(`${basePath}/clinical-reviews`);
  if (value === 'financial') router.push(`${basePath}/financial-review`);
}}
```

`showNewReviewAction` becomes `activeReviewTab === 'clinical'` (unchanged behavior for Clinical Reviews; Financial Review has its own "New Calculation" button inside `FinancialReviewIndex`).

**What — `layout.tsx`**

Add one feature flag read:
```ts
const isFinancialReviewEnabled = useFeatureFlag(FeatureFlag.ORDER_FINANCIAL_CALCULATION);
```

Add one path check (after `isClinicalReviewsPath`):
```ts
const isFinancialReviewPath =
  !!basePath && pathname?.startsWith(`${basePath}/financial-review`);
```

Add one `{children}` block (after the clinical-reviews block, before the dashboard block):
```ts
{isFinancialReviewPath && isFinancialReviewEnabled && !isLoading && !error
  ? children
  : null}
```

The existing `isDashboardPath` block and `<OrderClinicalReviewsPanel />` rendering are **not changed** — the panel itself now handles the Financial Review tab internally.

**RED phase tests for `OrderClinicalReviewsPanel.test.tsx` — simple interaction tests only**

Scope confirmed: this is a thin interaction test of the tab-related behavior we are adding. We do not re-test the existing clinical-reviews rendering, `<ReviewHistoryList>`, or data loading paths — they are covered by their own test files. Keep the file under ~100 LOC.

```
describe('OrderClinicalReviewsPanel — tab interaction', () => {
  beforeEach(() => jest.clearAllMocks())

  it('should hide the Financial Review tab when ORDER_FINANCIAL_CALCULATION flag is off')
  it('should show the Financial Review tab when ORDER_FINANCIAL_CALCULATION flag is on')
  it('should mark Clinical Reviews active when pathname does not include /financial-review')
  it('should mark Financial Review active when pathname includes /financial-review')
  it('should push to ${basePath}/clinical-reviews when the Clinical Reviews tab is clicked')
  it('should push to ${basePath}/financial-review when the Financial Review tab is clicked')
})
```

Mock surface (shape-only stubs):
- `useFeatureFlag` — return boolean per test
- `usePathname` — return controlled string per test
- `useRouter` — return `{ push: jest.fn() }`
- `useParams` — return `{ category: 'new', orderId: '319309' }`
- `useOrderDetails` — return `{ data: null, isLoading: false, error: null }` (we never read the data in these tests)
- `useClinicalReviews` — return `{ reviews: [], isLoading: false }` (shape only; we never inspect the list)

**Commit message**
```
[MLID-2307] - feat(financial-review): wire Financial Review tab into panel and layout

OrderClinicalReviewsPanel gains a flag-gated Financial Review tab with URL-
driven active state derived from usePathname(). layout.tsx gains an
isFinancialReviewPath guard so the route renders children when the flag is
enabled and the path matches. Navigating the URL with the flag off renders
nothing, satisfying the belt-and-suspenders gating requirement.
```

---

### Step 11 — Visual verification (pre-commit gate)

**No code changes in this step.** This is a mandatory pause for browser-side visual verification before committing UI changes.

1. Run `npm run dev:web` from the repo root.
2. Navigate to `/admin/feature-flags` and enable `ORDER_FINANCIAL_CALCULATION`.
3. Open any order in `/orders-tracker/new/[orderId]`.
4. Verify the **Financial Review** tab appears in the panel header alongside Clinical Reviews.
5. Click **Financial Review** — verify the tab highlights and the list view renders with seed calculations.
6. Click a calculation row — verify it expands in place with the breakdown grid and summary lines.
7. Click the copy icon — verify the check icon briefly appears.
8. Click **New Calculation** — verify the calculator form renders with all input fields and a live estimate sidebar.
9. Enter values — verify the live estimate updates in real time.
10. Press **ESC** — verify navigation back to the list view.
11. Fill in values and click **Save** — verify the new calculation appears at the top of the list.
12. Refresh the page — verify the newly saved calculation is gone (expected mock behavior).
13. Disable `ORDER_FINANCIAL_CALCULATION` in `/admin/feature-flags`. Verify the tab disappears and the URL `/financial-review` renders blank.

**Wait for user's browser-side visual confirmation before proceeding to Step 12.**

---

### Step 12 — PR document (DEFERRED to Stage 2)

**Files**
- `docs/agomez/PR/MLID-2307-financial-review-calculator.md` — Create (deferred)

**What**
Deferred until Stage 2 lands. No PR is opened with mock data in the branch. When Stage 2 replaces the seed with a real data layer, the PR draft will be written and the branch will be pushed for review.

**Commit message**
```
[MLID-2307] - docs(financial-review): add PR draft document
```

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/utils/featureFlags/featureFlags.ts` | Modify | Add `ORDER_FINANCIAL_CALCULATION` to enum and `DEFAULT_FEATURE_FLAGS` (default `false`) |
| `apps/web/app/orders-tracker/[category]/[orderId]/layout.tsx` | Modify | Add `isFinancialReviewEnabled` flag read, `isFinancialReviewPath` guard, and `{children}` block |
| `apps/web/app/orders-tracker/[category]/[orderId]/components/OrderClinicalReviewsPanel.tsx` | Modify | Add flag-gated Financial Review tab, URL-driven `activeReviewTab`, extended `onChange` handler |
| `apps/web/app/orders-tracker/[category]/[orderId]/components/OrderClinicalReviewsPanel.test.tsx` | Create | Tests for flag-gated tab visibility and URL-driven active state |
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/layout.tsx` | Create | Thin layout wrapping children in `FinancialCalculationsProvider` |
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/page.tsx` | Create | List view page shell |
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/new/page.tsx` | Create | New-calculation page shell |
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/calculate.ts` | Create | `CalculationInputs`, `CalculationResult`, `FinancialCalculation` types + `calculate()` + `fmt()` |
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/calculate.test.ts` | Create | TDD tests for all calculation paths and edge cases |
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/seed.ts` | Create | `SEED_CALCULATIONS` static array typed as `FinancialCalculation[]` |
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/FinancialCalculationsContext.tsx` | Create | `useState`-backed context provider + `useFinancialCalculations()` hook |
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/FinancialCalculationsContext.test.tsx` | Create | Context provider behavior tests |
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/CalculationBreakdown.tsx` | Create | Compact-mode breakdown: inputs grid, summary lines, drug/admin split bar |
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/CalculationBreakdown.test.tsx` | Create | Rendering tests for all breakdown elements and split-bar visibility conditions |
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/CalculationCard.tsx` | Create | Collapsed row + expand toggle + inline breakdown + copy action |
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/CalculationCard.test.tsx` | Create | Tests for expand/collapse, Latest chip, copy icon swap |
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/FinancialReviewIndex.tsx` | Create | List view: renders `CalculationCard` rows or empty state |
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/FinancialReviewIndex.test.tsx` | Create | Tests for list rendering, empty state, Latest chip delegation |
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/Calculator.tsx` | Create | New-calculation form: inputs, live estimate, save, ESC handler |
| `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/_components/Calculator.test.tsx` | Create | Tests for form state, live estimate accuracy, save flow, navigation |
| `docs/agomez/PR/MLID-2307-financial-review-calculator.md` | Create | PR draft document |

---

## Testing Strategy

### Unit tests — `calculate.ts`
The pure function is the highest-value test target because it encodes business logic. Cover:
- All four playground seed scenarios with exact `drugResp`/`adminResp`/`totalResp` values.
- OOP cap scaling: construct an input where `preAssistTotal > remainingOOP` and assert both legs are scaled (not just one capped).
- Assistance overflow: assert that admin assistance exceeding admin responsibility does not transfer to drug.
- Zero-deductible zero-coinsurance branch: assert the two special summary lines appear.
- `fmt()`: null, NaN, integers, decimals, values requiring comma separators.

### Component tests — React Testing Library
All component tests use RTL's `render`, `screen`, `fireEvent`/`userEvent`. Mock at boundaries:
- `useFinancialCalculations` — return controlled fixture data.
- `useParams` — return `{ category: 'new', orderId: '319309' }`.
- `useRouter` — return `{ push: jest.fn() }`.
- `usePathname` — return a controlled string.
- `useFeatureFlag` — return `true` or `false` per test case.
- `useSession` — return `{ data: { user: { email: 'test@example.com' } } }` or `{ data: null }`.
- `navigator.clipboard.writeText` — `jest.fn().mockResolvedValue(undefined)`.
- `crypto.randomUUID` — `jest.fn().mockReturnValue('test-uuid')`.

### Integration — route shells
The page shells (`page.tsx` files) are not individually tested. Their correctness is verified by the manual walkthrough and by the component tests that cover the partials they render.

### Manual visual verification (Step 11)
Thirteen explicit steps covering: tab appears/disappears with flag, navigation between tabs updates active state, list renders seed data, expand/collapse works, copy action works, new-calculation form live estimate is accurate, save appends to list, refresh resets state (expected), URL-only access with flag off renders blank.

---

## Security Considerations

### Input validation
All numeric inputs are client-side masked to `\d.` characters only. No server-side validation is needed because no data reaches the backend. The `calculate()` function uses `Math.max(0, ...)` clamping throughout, making it safe against negative inputs.

### Authorization
The feature is gated at both the UI tab level (flag check in `OrderClinicalReviewsPanel`) and the route level (`layout.tsx` children gate). No RBAC roles are required for this feature beyond the existing order-details access controls — any user who can view the order details page can see the tab when the flag is on.

### PHI considerations
No patient health information is displayed, computed, or stored. The calculator inputs are insurance plan financial parameters (deductible amounts, coinsurance percentages, OOP amounts, drug costs) and copay assistance amounts — these are financial estimates, not clinical data. No PHI masking is required. Do not log input values; they are not logged anywhere in the current implementation.

### Feature flag
The default is `false`, so no user can access this surface in production until an admin deliberately enables it. This is consistent with the project's policy for all new order-detail tab features.

---

## Risks and Open Questions

### Resolved — Stage 1 scope (mock-only UI)
Confirmed by the user (2026-05-20): this ticket is **Stage 1, UI only**. Real data wiring is a deliberate follow-up ticket. No Mongoose schema, no API route, no service layer, no pulse job. The plan proceeds as written.

### Resolved — Tab ordering
Confirmed by the user (2026-05-20): adding Financial Review as the **second** tab (index 1) is correct. If/when Auth Review is implemented, that ticket can reorder the list with a one-line change. No action needed in this ticket.

### Resolved — `OrderClinicalReviewsPanel` test scope
Confirmed by the user (2026-05-20): the new `OrderClinicalReviewsPanel.test.tsx` is a **simple interaction test only**. It tests:
- The Financial Review tab is hidden when the flag is off and visible when on.
- Clicking each tab calls `router.push()` with the correct path.
- `activeReviewTab` reflects the pathname.

It does **not** test the existing clinical-reviews rendering, `<ReviewHistoryList>`, or any data loading. Mock `useClinicalReviews`, `useOrderDetails`, `useFeatureFlag`, `useRouter`, and `usePathname` at the absolute minimum — return shape-only stubs and accept that deeper integration is covered elsewhere. Keep the file under ~100 LOC.

### Risk — Layout `children` gate ordering
The `layout.tsx` file has a precise sequence of `{children}` blocks. Inserting the `isFinancialReviewPath` block in the wrong position (e.g., after the `isDashboardPath` block) would cause it to never render. The plan places it between the `isClinicalReviewsPath` block and the `isDashboardPath` block. This must be verified visually in Step 11.

### Risk — Fake save disappears on refresh
This is expected and intentional behavior for mock-only scope. QA must be informed. Document in the PR description.

---

## Stage 2 — Data Layer (next phase, planning)

This section captures the data-layer findings discovered after the Stage 1 commit and the architectural decision still open between two options. It is **not** the final Stage 2 plan — that will come from a `/plan-task` pass once Option A or Option B is locked in. This is a forward-looking note that the planning conversation builds on.

### Goal
Replace the in-memory `FinancialCalculationsContext` + `seed.ts` with a real persistence and retrieval layer **scoped to the order** (not the patient), so the Financial Review tab can ship to production. Once Stage 2 lands, the `getCalculationsForOrder` orderId-remap shim is removed and per-order scoping is enforced at the data layer.

### PO direction (2026-05-20)
After a call with the PO, the data model for the new Order Details Financial Review **must be order-scoped, not patient-scoped**. The legacy `/patient/[id]/financial-calculator` surface (which writes to `patients.financialCalculationsHistory`) is the closest behavioral precedent but it is the **wrong** anchor for the new UI. The new calculator is tied to a specific order's financial review step. Patient-level history is not what PO wants surfaced here.

### Codebase analysis — what exists today (legacy patient-scoped flow)

The Stage 1 UI is a near-clone of the existing patient-scoped financial calculator. Everything below is real, in `develop`, and lives next to where new code will go:

| Concern | Location | Notes |
|---|---|---|
| Route | `apps/web/app/patient/[patientId]/financial-calculator/page.tsx` | Renders `<FinancialCalculatorModule />` inside `<PatientTabsModule>` |
| UI list + form module | `apps/web/components/Modules/financial-calculator/` | `financialCalculator.tsx` (list), `calculatorSidebar/calculatorSidebar.tsx` (drawer form), `calculatorHistoryItem/calculatorHistoryItem.tsx` (expandable row), `calculatorDynamicFormItems/calculatorDynamicFormItems.tsx` (form fields) |
| Form fields | `calculatorDynamicFormItems.tsx` | Deductible met/max, coinsurance %, OOP met/max, drug cost, admin cost, drug assistance, admin assistance — same 9 inputs as Stage 1 |
| API route | `apps/web/pages/api/patients/[id]/financialCalculation.ts` | `POST` only. Runs `calculateMedicalCosts()` (pure function) **plus** an Azure OpenAI call (`gpt-5-nano`) that produces a "Reason" appendix in markdown. Result + markdown blob are pushed onto the patient subdocument. |
| Service layer (data write) | `apps/web/services/mongodb/financialCalculation.ts` | One function, `insertFCItem`, that does `db.collection('patients').updateOne({ _id }, { $push: { financialCalculationsHistory: { _id: ObjectId, ...newFCItem } } })` — raw MongoDB driver, not Mongoose |
| DTO | `apps/web/types/financialCalculationDTO.ts` | `FinancialCalculationsHistory` = `{ _id, summary, resultBreakDownText? }`. `FinancialCalculationSummary` = `{ calculationOwner, calculationDate, totalResponsibility, patientDrugResponsibility, patientAdminResponsibility }` — three `{title,value}` pairs only |
| Mongoose model | `apps/web/models/Patient.ts` | Patient model does **not** declare `financialCalculationsHistory` in the schema — the field is added at runtime by migration and pushed via raw driver. There is no Mongoose validation for it today. |
| Migration | `apps/web/db-update/migration/utils/patients/up-patient-fc.mjs` | Backfilled `financialCalculationsHistory: []` on every patient. Down migration removes empty arrays only. |
| Read path | `apps/web/components/Modules/financial-calculator/financialCalculator.tsx:21` | Reads `patient?.financialCalculationsHistory` from `usePatientProvider()` — already in memory when the patient page loads. No dedicated read endpoint. |

### Legacy schema vs. Stage 1 schema — fidelity gap

The legacy `FinancialCalculationsHistory` stores **only** the three summary numbers and a markdown blob. The Stage 1 in-memory `FinancialCalculation` type stores the full `CalculationInputs` (all 9 user-entered fields) **and** `CalculationResult.summaryLines` (a structured array). This matters:

- The legacy schema loses the original inputs after save — you cannot re-display "what they entered" without parsing the markdown blob.
- The Stage 1 UI **already renders** an expandable inputs grid (`CalculationBreakdown.tsx`) that depends on having the raw `inputs` object preserved.

Reusing the legacy schema verbatim in Option A or Option B would silently regress the Stage 1 UI to "summary only" — the inputs grid would have nothing to show. **Stage 2 must persist the full `CalculationInputs` + `CalculationResult.summaryLines`** regardless of which option is chosen, OR we accept that the inputs grid only shows on the in-session calculation and disappears on refresh. The cleaner choice is to extend the schema, not to truncate the UI.

### Order collections — current state

`auditLog.migration.ts:3` confirms the canonical trio: `['neworders', 'maintenanceorders', 'leadorders']`. **None of the three has a Mongoose schema** in `apps/web/models/` — they are read/written by raw driver. `Order.ts` (Mongoose) is for a different surface and uses `mapToLisaOrders` to link back to one of these collections by hex string. This affects Option A's complexity: there is no Mongoose validation to extend, so we would either write raw-driver code (which the team has been moving away from — new code is expected to use Mongoose) or introduce Mongoose models from scratch (a much bigger undertaking, since `neworders`/`maintenanceorders` are accessed by raw driver across dozens of existing files).

### Architectural options

Two viable shapes are on the table:

#### Option A — Embedded array on order docs (`financialCalculations` field on each order collection)

Add a `financialCalculations: FinancialCalculation[]` field to documents in `neworders` (and possibly `maintenanceorders` / `leadorders` once PO confirms). Read by including the field in the existing order-detail fetch; write by `$push` to the same order doc.

**Pros**
- Mirrors the legacy patient pattern — PO/engineering team is already familiar with the shape.
- Co-located with the order — single query to load order + its calculations; no `$lookup` needed.
- Cheap to read on the order-detail page (no extra round trip).
- Natural "history is part of the order" mental model.

**Cons**
- Three collections × no Mongoose models = code change spread across raw-driver writes for each collection PO confirms (`neworders` only? all three?).
- Embedded arrays grow unboundedly. Today usage is low; in 12 months an active maintenance order could have 50+ recalculations. MongoDB document size cap (16 MB) is not a near-term risk for this payload, but updates rewrite the entire array which is wasteful.
- Cannot easily query "all calculations by user X" or "all calculations in date range" without scanning all order docs.
- Migration to a real collection later is painful (have to copy nested arrays out).
- Cache invalidation: `orderstrackercache` only re-caches on order doc change. Pushing into the order **does** trigger re-cache for that order, which is desirable — but it also forces a re-cache of unrelated order fields just because we appended a calculation, which is wasteful work.
- Indexing: no useful per-calculation index possible.

#### Option B — Dedicated `financialCalculations` collection with `orderId` ref

Create a new top-level collection (Mongoose model in `apps/web/models/FinancialCalculation.ts`) with documents like:
```
{
  _id, orderId: ObjectId, orderType: 'new' | 'maintenance' | 'lead',
  inputs: CalculationInputs, result: CalculationResult,
  author: string, calculationDate: Date,
  createdAt, updatedAt
}
```
Read by `find({ orderId, orderType })`; write by `insertOne`. Index on `{ orderId: 1, calculationDate: -1 }`.

**Pros**
- Uses Mongoose (the team's preferred ODM for new code) — schema validation lives in the model file.
- Independent of `orderstrackercache` — inserting a calculation does NOT trigger an order cache rebuild for unrelated fields.
- Supports future queries cleanly: "all calculations by author", "by date range", "for an order **and** all orders for that patient", export jobs.
- Indexed read path; no array rewrite on append.
- Future-proof against PO eventually wanting cross-order analytics ("how often does the Drug Cost field change between recalculations?").
- One model + one service + one set of routes — same shape across all three order collections (just a discriminator field).
- Pulling/restoring data is a single-collection operation, not three.

**Cons**
- One additional `find()` per order-detail page load (acceptable — it's an indexed lookup by `orderId`, returns a small bounded set).
- New collection means a new entry in any audit/PHI sweep tooling. Need to confirm `services/audit` coverage.
- Discriminator `orderType` is mildly ugly. Alternatives: separate collections per order type (rejected — same logic three times), or single field assumed-`neworders`-only until proven otherwise.

#### Recommendation (pending PO confirmation)

**Option B**. The collection is small, the queries are cleanly indexed, the Mongoose-first preference is satisfied, and we keep the door open for cross-order or cross-author analytics that the PO has not asked for yet but will plausibly want once the feature is in use for a quarter. Option A's only real advantage is "one less query on the detail page" — which is not the bottleneck on that page (the page already fans out into clinical reviews, documents, etc.).

If PO needs the cheapest possible Stage 2 ship (e.g., for a near-term demo), Option A on **`neworders` only** is acceptable as a stopgap with a clearly-noted migration follow-up to Option B before adding maintenance/lead.

### Open questions to confirm with PO before locking the option

1. **Which order collections?** `neworders` only, or also `maintenanceorders` and `leadorders`? Stage 1 is wired to render under `/orders-tracker/[category]/[orderId]/financial-review` for any category — the data layer should match.
2. **Author/role**: who saves calculations? Any user with order-detail access, or finance-role-only? Affects authorization checks on the POST route.
3. **Editability**: is a saved calculation immutable, or can it be edited / soft-deleted? Drives schema (`deletedAt`, `editedAt`) and UI (no edit button today).
4. **AI-generated "Reason" markdown**: the legacy flow appends an Azure OpenAI explanation to the result. Does the new order-scoped UI also want this, or do we drop it (it's not part of Stage 1's render)?
5. **Audit / PHI sweep**: does `services/audit` automatically cover new collections, or do we register the new collection somewhere?
6. **Migration of legacy data**: are the items in `patients.financialCalculationsHistory` to be backfilled into the new order-scoped store, or is this a fresh start? Backfill is hard either way because the legacy items have no `orderId` — they were never linked to an order. Likely answer: fresh start, leave legacy data in place, decommission legacy UI separately.

### Workstreams (once Option B is confirmed)

1. **Understand** — final PO sign-off on the 6 open questions above and on Option B.
2. **Analyze** — confirm whether the legacy patient-scoped flow stays or sunsets after the new flow ships (deprecation of `/patient/[id]/financial-calculator` is a separate ticket).
3. **Design**
   - `apps/web/models/FinancialCalculation.ts` Mongoose schema with `CalculationInputs`, `CalculationResult`, `orderId`, `orderType`, author, dates.
   - REST contract: `GET /api/orders/[orderType]/[orderId]/financial-calculations`, `POST /api/orders/[orderType]/[orderId]/financial-calculations`. (Route shape tentative — pin down once Option B is locked.)
   - Service layer in `apps/web/services/mongodb/financialCalculations.ts` (Mongoose, NOT raw driver).
   - Index strategy: `{ orderId: 1, calculationDate: -1 }`, plus optional `{ author: 1 }` if author-filtering is in scope.
   - Authorization rules (default to any-user-with-order-detail-access, tighten if PO says finance-role only).
4. **Build**
   - Add Mongoose model + service layer with TDD.
   - Add API routes with TDD.
   - Add a `useFinancialCalculations` client hook that hits the API instead of in-memory state.
   - Replace `FinancialCalculationsProvider` body or remove it in favor of the hook; keep the same call sites unchanged.
   - Delete `seed.ts` and the orderId-remap shim in `getCalculationsForOrder`.
   - Update the Calculator's `addCalculation` flow to call `POST` and revalidate the list.
   - Leave the `ORDER_FINANCIAL_CALCULATION` flag default `false` until QA sign-off.

### Out of Stage 2 (defer further)
- Editing/deleting saved calculations (if PO confirms immutable).
- Printable / shareable calculation summary export.
- Calculation versioning or audit trail beyond what `services/audit` already provides.
- Auto-fill from upstream data sources (deductible/OOP from `patient_insurances`, drug cost from drug catalog, etc.) — separate ticket.
- The standalone `[calculationId]` detail page (still out of scope per playground UX).
- Sunset / migration of the legacy `/patient/[id]/financial-calculator` UI.

### Indicators that Stage 2 is ready to ship
- All Stage 1 components untouched except `FinancialCalculationsContext` swap.
- The orderId-remap shim is gone.
- The seed file is deleted.
- Data layer scopes by orderId (not patientId) — confirmed by integration tests.
- The PR description explicitly calls out that mock data has been removed.

---

## Stage 2 — Data Model Proposal for PE / TL Review

> Self-contained section — share this with the Principal Engineer and Tech Lead. Everything they need to weigh in is below; the rest of this doc is for my own context.

### Context (one paragraph)

We are adding a new **Financial Review** tab to the Order Details v2 dashboard. It lets a user run a patient-responsibility calculation against a specific order and save a history of those calculations, scoped to that order. There is an existing patient-scoped equivalent at `/patient/[id]/financial-calculator` that writes a subdocument array to `patients.financialCalculationsHistory[]` (raw MongoDB driver, no Mongoose). PO has confirmed we want the new flow scoped to the **order**, not the patient, and limited to `neworders` and `maintenanceorders` (not `leadorders`). Stage 1 (UI with in-memory mock data) is committed locally on `feature/MLID-2307-financial-review`. Stage 2 is the data layer; we need to lock the shape before building.

### What gets persisted (same for either option)

This is the per-calculation payload that needs to be stored. Same 9 inputs + result shape as the Stage 1 UI already renders:

```ts
type FinancialCalculation = {
  _id: ObjectId;
  // 9 user-entered fields
  inputs: {
    deductibleMet: number;
    deductibleMax: number;
    coinsurance: number;     // 0-100
    oopMet: number;
    oopMax: number;
    drugCost: number;
    adminCost: number;
    drugAssist: number;
    adminAssist: number;
  };
  // Computed by a pure function in apps/web/.../financial-review/_components/calculate.ts
  result: {
    drugResp: number;
    adminResp: number;
    totalResp: number;
    summaryLines: string[];  // human-readable explanation lines, built from inputs
  };
  author: string;            // user email from NextAuth session
  calculationDate: Date;
  createdAt: Date;
  updatedAt: Date;
};
```

Note: this is a **richer schema than the legacy `patients.financialCalculationsHistory`**, which only stored summary numbers + a markdown blob. The Stage 1 UI depends on `inputs` and `summaryLines` being present to render its inputs grid and breakdown view. Whichever option is chosen, we need to persist this full shape.

### Current state of the order collections — important fact

`neworders`, `maintenanceorders`, and `leadorders` are all accessed via **raw MongoDB driver** (`db.collection(COLLECTION.NewOrders)`). **None of them has a Mongoose model.** The `apps/web/models/Order.ts` Mongoose model is for the unrelated `orders` collection (WeInfuse-mirrored), not these. This affects Option A more than Option B (see below).

---

### Option A — Embedded array on each order document

Add a `financialCalculations: FinancialCalculation[]` field to documents in `neworders` and `maintenanceorders`. Read it by including the field on the existing order-detail fetch. Write it by `$push` to the parent order doc.

**Proposed structure**

```ts
// On every document in neworders and maintenanceorders:
{
  _id: ObjectId,
  // ... existing order fields ...
  financialCalculations: [
    {
      _id: ObjectId,
      inputs: { deductibleMet, deductibleMax, coinsurance, oopMet, oopMax,
                drugCost, adminCost, drugAssist, adminAssist },
      result: { drugResp, adminResp, totalResp, summaryLines: string[] },
      author: 'user@mylocalinfusion.com',
      calculationDate: ISODate(...),
      createdAt: ISODate(...),
      updatedAt: ISODate(...),
    },
    // ...
  ],
}
```

**Reads** ride along with the existing order fetch — no extra query.

**Writes** use the raw driver (consistent with how `neworders` and `maintenanceorders` are touched everywhere else):

```ts
// apps/web/services/mongodb/financialCalculations.ts
export async function pushFinancialCalculation(
  orderType: 'new' | 'maintenance',
  orderId: ObjectId,
  calc: Omit<FinancialCalculation, '_id'>
) {
  const collectionName = orderType === 'new'
    ? COLLECTION.NewOrders
    : COLLECTION.MaintenanceOrders;
  const { db } = await connectToDatabase();
  return db.collection(collectionName).updateOne(
    { _id: orderId },
    { $push: { financialCalculations: { _id: new ObjectId(), ...calc } } }
  );
}
```

**Migration** to backfill empty arrays (`db-update/migration/utils/orders/up-orders-financial-calculations.mjs`), mirroring the existing `up-patient-fc.mjs` pattern.

**Pros**
- Single query to load order + its calculations (no `$lookup`).
- Mirrors the legacy patient-scoped pattern — familiar shape.
- Append is one `$push`; no separate transactional concern.
- Cache: `orderstrackercache` already re-caches on order doc change, so saved calculations show up automatically next time the cache is touched.

**Cons**
- **No Mongoose schema for these collections today.** "Adding the field to the schema" really means either (a) keep using raw driver — no validation, matches the legacy patient pattern but conflicts with the team's stated direction of using Mongoose for all new database code — or (b) introduce brand-new Mongoose models for `neworders` + `maintenanceorders` (huge scope — dozens of files would need to be touched to migrate from raw driver to Mongoose). Realistically, this option is raw-driver-only for Stage 2.
- Embedded arrays grow unboundedly. Each save rewrites the array. Not a near-term storage problem, but wasteful on updates.
- Forces a re-cache of the entire order doc in `orderstrackercache` every time a calculation is saved (even though the change is unrelated to the cached columns).
- Hard to query across orders (e.g. "all calculations by user X this week", "average totalResp by drug") — would need `$unwind`.
- Migration to a separate collection later is expensive (have to copy nested arrays out and rewrite call sites).
- Schema duplication across `neworders` and `maintenanceorders` — both collections grow a new field independently.

---

### Option B — Dedicated collection

Create a new top-level collection for financial calculations, with a reference to the source order. Two sub-shapes are viable; we need to pick one.

#### Option B.1 — Single collection with `orderType` discriminator

One collection: `financialCalculations`. Each document carries an `orderType` field that says which order collection the `orderId` points at.

```ts
// apps/web/models/FinancialCalculation.ts (NEW)
import mongoose, { Schema, Document } from 'mongoose';
import { COLLECTION } from '@/utils/constants';

export type OrderType = 'new' | 'maintenance';

export interface FinancialCalculationDocument extends Document {
  orderId: mongoose.Types.ObjectId;     // _id in neworders OR maintenanceorders
  orderType: OrderType;                 // discriminator: which collection orderId points at
  inputs: {
    deductibleMet: number;
    deductibleMax: number;
    coinsurance: number;
    oopMet: number;
    oopMax: number;
    drugCost: number;
    adminCost: number;
    drugAssist: number;
    adminAssist: number;
  };
  result: {
    drugResp: number;
    adminResp: number;
    totalResp: number;
    summaryLines: string[];
  };
  author: string;
  calculationDate: Date;
  createdAt: Date;
  updatedAt: Date;
}

const FinancialCalculationSchema = new Schema<FinancialCalculationDocument>(
  {
    orderId: { type: Schema.Types.ObjectId, required: true, index: true },
    orderType: {
      type: String,
      enum: ['new', 'maintenance'],
      required: true,
      index: true,
    },
    inputs: {
      deductibleMet: { type: Number, required: true, min: 0 },
      deductibleMax: { type: Number, required: true, min: 0 },
      coinsurance:   { type: Number, required: true, min: 0, max: 100 },
      oopMet:        { type: Number, required: true, min: 0 },
      oopMax:        { type: Number, required: true, min: 0 },
      drugCost:      { type: Number, required: true, min: 0 },
      adminCost:     { type: Number, required: true, min: 0 },
      drugAssist:    { type: Number, required: true, min: 0 },
      adminAssist:   { type: Number, required: true, min: 0 },
    },
    result: {
      drugResp:    { type: Number, required: true, min: 0 },
      adminResp:   { type: Number, required: true, min: 0 },
      totalResp:   { type: Number, required: true, min: 0 },
      summaryLines:{ type: [String], default: [] },
    },
    author:          { type: String, required: true },
    calculationDate: { type: Date,   required: true, default: () => new Date() },
  },
  { timestamps: true, collection: 'financialCalculations' }
);

// Compound index: list a single order's calculations newest-first.
FinancialCalculationSchema.index({ orderId: 1, orderType: 1, calculationDate: -1 });

export const FinancialCalculation =
  (mongoose.models.financialCalculations as mongoose.Model<FinancialCalculationDocument>) ||
  mongoose.model<FinancialCalculationDocument>(
    'financialCalculations',
    FinancialCalculationSchema
  );
```

**Read** (one indexed query):
```ts
FinancialCalculation.find({ orderId, orderType }).sort({ calculationDate: -1 }).lean();
```

**Write** (one `create`):
```ts
FinancialCalculation.create({ orderId, orderType, inputs, result, author });
```

**Pros**
- Mongoose-first (aligns with the team's direction of using Mongoose for all new database code); validation lives in one file.
- One model, one service, one set of routes — no duplication.
- Indexed lookup; appends do not rewrite an array.
- Doesn't touch `orderstrackercache` invalidation.
- Easy to extend (`orderType: 'lead'` is a one-line enum change if PO ever asks).
- Cross-order analytics are trivial: `find({ author }).sort({ calculationDate: -1 })`.

**Cons**
- The `orderType` discriminator is mildly ugly — every read needs to pass both `orderId` and `orderType`, and they must match. Easy to misuse if `orderType` is dropped at the API layer.
- `orderId` alone is not unique across the two source collections — small but real footgun if anyone ever forgets to filter by `orderType` (in practice, ObjectIds are 96-bit unique, so a stray match is astronomically unlikely; still, the type signature is the safety net).
- One extra collection in audit/PHI sweeps.

#### Option B.2 — Two separate collections (mirrors source collections)

Two collections: `newOrdersFinancialCalculations` and `maintenanceOrdersFinancialCalculations`. Same document shape as B.1 but **without** the `orderType` field. The collection name is the discriminator.

```ts
// apps/web/models/NewOrderFinancialCalculation.ts and
// apps/web/models/MaintenanceOrderFinancialCalculation.ts
// (Same schema body; only the model name and collection name differ.)
```

Service layer:
```ts
function modelFor(orderType: 'new' | 'maintenance') {
  return orderType === 'new'
    ? NewOrderFinancialCalculation
    : MaintenanceOrderFinancialCalculation;
}
```

**Pros**
- No discriminator field — `orderId` is the natural key within each collection.
- Mirrors the source-collection symmetry (`neworders` ↔ `newOrdersFinancialCalculations`).
- If `neworders` and `maintenanceorders` ever diverge in their financial flow (e.g. maintenance gets extra fields like "treatment number"), the schemas can diverge cleanly.

**Cons**
- Two models, two collections, two indexes to maintain — same code two places.
- Adding `leadorders` later means a third collection + a third model (vs. one enum value in B.1).
- Audit / PHI sweep coverage now spans two collections instead of one.
- Slightly more boilerplate in the service layer to dispatch by `orderType`.

---

### Side-by-side comparison

| Concern | Option A — embedded array | Option B.1 — one collection w/ discriminator | Option B.2 — two collections |
|---|---|---|---|
| Mongoose schema | **None** (would need to introduce, huge scope) | One new model | Two new models |
| Aligns with team's "new code = Mongoose" direction | ❌ (raw driver) | ✅ | ✅ |
| Read path | Free (rides on order fetch) | One indexed `find` | One indexed `find` |
| Write path | `$push` on order | `create` on collection | `create` on collection |
| Document growth | Order doc grows unboundedly | Fixed-size new doc per calc | Fixed-size new doc per calc |
| `orderstrackercache` impact | Every save re-caches the order | Untouched | Untouched |
| Cross-order analytics | Hard (`$unwind`) | Easy | Easy (but per-collection) |
| Future "lead orders" support | Add field to a third collection | One enum value | A third collection + model |
| Risk of mis-scoping (`orderId` collision across order types) | None (data is nested) | Real — `orderType` must always be passed | None (collection is the namespace) |
| Migration cost from this to another shape later | High | Low | Low |
| New collections in audit / PHI tooling | 0 | 1 | 2 |
| Stage 2 implementation effort (rough) | Small (raw-driver `$push` + read change + migration) | Small-medium (model + service + routes + migration) | Medium (×2 of B.1) |

### Recommendation (mine, for the conversation)

**Option B.1** — single dedicated collection with an `orderType` discriminator. It's Mongoose-first, indexed, append-cheap, easy to extend if PO adds `leadorders` later, and avoids growing the already-large `neworders` documents. The discriminator footgun is real but easy to mitigate with a service-layer wrapper that always takes both `orderId` and `orderType` together.

**Option A** is tempting because reads are free, but it permanently couples the calculation data to the order's storage shape — and we lose Mongoose validation on a brand new piece of business logic that depends on numeric inputs being well-formed.

**Option B.2** is the right pick *only* if PE/TL think the two order types will diverge in financial behavior soon.

### Questions for PE / TL

1. **Option A vs B** — does the team have a strong existing direction on "ride along on the order doc" vs "dedicated collection" for order-scoped derived data? Any precedent we should follow?
2. **If B, is B.1 (discriminator) or B.2 (two collections) preferred?** Specifically: do you anticipate `neworders` and `maintenanceorders` diverging on financial-calculation needs?
3. **Mongoose vs raw driver for the new write path.** `neworders`/`maintenanceorders` are raw-driver everywhere. Is a Mongoose-first new collection (Option B) the preferred go-forward stance, or should I match the local convention?
4. **Authorization** — any role gate on saving calculations (finance-only?), or is any user with order-detail access fine?
5. **Immutability** — saved calculations stay immutable, or do we need edit/soft-delete? (Drives whether the schema needs `editedAt` / `deletedAt`.)
6. **Audit / PHI sweep registration** — does a new collection need to be registered anywhere, or does `services/audit` pick it up automatically?

---

## Definition of Done — Stage 1

- [x] `ORDER_FINANCIAL_CALCULATION` enum entry and default present in `featureFlags.ts`
- [x] `calculate.ts` ported with all four seed scenarios producing exact expected outputs
- [x] OOP-cap scaling and assistance-overflow edge case tests passing
- [x] `FinancialCalculationsContext` tests passing; provider confirms `addCalculation` prepends to list
- [x] `CalculationBreakdown` renders compact-mode only; split bar hidden when either leg is 0
- [x] `CalculationCard` expand/collapse, Latest chip, and copy action all tested
- [x] `FinancialReviewIndex` empty state and list rendering tested; Latest chip passed to first item only
- [x] `Calculator` save flow, live estimate, ESC handler, and author fallback all tested
- [x] `OrderClinicalReviewsPanel` test: tab hidden when flag off, visible when flag on
- [x] All new tests passing: `npm run test` (55/55 across 7 suites)
- [x] TypeScript strict mode passing: `npm run types:check` (from `apps/web/`)
- [x] Manual visual sign-off from user
- [x] No `any` types, no `console.log`, no `Co-Authored-By` in commits
- [x] **Stage 1 committed locally on `feature/MLID-2307-financial-review`** (branch held; not pushed; no PR opened)

## Definition of Done — Stage 2 (gates the PR)

- [ ] Data layer designed with PO/PM input documented
- [ ] Mongoose model + service layer + API routes implemented with TDD
- [ ] `seed.ts` and orderId-remap shim removed
- [ ] Real `useFinancialCalculations` hook backed by API; no in-memory drift between list and form
- [ ] PR draft written to `docs/agomez/PR/MLID-2307-financial-review-calculator.md`
- [ ] PR explicitly notes mock data has been removed and feature flag still defaults to `false` for production rollout
