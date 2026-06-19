# MLID-2605 — Decouple Financial Calculator and Prior Auth from CLINICAL_REVIEWS Flag

## Task Reference

- **Jira**: [MLID-2605](https://localinfusion.atlassian.net/browse/MLID-2605)
- **Story Points**: 4
- **Branch**: `fix/MLID-2605-decouple-review-tab-flags`
- **Base Branch**: `develop`
- **Status**: Done — browser-verified; committed as `b02ce8da0` on `fix/MLID-2605-decouple-review-tab-flags` (pending push + PR)
- **Parent**: MLID-2190 (Production Bugs tracking bucket — no epic plan file)

---

## Summary

When `CLINICAL_REVIEWS` is OFF but `ORDER_FINANCIAL_CALCULATION` and/or `ORDER_DETAILS_AUTH_REVIEW` are ON, the Financial Review and Prior Auth sub-tabs do not appear in `/orders-tracker/[category]/[orderId]`. The root cause is that `layout.tsx` mounts the shared review-tabs panel only when `isClinicalReviewsEnabled` is true, and the panel itself always includes "Clinical Review" as an unconditional tab entry. This fix decouples the three sub-tabs so each is gated exclusively by its own flag.

This task also **renames the misleadingly-named component** that hosts all three tabs. The component is currently `OrderClinicalReviewsPanel`, but it is a generic wrapper for the Clinical Review, Prior Auth, and Financial Review tabs — not a clinical-only panel. (Its misleading name is arguably what made this bug easy to introduce: a "ClinicalReviews" panel naturally got gated by the clinical flag.) Step 1 renames it to **`OrderDetailsReviewsPanel`** first, as a pure refactor commit, so the subsequent logic changes land on a correctly-named component. Throughout this plan, the new name `OrderDetailsReviewsPanel` and the new test-id `order-details-reviews-panel` refer to the post-rename state.

---

## Codebase Analysis

**`apps/web/app/orders-tracker/[category]/[orderId]/layout.tsx`**

The file declares all feature flags at lines 173–198 using `useFeatureFlag`. The four variables relevant to this fix are:

- `isOrderDetailsActionsEnabled` (line 177) — master switch for the whole review panel area
- `isClinicalReviewsEnabled` (line 180)
- `isFinancialReviewEnabled` (line 181)
- `isAuthReviewEnabled` (line 184)

There are four gating expressions in the render output (lines 651–674):

1. **Clinical routed children** (lines 651–657): `isClinicalReviewsPath && isOrderDetailsActionsEnabled && isClinicalReviewsEnabled` — unchanged by this fix.
2. **Financial routed children** (lines 658–664): `isFinancialReviewPath && isOrderDetailsActionsEnabled && isFinancialReviewEnabled` — unchanged by this fix.
3. **Auth routed children** (lines 665–667): `isAuthReviewPath && isAuthReviewEnabled` — missing the master switch; this fix adds `isOrderDetailsActionsEnabled`.
4. **Dashboard panel mount** (lines 668–674): `isDashboardPath && isOrderDetailsActionsEnabled && isClinicalReviewsEnabled` — this fix expands the condition to mount the panel when ANY of the three sub-flags is on.

**`apps/web/app/orders-tracker/[category]/[orderId]/components/OrderDetailsReviewsPanel.tsx`**

- Lines 21–26: reads `isFinancialReviewEnabled` and `isAuthReviewEnabled` via `useFeatureFlag`. Does NOT currently read `CLINICAL_REVIEWS`.
- Lines 40–49 (`reviewTabItems` useMemo): "Clinical Review" tab is unconditional; Prior Auth and Financial are already conditional on their flags. Dependency array: `[isAuthReviewEnabled, isFinancialReviewEnabled]`.
- Lines 51–59 (`activeReviewTab` useMemo): falls back to `'clinical'` regardless of whether `CLINICAL_REVIEWS` is on. Dependency array: `[isAuthReviewEnabled, isFinancialReviewEnabled, pathname]`.
- Lines 61–73: `useClinicalReviews(data?.patientId ?? undefined, ...)` — always fetches clinical reviews data even when `CLINICAL_REVIEWS` is off.

**Existing test mocking patterns:**

- `layout.test.tsx`: `useFeatureFlag` is mocked as a module-level function with mutable `let` boolean variables, switched by flag string. `ORDER_FINANCIAL_CALCULATION` is NOT yet in the switch statement and falls through to `return true` (line 123). `mockOrderDetailsActionsFlag` and `mockClinicalReviewsFlag` are reset to `false` in `beforeEach`. `screen.getByTestId('order-details-reviews-panel')` asserts panel mount.
- `OrderDetailsReviewsPanel.test.tsx`: `useFeatureFlag` is mocked as `jest.fn()` via `useFeatureFlagMock`. Tests call `.mockReturnValue(boolean)` (all flags same value) or `.mockImplementation((flag) => flag === FeatureFlag.X)` (per-flag). `usePathname` is `jest.fn()` via `usePathnameMock`.

**Financial calculator audit:** No coupling between the financial calculator UI or backend routes and `CLINICAL_REVIEWS` / `useClinicalReviews` exists outside the panel mount. This is verified during implementation (see Step 2).

---

## Acceptance Criteria — Gating Matrix

| `ORDER_DETAILS_ACTIONS` | `CLINICAL_REVIEWS` | `ORDER_FINANCIAL_CALCULATION` | `ORDER_DETAILS_AUTH_REVIEW` | Dashboard panel mounts? | Clinical tab visible? | Financial tab visible? | Prior Auth tab visible? |
|---|---|---|---|---|---|---|---|
| OFF | any | any | any | No | — | — | — |
| ON | OFF | OFF | OFF | No | — | — | — |
| ON | ON | OFF | OFF | Yes | Yes | No | No |
| ON | OFF | ON | OFF | Yes | No | Yes | No |
| ON | OFF | OFF | ON | Yes | No | No | Yes |
| ON | ON | ON | ON | Yes | Yes | Yes | Yes |
| ON | OFF | ON | ON | Yes | No | Yes | Yes |

Auth-review route children gate:

| `ORDER_DETAILS_ACTIONS` | `ORDER_DETAILS_AUTH_REVIEW` | Children rendered at `/auth-review`? |
|---|---|---|
| OFF | ON | No (new behavior — master switch now required) |
| ON | ON | Yes |
| ON | OFF | No |

---

## Implementation Steps

### Step 1 — REFACTOR: rename `OrderClinicalReviewsPanel` → `OrderDetailsReviewsPanel`

- **Files**: rename `OrderClinicalReviewsPanel.tsx`, `OrderClinicalReviewsPanel.test.tsx`, `OrderClinicalReviewsPanel.module.css` (all under `apps/web/app/orders-tracker/[category]/[orderId]/components/`) to `OrderDetailsReviewsPanel.*`. Update all importers.
- **What**: A pure rename, no behavior change. Do this first so every later edit lands on the correctly-named component. The existing test suite (with the renamed test-id) must stay GREEN — this commit changes names only.

Rename actions:

1. **Rename the 3 files** (use `git mv` to preserve history):
   - `OrderClinicalReviewsPanel.tsx` → `OrderDetailsReviewsPanel.tsx`
   - `OrderClinicalReviewsPanel.test.tsx` → `OrderDetailsReviewsPanel.test.tsx`
   - `OrderClinicalReviewsPanel.module.css` → `OrderDetailsReviewsPanel.module.css`
2. **In `OrderDetailsReviewsPanel.tsx`**: rename the function/default export `OrderClinicalReviewsPanel` → `OrderDetailsReviewsPanel`; update the CSS import to `'./OrderDetailsReviewsPanel.module.css'`.
3. **Update the 4 importers** (default-import name + JSX usage):
   - `apps/web/app/orders-tracker/[category]/[orderId]/layout.tsx`
   - `apps/web/app/orders-tracker/[category]/[orderId]/clinical-reviews/page.tsx`
   - `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/page.tsx`
   - `apps/web/app/orders-tracker/[category]/[orderId]/financial-review/page.tsx`
4. **Update `layout.test.tsx`** (the file does NOT move — only references change). The `order-clinical-reviews-panel` test-id is defined by a MOCK here, not by the real component (the real component has no `data-testid`). Four edits:
   - **Line 89 — `jest.mock('./components/OrderClinicalReviewsPanel', …)` module path** → `'./components/OrderDetailsReviewsPanel'`. **CRITICAL**: if this path is not updated, Jest silently stops intercepting the module (layout.tsx now imports the renamed file), the REAL panel renders instead of the mock, and the layout tests break in a confusing way (not an obvious "module not found"). This was the easy-to-miss item.
   - Line 91 — mock function name `MockOrderClinicalReviewsPanel` → `MockOrderDetailsReviewsPanel` (cosmetic, for readability).
   - Line 92 — `data-testid="order-clinical-reviews-panel"` → `"order-details-reviews-panel"`.
   - Line 550 — the assertion `getByTestId('order-clinical-reviews-panel')` → `'order-details-reviews-panel'` (1 occurrence today).
5. **Update the renamed `OrderDetailsReviewsPanel.test.tsx`** (16 references; relative mock paths like `../clinical-reviews/…` and `../OrderDetailsContext` stay unchanged since the file does not change directory):
   - Line 7 — import path `'./OrderClinicalReviewsPanel'` → `'./OrderDetailsReviewsPanel'` and the imported binding name.
   - Line 67 — `describe('OrderClinicalReviewsPanel — tab interaction')` label.
   - Lines 97–251 — 14 × `render(<OrderClinicalReviewsPanel />)` → `<OrderDetailsReviewsPanel />`.

A whole-word replace of `OrderClinicalReviewsPanel` → `OrderDetailsReviewsPanel` inside the renamed test file covers the import binding, the 14 render calls, and the describe label in one pass; then fix the import path string and (in `layout.test.tsx`) the `jest.mock` path string and test-id string separately.

Verification: `npm run test` for the two affected test files must pass unchanged (names only — no assertion logic changes), and `npx tsc --noEmit` must be clean. No RED/GREEN here — this is a refactor commit that keeps tests green. Run the suite at the end of this step specifically to confirm the `jest.mock` path change took effect (the panel still renders as a mocked `<div data-testid="order-details-reviews-panel" />`, not the real component).

---

### Step 2 — RED: Financial calculator audit grep + write failing layout tests

- **Files**: `apps/web/app/orders-tracker/[category]/[orderId]/layout.test.tsx`
- **What**: Run the audit grep first (no source changes), then add four failing test cases to the existing `describe('OrderDetailsLayout — auth-review gate')` block and one new `describe` block. Tests must fail because the current source still requires `isClinicalReviewsEnabled` for the dashboard panel mount.

**Audit command to run during implementation (expected: zero matches):**

```
grep -rn "CLINICAL_REVIEWS\|useClinicalReviews\|isClinicalReviewsEnabled" \
  apps/web/app/orders-tracker/\[category\]/\[orderId\]/financial-review \
  apps/web/components/Modules/financial-calculator \
  apps/web/app/api/orders
```

Expected result: no output. If any matches are found, remove the coupling before proceeding.

**Add `mockFinancialReviewFlag` to the layout test mock** — currently `ORDER_FINANCIAL_CALCULATION` falls through to `return true`; make it explicit so tests can control it. In the module-level `let` declarations block (after line 110), add:

```typescript
let mockFinancialReviewFlag = false;
```

In the `useFeatureFlag` mock switch (after the `ORDER_DETAILS_AUTH_REVIEW` case, before `return true`):

```typescript
if (flag === 'ORDER_FINANCIAL_CALCULATION') return mockFinancialReviewFlag;
```

In `beforeEach` of both `describe` blocks, add:

```typescript
mockFinancialReviewFlag = false;
```

**Adjust the existing `describe('OrderDetailsLayout — auth-review gate')` block (lines 1413–1459) to the new master switch.**

This describe block currently encodes the OLD contract — that `ORDER_DETAILS_AUTH_REVIEW` alone is sufficient to render auth-review children. Under the new master-switch model, the auth route also requires `ORDER_DETAILS_ACTIONS`. Three changes:

1. **Modify the existing test at line 1429** — currently titled `'renders children on the auth-review path when ORDER_DETAILS_AUTH_REVIEW is on'`, it sets only `mockAuthReviewFlag = true` (leaving `mockOrderDetailsActionsFlag = false` from the `beforeEach`). This encodes the old contract. Rewrite it to the new contract:
   - Rename to `'renders children on the auth-review path when both ORDER_DETAILS_ACTIONS and ORDER_DETAILS_AUTH_REVIEW are on'`
   - Add `mockOrderDetailsActionsFlag = true;` alongside `mockAuthReviewFlag = true;`
   - Assertion unchanged (`getByTestId('auth-children')` IS in document)
   - This stays GREEN against current source (auth flag on already renders) and remains GREEN after Step 4 — it is the happy-path regression guard for the new contract. Making this edit in Step 2 means Step 4 does NOT have to retroactively "repair" a broken test mid-implementation.

2. **Add a new RED test** `'does not render children on the auth-review path when ORDER_DETAILS_ACTIONS is off even if ORDER_DETAILS_AUTH_REVIEW is on'`
   - Set `mockAuthReviewFlag = true`, leave `mockOrderDetailsActionsFlag = false`
   - Assert `screen.queryByTestId('auth-children')` is NOT in document (and the order heading still renders, mirroring the line 1443 test)
   - RED reason: current source at line 665 does not check `isOrderDetailsActionsEnabled`, so children render today → assertion fails → RED. Goes GREEN after Step 4 Change B.

3. **Leave the existing test at line 1443** (`'does not render children on the auth-review path when the flag is off'`, both flags off → not rendered) unchanged — it remains valid under the new contract.

**New `describe` block: `'OrderDetailsLayout — dashboard panel decoupling'`**

```typescript
describe('OrderDetailsLayout — dashboard panel decoupling', () => {
  beforeEach(() => {
    jest.clearAllMocks();
    mockPathname = '/orders-tracker/new/order-123';
    mockCategory = 'new';
    mockOrderHistoryFlag = true;
    mockDocumentsFlag = true;
    mockNotificationsFlag = true;
    mockRtsNoGoDateAutomationFlag = true;
    mockNoGoContextCaptureFlag = true;
    mockOrderDetailsActionsFlag = false;
    mockClinicalReviewsFlag = false;
    mockAuthReviewFlag = false;
    mockFinancialReviewFlag = false;
    mockFetch.mockResolvedValue(mockOrderResponse);
  });
  // ...
});
```

Test: `'mounts the dashboard panel when ORDER_DETAILS_ACTIONS and ORDER_FINANCIAL_CALCULATION are on but CLINICAL_REVIEWS is off'`
- Set `mockOrderDetailsActionsFlag = true`, `mockFinancialReviewFlag = true`, `mockClinicalReviewsFlag = false`
- Assert `screen.getByTestId('order-details-reviews-panel')` is in document
- RED reason: current source requires `isClinicalReviewsEnabled`

Test: `'mounts the dashboard panel when ORDER_DETAILS_ACTIONS and ORDER_DETAILS_AUTH_REVIEW are on but CLINICAL_REVIEWS is off'`
- Set `mockOrderDetailsActionsFlag = true`, `mockAuthReviewFlag = true`, `mockClinicalReviewsFlag = false`
- Assert `screen.getByTestId('order-details-reviews-panel')` is in document
- RED reason: same

Test: `'does not mount the dashboard panel when ORDER_DETAILS_ACTIONS is on but all sub-flags are off'`
- Set `mockOrderDetailsActionsFlag = true`, all three sub-flags false
- Assert `screen.queryByTestId('order-details-reviews-panel')` is NOT in document
- RED reason: current source passes this because `isClinicalReviewsEnabled` is false — but it must continue to pass after the fix (regression guard)

Test: `'does not mount the dashboard panel when ORDER_DETAILS_ACTIONS is off regardless of sub-flags'`
- Set `mockOrderDetailsActionsFlag = false`, all three sub-flags true
- Assert `screen.queryByTestId('order-details-reviews-panel')` is NOT in document
- RED reason: regression guard — must stay NOT in document

---

### Step 3 — RED: Write failing panel tests for CLINICAL_REVIEWS decoupling

- **Files**: `apps/web/app/orders-tracker/[category]/[orderId]/components/OrderDetailsReviewsPanel.test.tsx`
- **What**: Add a new `describe` block to the existing test file. These tests fail because `CLINICAL_REVIEWS` is not yet read in the panel and "Clinical Review" tab is unconditional.

**New `describe` block: `'OrderDetailsReviewsPanel — CLINICAL_REVIEWS flag decoupling'`**

```typescript
describe('OrderDetailsReviewsPanel — CLINICAL_REVIEWS flag decoupling', () => {
  beforeEach(() => {
    jest.clearAllMocks();
    pushMock = jest.fn();
    useParamsMock.mockReturnValue({ category: 'new', orderId: '319309' });
    useRouterMock.mockReturnValue({ push: pushMock });
    usePathnameMock.mockReturnValue(basePath);
    useOrderDetailsMock.mockReturnValue({
      data: { patientId: 'patient-1', id: 'order-1' },
      isLoading: false,
      error: null,
    });
    useClinicalReviewsMock.mockReturnValue({
      reviews: [],
      isLoading: false,
      error: null,
      page: 1,
      pageSize: 10,
      total: 0,
      totalPages: 1,
      setPage: jest.fn(),
      setPageSize: jest.fn(),
    });
  });
  // ...
});
```

Test: `'hides the Clinical Review tab when CLINICAL_REVIEWS flag is off'`
- `useFeatureFlagMock.mockImplementation((flag) => flag === FeatureFlag.ORDER_FINANCIAL_CALCULATION)`
  (CLINICAL_REVIEWS returns false, FINANCIAL returns true, AUTH returns false)
- Assert `screen.queryByText('Clinical Review')` is NOT in document
- Assert `screen.getByText('Financial Review')` is in document
- RED reason: "Clinical Review" tab is unconditional in current source

Test: `'shows the Clinical Review tab when CLINICAL_REVIEWS flag is on'`
- `useFeatureFlagMock.mockImplementation((flag) => flag === FeatureFlag.CLINICAL_REVIEWS)`
- Assert `screen.getByText('Clinical Review')` is in document
- Assert `screen.queryByText('Financial Review')` is NOT in document
- RED reason: passes today (tab is unconditional), but once step 3 lands it becomes a real guard

Test: `'shows only Prior Auth and Financial Review tabs when CLINICAL_REVIEWS is off and both other flags are on'`
- `useFeatureFlagMock.mockImplementation((flag) => flag === FeatureFlag.ORDER_DETAILS_AUTH_REVIEW || flag === FeatureFlag.ORDER_FINANCIAL_CALCULATION)`
- Assert `screen.queryByText('Clinical Review')` is NOT in document
- Assert `screen.getByText('Prior Auth')` is in document
- Assert `screen.getByText('Financial Review')` is in document
- RED reason: "Clinical Review" is unconditional today

Test: `'defaults activeReviewTab to Financial Review when CLINICAL_REVIEWS is off, ORDER_FINANCIAL_CALCULATION is on, and pathname is the base order path'`
- `useFeatureFlagMock.mockImplementation((flag) => flag === FeatureFlag.ORDER_FINANCIAL_CALCULATION)`
- `usePathnameMock.mockReturnValue(basePath)` (not a sub-path)
- Assert `screen.getByTestId('financial-review-index')` is in document
- Assert `screen.queryByText('Loading clinical reviews...')` is NOT in document (no clinical render)
- RED reason: `activeReviewTab` currently falls back to `'clinical'` regardless

Test: `'defaults activeReviewTab to Prior Auth when only ORDER_DETAILS_AUTH_REVIEW is on and pathname is the base order path'`
- `useFeatureFlagMock.mockImplementation((flag) => flag === FeatureFlag.ORDER_DETAILS_AUTH_REVIEW)`
- `usePathnameMock.mockReturnValue(basePath)`
- Assert `screen.getByTestId('auth-review-index')` is in document
- RED reason: same fallback issue

---

### Step 4 — GREEN: Fix `layout.tsx` — dashboard panel mount condition and auth master-switch

- **Files**: `apps/web/app/orders-tracker/[category]/[orderId]/layout.tsx`
- **What**: Two changes, both in the render return block.

**Change A — dashboard panel mount (lines 668–674):**

Current:
```typescript
{isDashboardPath &&
isOrderDetailsActionsEnabled &&
isClinicalReviewsEnabled &&
!isLoading &&
!error ? (
  <OrderDetailsReviewsPanel />
) : null}
```

Replace with:
```typescript
{isDashboardPath &&
isOrderDetailsActionsEnabled &&
(isClinicalReviewsEnabled ||
  isAuthReviewEnabled ||
  isFinancialReviewEnabled) &&
!isLoading &&
!error ? (
  <OrderDetailsReviewsPanel />
) : null}
```

**Change B — auth routed children gate (lines 665–667):**

Current:
```typescript
{isAuthReviewPath && isAuthReviewEnabled && !isLoading && !error
  ? children
  : null}
```

Replace with:
```typescript
{isAuthReviewPath &&
isOrderDetailsActionsEnabled &&
isAuthReviewEnabled &&
!isLoading &&
!error
  ? children
  : null}
```

After this change, the RED tests from Step 2 must now be GREEN — including the new auth-review master-switch test (`ORDER_DETAILS_ACTIONS` off + `ORDER_DETAILS_AUTH_REVIEW` on → children NOT rendered). The auth-review happy-path test (renamed to "both on" in Step 2) and the line 1443 "both off" test stay GREEN. No test repair is needed here: Step 2 already aligned the existing auth-review block to the new contract, so Change B does not break any previously-passing test.

---

### Step 5 — GREEN: Fix `OrderDetailsReviewsPanel.tsx` — add CLINICAL_REVIEWS flag and fix active tab fallback

- **Files**: `apps/web/app/orders-tracker/[category]/[orderId]/components/OrderDetailsReviewsPanel.tsx`
- **What**: Three sub-changes.

**Sub-change A — add `isClinicalReviewsEnabled` (after line 26):**

```typescript
const isClinicalReviewsEnabled = useFeatureFlag(FeatureFlag.CLINICAL_REVIEWS);
```

**Sub-change B — make Clinical Review tab conditional in `reviewTabItems` (lines 40–49):**

Current:
```typescript
const reviewTabItems = useMemo(
  () => [
    { label: 'Clinical Review', value: 'clinical' },
    ...(isAuthReviewEnabled ? [{ label: 'Prior Auth', value: 'auth' }] : []),
    ...(isFinancialReviewEnabled
      ? [{ label: 'Financial Review', value: 'financial' }]
      : []),
  ],
  [isAuthReviewEnabled, isFinancialReviewEnabled]
);
```

Replace with:
```typescript
const reviewTabItems = useMemo(
  () => [
    ...(isClinicalReviewsEnabled
      ? [{ label: 'Clinical Review', value: 'clinical' }]
      : []),
    ...(isAuthReviewEnabled ? [{ label: 'Prior Auth', value: 'auth' }] : []),
    ...(isFinancialReviewEnabled
      ? [{ label: 'Financial Review', value: 'financial' }]
      : []),
  ],
  [isClinicalReviewsEnabled, isAuthReviewEnabled, isFinancialReviewEnabled]
);
```

**Sub-change C — fix `activeReviewTab` fallback (lines 51–59):**

Current:
```typescript
const activeReviewTab = useMemo(() => {
  if (isAuthReviewEnabled && pathname?.includes('/auth-review')) {
    return 'auth';
  }
  if (isFinancialReviewEnabled && pathname?.includes('/financial-review')) {
    return 'financial';
  }
  return 'clinical';
}, [isAuthReviewEnabled, isFinancialReviewEnabled, pathname]);
```

Replace with:
```typescript
const activeReviewTab = useMemo(() => {
  if (isClinicalReviewsEnabled && pathname?.includes('/clinical-reviews')) {
    return 'clinical';
  }
  if (isAuthReviewEnabled && pathname?.includes('/auth-review')) {
    return 'auth';
  }
  if (isFinancialReviewEnabled && pathname?.includes('/financial-review')) {
    return 'financial';
  }
  // Fallback: return the first enabled tab in priority order
  if (isClinicalReviewsEnabled) return 'clinical';
  if (isAuthReviewEnabled) return 'auth';
  if (isFinancialReviewEnabled) return 'financial';
  return 'clinical';
}, [
  isClinicalReviewsEnabled,
  isAuthReviewEnabled,
  isFinancialReviewEnabled,
  pathname,
]);
```

After this change, all five new panel tests from Step 3 must be GREEN.

**Update existing panel tests that now break** — the tests at lines 93–111 (`'hides the Financial Review tab when ... flag is off'` and `'shows the Financial Review tab when ... flag is on'`) use `useFeatureFlagMock.mockReturnValue(false/true)` which sets ALL flags to the same value. When `CLINICAL_REVIEWS` is also controlled by that mock:

- `mockReturnValue(false)` — Clinical Review will now be hidden. The existing test at line 93 asserts `screen.getByText('Clinical Review')` is in document, which will FAIL. Update it: change to `useFeatureFlagMock.mockImplementation((flag) => flag === FeatureFlag.CLINICAL_REVIEWS)` to keep Clinical visible while Financial stays hidden.
- `mockReturnValue(true)` — Clinical Review is still visible. The existing test at line 103 passes without change.

Similarly review all tests in `describe('OrderDetailsReviewsPanel — tab interaction')` that use `mockReturnValue(false)` and assert `'Clinical Review'` is present — those will now fail unless `CLINICAL_REVIEWS` returns true. Audit each one:

- Line 93: set `mockImplementation` to return true only for `CLINICAL_REVIEWS` (Financial = false).
- Line 185 (`'hides the Calculate button when the feature flag is off...'`): uses `mockReturnValue(false)`. Clinical Review will be hidden. Test does not assert `'Clinical Review'` presence — but the panel renders with no active tab and may break in a different way. Update to `mockImplementation((flag) => flag === FeatureFlag.CLINICAL_REVIEWS)` (only Clinical on, Financial and Auth off).
- Line 207 (`'hides the Prior Auth tab when ORDER_DETAILS_AUTH_REVIEW flag is off'`): uses `mockReturnValue(false)`. Clinical Review would be hidden. Asserts `getByText('Clinical Review')` implicitly is needed — actually this test only checks `queryByText('Prior Auth')`. However rendering with no tabs visible might cause the `!data.patientId` guard or null returns. Update to `mockImplementation((flag) => flag === FeatureFlag.CLINICAL_REVIEWS)`.
- Line 247 (`'shows the empty-state start review action on the Clinical Review tab'`): uses `mockReturnValue(false)`, meaning Clinical Review tab would be hidden and `activeReviewTab` would fallback away from `'clinical'`, breaking the test. Update to `mockImplementation((flag) => flag === FeatureFlag.CLINICAL_REVIEWS)`.

---

### Step 6 — OPTIONAL REFACTOR: Guard `useClinicalReviews` fetch when `CLINICAL_REVIEWS` is off

- **Files**: `apps/web/app/orders-tracker/[category]/[orderId]/components/OrderDetailsReviewsPanel.tsx`
- **What**: Prevent an unnecessary clinical-reviews API call when the Clinical Review tab is not enabled.

**Hook behavior (verified — resolves the earlier open question):** `useClinicalReviews` (`clinical-reviews/_components/hooks/useClinicalReviews.ts:43-50`) only short-circuits (no fetch) when **both** `patientId` AND `options.lisaOrderId` are falsy:

```typescript
if (!patientId && !options.lisaOrderId) {
  setReviews([]); setTotal(0); setTotalPages(0); setError(null);
  return;
}
```

The panel always passes `lisaOrderId: data?.id`, so gating only `patientId` (as originally drafted) is NOT enough — the hook would still fetch by `lisaOrderId`. To truly skip the fetch, gate **both** arguments on `isClinicalReviewsEnabled`.

Current (lines 61–73):
```typescript
} = useClinicalReviews(data?.patientId ?? undefined, {
  lisaOrderId: data?.id,
});
```

Change to:
```typescript
} = useClinicalReviews(
  isClinicalReviewsEnabled ? (data?.patientId ?? undefined) : undefined,
  { lisaOrderId: isClinicalReviewsEnabled ? data?.id : undefined }
);
```

With both arguments undefined when the flag is off, the hook hits the early-return and performs no fetch. (The hook must still be called unconditionally — React rules of hooks forbid a conditional call.)

If applied: add a panel test asserting that when `CLINICAL_REVIEWS` is off, `useClinicalReviewsMock` is called with `undefined` as both the first argument and `lisaOrderId`. This step remains optional (a no-network-waste cleanup); it does not affect the user-visible bug fix and can be dropped if it complicates review.

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `.../components/OrderClinicalReviewsPanel.tsx` → `OrderDetailsReviewsPanel.tsx` | Rename | Rename component file + function/default export + CSS import (Step 1) |
| `.../components/OrderClinicalReviewsPanel.test.tsx` → `OrderDetailsReviewsPanel.test.tsx` | Rename | Rename test file + describe label + test-id (Step 1) |
| `.../components/OrderClinicalReviewsPanel.module.css` → `OrderDetailsReviewsPanel.module.css` | Rename | Rename CSS module (Step 1) |
| `.../clinical-reviews/page.tsx` | Modify | Update import + JSX to `OrderDetailsReviewsPanel` (Step 1) |
| `.../auth-review/page.tsx` | Modify | Update import + JSX to `OrderDetailsReviewsPanel` (Step 1) |
| `.../financial-review/page.tsx` | Modify | Update import + JSX to `OrderDetailsReviewsPanel` (Step 1) |
| `apps/web/app/orders-tracker/[category]/[orderId]/layout.tsx` | Modify | Update import/JSX to renamed component (Step 1); expand dashboard panel mount to OR across sub-flags; add master switch to auth-review gate (Step 4) |
| `apps/web/app/orders-tracker/[category]/[orderId]/layout.test.tsx` | Modify | Update test-id to `order-details-reviews-panel` (Step 1); add `mockFinancialReviewFlag`; add new failing tests for decoupled panel mount and auth master-switch (Step 2) |
| `.../components/OrderDetailsReviewsPanel.tsx` | Modify | Read `CLINICAL_REVIEWS` flag; make Clinical tab conditional; fix fallback in `activeReviewTab` (Step 5) |
| `.../components/OrderDetailsReviewsPanel.test.tsx` | Modify | Add new describe block for CLINICAL_REVIEWS decoupling; fix existing tests broken by the new flag read (Step 3) |

---

## Testing Strategy

**Unit tests — `layout.test.tsx`:**
- Dashboard panel mounts when `ORDER_DETAILS_ACTIONS` ON + `ORDER_FINANCIAL_CALCULATION` ON + `CLINICAL_REVIEWS` OFF
- Dashboard panel mounts when `ORDER_DETAILS_ACTIONS` ON + `ORDER_DETAILS_AUTH_REVIEW` ON + `CLINICAL_REVIEWS` OFF
- Dashboard panel does NOT mount when `ORDER_DETAILS_ACTIONS` ON + all three sub-flags OFF
- Dashboard panel does NOT mount when `ORDER_DETAILS_ACTIONS` OFF + all sub-flags ON
- Auth children NOT rendered when `ORDER_DETAILS_ACTIONS` OFF + `ORDER_DETAILS_AUTH_REVIEW` ON (new behavior)
- Auth children rendered when `ORDER_DETAILS_ACTIONS` ON + `ORDER_DETAILS_AUTH_REVIEW` ON (regression guard)

**Unit tests — `OrderDetailsReviewsPanel.test.tsx`:**
- Clinical Review tab hidden when `CLINICAL_REVIEWS` OFF; visible when ON
- Financial and Prior Auth tabs appear independently of `CLINICAL_REVIEWS`
- `activeReviewTab` defaults to first enabled tab (financial when only `ORDER_FINANCIAL_CALCULATION` is on)
- `activeReviewTab` defaults to Prior Auth when only `ORDER_DETAILS_AUTH_REVIEW` is on
- All three tabs visible when all three flags are on (regression)

**Manual verification:**

1. Start the dev server (`npm run dev:web`).
2. In the admin panel at `/admin/feature-flags`, set `ORDER_DETAILS_ACTIONS` ON, `CLINICAL_REVIEWS` OFF, `ORDER_FINANCIAL_CALCULATION` ON, `ORDER_DETAILS_AUTH_REVIEW` ON.
3. Navigate to any new order detail page (`/orders-tracker/new/[id]`).
4. Confirm the panel renders showing only "Prior Auth" and "Financial Review" tabs — no "Clinical Review" tab.
5. Click each tab and confirm the correct content renders.
6. Confirm navigating directly to `/orders-tracker/new/[id]/auth-review` renders auth content (master switch on).
7. Turn `ORDER_DETAILS_ACTIONS` OFF and confirm the panel disappears and `/auth-review` route shows nothing.
8. Turn `CLINICAL_REVIEWS` ON (with `ORDER_FINANCIAL_CALCULATION` and `ORDER_DETAILS_AUTH_REVIEW` also on) and confirm all three tabs appear.

**Mock boundaries identified:**
- `useFeatureFlag` — mocked at module level in both test files
- `useClinicalReviews` — mocked as `jest.fn()` in panel tests
- `useOrderDetails` — mocked as `jest.fn()` in panel tests
- `next/navigation` — mocked in both test files
- `global.fetch` — mocked in layout tests

---

## Security Considerations

- **Input validation**: Not applicable — this is a UI-only gating fix.
- **Authorization**: Feature flags are already enforced at both API routes (existing gating in `route.ts` files) and UI level. This fix affects only the UI panel mount logic; the API gating for each feature is unchanged.
- **PHI handling**: No PHI is introduced or logged by these changes. The panel renders patient-linked data only when the order already loaded; no new data sources are accessed.
- **Feature flag gating**: The master switch `ORDER_DETAILS_ACTIONS` continues to gate the entire panel at the layout level. Individual sub-tab flags gate their own tabs inside the panel. No change to backend flag enforcement.

---

## Pre-PR Checklist

Run all commands from `apps/web/`:

```bash
# Tests (run after each step to confirm RED then GREEN)
npx jest --testPathPattern="layout.test|OrderDetailsReviewsPanel.test" --no-coverage

# Full test suite (confirm no regressions)
npm run test

# Type check (run from apps/web/)
npx tsc --noEmit

# Lint
npm run lint:fix
```

Coverage must remain at or above 80% on both modified test files. The existing test suite already covers the modified lines; the new tests add coverage for the previously untested flag combinations.

**Commit message format** (one commit per step, no Co-Authored-By):

- Step 1: `[MLID-2605] - refactor(orders-tracker): rename OrderClinicalReviewsPanel to OrderDetailsReviewsPanel`
- Step 2: `[MLID-2605] - test(orders-tracker): add failing tests for decoupled panel mount in layout`
- Step 3: `[MLID-2605] - test(orders-tracker): add failing tests for CLINICAL_REVIEWS decoupling in panel`
- Step 4: `[MLID-2605] - fix(orders-tracker): expand dashboard panel mount and add master switch to auth gate`
- Step 5: `[MLID-2605] - fix(orders-tracker): decouple Clinical Review tab from CLINICAL_REVIEWS flag in panel`
- Step 6 (if applied): `[MLID-2605] - refactor(orders-tracker): guard useClinicalReviews fetch when CLINICAL_REVIEWS is off`

---

## Open Questions

All previously-open questions are now resolved (kept here as an audit trail):

- [x] **Auth-review under the master switch** — RESOLVED: confirmed with the user that `ORDER_DETAILS_ACTIONS` is the panel-level master switch and the auth-review route is pulled under it (a deliberate behavior change). The existing `auth-review gate` describe block in `layout.test.tsx` (lines 1413–1459) is adjusted to this contract in Step 2: the line 1429 test is rewritten to require both flags ("both on"), a new RED test asserts `ACTIONS` off + `AUTH` on → children NOT rendered, and the line 1443 "both off" test is unchanged. Step 4 implements the gate; no mid-implementation test repair is needed.
- [x] **Step 6 `useClinicalReviews` guard** — RESOLVED: verified the hook (`useClinicalReviews.ts:43-50`) only skips fetching when BOTH `patientId` and `lisaOrderId` are falsy. The panel always passes `lisaOrderId`, so Step 6 must gate BOTH arguments on `isClinicalReviewsEnabled` (not just `patientId`). Step 6 updated accordingly; it remains an optional cleanup.
