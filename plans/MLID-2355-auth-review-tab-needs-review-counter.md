# MLID-2355 — Auth Review Tab Needs-Review Counter (Badge)

## Task Reference

- **Jira**: [MLID-2355](https://localinfusion.atlassian.net/browse/MLID-2355)
- **Story Points**: 3
- **Branch**: `feature/MLID-2355-auth-review-needs-review-counter`
- **Base Branch**: `develop`
- **Status**: Done — merged to `develop` (PR created and merged by the author)

> **Delivery summary.** Feature branch `feature/MLID-2355-auth-review-needs-review-counter` was pushed, PR opened against `develop`, and merged. Three commits landed:
> - `e45b9b792` — feat(auth-review): add Needs Review counter badge to Prior Auth tab
> - `65abd04b2` — fix(prior-auth): count needs_review as not-reviewed in list filter (shared `buildPriorAuthListPipeline`: `needs_review` now `{ $ne: 'reviewed' }`)
> - `df7e588dc` — test(prior-auth): align patient route needs_review assertions with `$ne` filter
>
> Quality gate passed (tests, types, prettier, eslint clean; coverage: `query.ts` 100%, hook 100%, panel 89%). Feature gated by `ORDER_DETAILS_AUTH_REVIEW` (no new flag). Jira not transitioned — team moves the ticket to QA after code lands on `develop`.

> **Confirm base branch before branching.** Per team convention, never branch without verifying the base. Epic MLID-2206 is fully merged to `develop` and there is no `epic/MLID-2206` branch. Confirm `develop` is the correct base before creating the feature branch.

---

## Summary

Add a green numeric badge to the "Prior Auth" sub-tab inside `OrderDetailsReviewsPanel`. The badge shows the count of prior-auth documents for the open order that have `needs_review` status. The count comes from a new focused hook that calls the existing `GET /api/orders/new/${orderId}/auth-review` endpoint with `?reviewStatus=needs_review&page=1&pageSize=1` and reads the `total` field. No new API route, no SignalR, no per-user read state, no new collections, no new feature flags, and no env vars are introduced.

---

## Codebase Analysis

**Panel that owns the tab:**
`apps/web/app/orders-tracker/[category]/[orderId]/components/OrderDetailsReviewsPanel.tsx` — `export default function OrderDetailsReviewsPanel()`. Reads `orderId` via `useParams<{ category: string; orderId: string }>()`. Builds `reviewTabItems` in a `useMemo`; the Prior Auth entry is `{ label: 'Prior Auth', value: 'auth' }`, present only when `isAuthReviewEnabled`. The `Tabs` component (from `@repo/ui`) accepts a `badge?: number` on each tab item.

**Documents badge pattern (exact model to follow):**
`apps/web/app/orders-tracker/[category]/[orderId]/layout.tsx` — calls `useDocumentsTabNotifications(orderId)`, destructures `{ unreadCount: documentsUnreadCount }`, and spreads the badge conditionally:

```
...(isNotificationsEnabled && documentsUnreadCount > 0 ? { badge: documentsUnreadCount } : {})
```

The badge for this task follows the same hide-when-zero pattern, but has no separate notification feature flag — the tab itself is already behind `ORDER_DETAILS_AUTH_REVIEW`.

**Existing data hook (template for the new hook and its tests):**
`apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/hooks/useOrderAuthReview.ts` — uses native `fetch` + `AbortController`, no SWR. Returns `{ ..., total, ... }`. The list URL is built from `URLSearchParams`; the `reviewStatus` param key is confirmed by `FILTER_PARAM_MAP`.

**Test template:**
`apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/hooks/useOrderAuthReview.test.tsx` — mocks `global.fetch = jest.fn()` directly (no module mock), and mocks `@/utils/logger` with `jest.mock('@/utils/logger', ...)`. The new hook's test file must follow this exact pattern.

**Hooks barrel:**
`apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/hooks/index.ts` — currently exports `useOrderAuthReview` and its types. The new hook and its return type must be added here.

**Tab mock (for panel tests):**
`apps/web/mocks/repo-ui.tsx` — the `Tabs` mock renders `<span data-testid={`badge-${tab.label}`}>{tab.badge}</span>` only when `tab.badge` is truthy. So the testid for the Prior Auth badge will be `badge-Prior Auth`.

**Type for review status:**
`apps/web/types/priorAuth.ts` — `PriorAuthReviewStatus = 'needs_review' | 'reviewed'`.

**API endpoint (no change):**
`apps/web/app/api/orders/new/[orderId]/auth-review/route.ts` returns `{ letters, total, totalPages }`. With `?reviewStatus=needs_review&page=1&pageSize=1`, `total` is the count of all `needs_review` documents for the order (server-computed via `$facet`), regardless of the single-item page.

---

## Implementation Steps

### Step 1 — RED: Write failing tests for `useAuthReviewNeedsReviewCount`

- **Files**:
  - Create `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/hooks/useAuthReviewNeedsReviewCount.test.tsx`

- **What**: Write all test cases for the new hook before the hook exists. Tests will fail because the import does not resolve. Following the exact pattern of `useOrderAuthReview.test.tsx`: mock `global.fetch = jest.fn()` directly, mock `@/utils/logger`, use `renderHook` + `waitFor` from `@testing-library/react`. Import the hook from `./useAuthReviewNeedsReviewCount`.

  Test cases to cover:

  1. **returns count from `total` on successful fetch** — mock fetch returning `{ total: 3 }`, assert `result.current.count === 3` after `waitFor`.
  2. **returns count 0 when `total` is 0** — mock fetch returning `{ total: 0 }`, assert `result.current.count === 0`.
  3. **sets `isLoading` true during fetch, false after** — verify `isLoading` starts true and settles to false on resolution.
  4. **returns count 0 and does not throw on fetch error** — mock fetch rejecting with `new Error('Network error')`, assert `result.current.count === 0` and `result.current.isLoading === false`.
  5. **returns count 0 and does not call fetch when orderId is empty string** — render with `orderId: ''`, assert `global.fetch` was never called and `result.current.count === 0`.
  6. **fetches the correct URL with `reviewStatus=needs_review&page=1&pageSize=1`** — assert `global.fetch` was called with a URL that `includes('/api/orders/new/ord1/auth-review')` and `includes('reviewStatus=needs_review')` and `includes('page=1')` and `includes('pageSize=1')`.

Run tests: `cd apps/web && npx jest useAuthReviewNeedsReviewCount` — expected: all fail with "Cannot find module".

---

### Step 2 — GREEN: Implement `useAuthReviewNeedsReviewCount`

- **Files**:
  - Create `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/hooks/useAuthReviewNeedsReviewCount.ts`

- **What**: Implement the minimal hook to make Step 1 tests pass.

  Exported type:
  ```typescript
  export type UseAuthReviewNeedsReviewCountReturn = {
    count: number;
    isLoading: boolean;
  };
  ```

  Exported function signature:
  ```typescript
  export function useAuthReviewNeedsReviewCount(
    orderId: string
  ): UseAuthReviewNeedsReviewCountReturn
  ```

  Logic:
  - `'use client'` directive at top.
  - Imports: `PriorAuthReviewStatus` from `@/types/priorAuth`, `logger` from `@/utils/logger`, `useEffect` + `useState` from `react`.
  - State: `count` (number, 0), `isLoading` (boolean, false).
  - `useEffect` with `[orderId]` dependency:
    - If `!orderId`, return early (no fetch, count stays 0).
    - Create `AbortController`.
    - Set `isLoading` true.
    - Build URL: `/api/orders/new/${orderId}/auth-review` with `URLSearchParams` setting `reviewStatus` to `'needs_review' satisfies PriorAuthReviewStatus`, `page` to `'1'`, `pageSize` to `'1'`.
    - Fetch with `signal: controller.signal`.
    - On success: parse JSON, set `count` to `data.total ?? 0`.
    - On error: ignore `AbortError`; for all other errors call `logger.error('Failed to fetch auth review needs-review count', { orderId })` and leave count at 0.
    - In `finally`: set `isLoading` false.
    - Return cleanup `() => controller.abort()`.
  - Return `{ count, isLoading }`.

Run tests: `cd apps/web && npx jest useAuthReviewNeedsReviewCount` — expected: all pass.

---

### Step 3 — GREEN: Export from barrel

- **Files**:
  - Modify `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/hooks/index.ts`

- **What**: Add exports for the new hook and its return type so consumers can import from the barrel. Append to the existing export list:

  ```typescript
  export {
    useAuthReviewNeedsReviewCount,
    type UseAuthReviewNeedsReviewCountReturn,
  } from './useAuthReviewNeedsReviewCount';
  ```

  No tests needed for a re-export barrel. Verify TypeScript compiles: `cd apps/web && npx tsc --noEmit`.

---

### Step 4 — RED: Write failing tests for the badge in `OrderDetailsReviewsPanel`

- **Files**:
  - Modify `apps/web/app/orders-tracker/[category]/[orderId]/components/OrderDetailsReviewsPanel.test.tsx`

- **What**: Add a new `describe` block for the Prior Auth badge. The panel test already mocks `@repo/ui` via `require('@/mocks/repo-ui')`, which renders `<span data-testid="badge-Prior Auth">` when `badge` is truthy on the Prior Auth tab item. The new mock for the hook follows the same module-mock pattern used for `useClinicalReviews`.

  Add at the top of the file (alongside the other mock declarations):
  ```typescript
  jest.mock(
    '../auth-review/_components/hooks/useAuthReviewNeedsReviewCount',
    () => ({
      useAuthReviewNeedsReviewCount: jest.fn(),
    })
  );
  ```
  And a typed cast:
  ```typescript
  import { useAuthReviewNeedsReviewCount } from '../auth-review/_components/hooks/useAuthReviewNeedsReviewCount';
  const useAuthReviewNeedsReviewCountMock = useAuthReviewNeedsReviewCount as jest.Mock;
  ```

  In `beforeEach` of every existing `describe` block, add a default return:
  ```typescript
  useAuthReviewNeedsReviewCountMock.mockReturnValue({ count: 0, isLoading: false });
  ```

  Add a new `describe` block:

  ```
  describe('OrderDetailsReviewsPanel — Prior Auth badge', () => {
    beforeEach(() => {
      jest.clearAllMocks();
      // same base setup as other describe blocks (useParams, useRouter, useOrderDetails, useClinicalReviews)
      useAuthReviewNeedsReviewCountMock.mockReturnValue({ count: 0, isLoading: false });
    });

    it('shows badge with count on the Prior Auth tab when flag is on and count > 0', ...);
      // useFeatureFlag: ORDER_DETAILS_AUTH_REVIEW = true
      // useAuthReviewNeedsReviewCount returns { count: 2, isLoading: false }
      // expect screen.getByTestId('badge-Prior Auth') to have text content '2'

    it('does not show badge on the Prior Auth tab when count is 0', ...);
      // useAuthReviewNeedsReviewCount returns { count: 0, isLoading: false }
      // expect screen.queryByTestId('badge-Prior Auth') to not be in the document

    it('does not show Prior Auth tab or badge when ORDER_DETAILS_AUTH_REVIEW flag is off', ...);
      // useFeatureFlag returns false for ORDER_DETAILS_AUTH_REVIEW
      // expect screen.queryByText('Prior Auth') not in the document
      // expect screen.queryByTestId('badge-Prior Auth') not in the document

    it('calls useAuthReviewNeedsReviewCount with the orderId from params', ...);
      // render with orderId '319309' in params
      // expect useAuthReviewNeedsReviewCountMock to have been called with '319309'
  });
  ```

  Run tests: `cd apps/web && npx jest OrderDetailsReviewsPanel` — the new test cases fail because the panel does not yet call the hook.

---

### Step 5 — GREEN: Wire badge into `OrderDetailsReviewsPanel`

- **Files**:
  - Modify `apps/web/app/orders-tracker/[category]/[orderId]/components/OrderDetailsReviewsPanel.tsx`

- **What**: Three changes to the component.

  1. Import the new hook:
     ```typescript
     import { useAuthReviewNeedsReviewCount } from '../auth-review/_components/hooks';
     ```

  2. Call the hook, passing `orderId` from params (which is already extracted as `params?.orderId`):
     ```typescript
     const { count: authReviewNeedsReviewCount } = useAuthReviewNeedsReviewCount(
       params?.orderId ?? ''
     );
     ```
     Place this call near the other hook calls at the top of the function body, after the `useParams` call.

  3. Spread the badge on the Prior Auth tab item inside `reviewTabItems`:
     ```typescript
     ...(isAuthReviewEnabled
       ? [
           {
             label: 'Prior Auth',
             value: 'auth',
             ...(authReviewNeedsReviewCount > 0
               ? { badge: authReviewNeedsReviewCount }
               : {}),
           },
         ]
       : []),
     ```

  Also add `authReviewNeedsReviewCount` to the `useMemo` dependency array for `reviewTabItems`.

  Run tests: `cd apps/web && npx jest OrderDetailsReviewsPanel` — all pass.

---

### Step 6 — REFACTOR: Review, lint, and type-check

- **Files**: No new files; no logic changes. Verify the two implementation files and two test files compile and lint cleanly.

- **What**:
  - Run `cd apps/web && npx tsc --noEmit` — must pass with zero errors.
  - Run `cd apps/web && npx eslint app/orders-tracker/\\[category\\]/\\[orderId\\]/auth-review/_components/hooks/useAuthReviewNeedsReviewCount.ts app/orders-tracker/\\[category\\]/\\[orderId\\]/components/OrderDetailsReviewsPanel.tsx` — must pass.
  - Run the full test suite for the changed files:
    ```
    cd apps/web && npx jest useAuthReviewNeedsReviewCount OrderDetailsReviewsPanel
    ```
  - Confirm no `any` type is present anywhere in the new or modified files.
  - Confirm `'use client'` directive is present in `useAuthReviewNeedsReviewCount.ts` (it uses `useEffect` / `useState`).
  - Confirm `useAuthReviewNeedsReviewCount` does not appear in the `useMemo` dependency array as a stable identity concern — it is a named hook import, not a closure value, so no issue.

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/hooks/useAuthReviewNeedsReviewCount.ts` | Create | New focused hook that fetches `needs_review` count for an order |
| `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/hooks/useAuthReviewNeedsReviewCount.test.tsx` | Create | Tests for the new hook (TDD RED-first) |
| `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/hooks/index.ts` | Modify | Re-export the new hook and its return type |
| `apps/web/app/orders-tracker/[category]/[orderId]/components/OrderDetailsReviewsPanel.tsx` | Modify | Call new hook, spread badge on the Prior Auth tab item |
| `apps/web/app/orders-tracker/[category]/[orderId]/components/OrderDetailsReviewsPanel.test.tsx` | Modify | Add mock for new hook, add badge-specific describe block |

---

## Testing Strategy

### Unit tests — `useAuthReviewNeedsReviewCount`

All test cases are in `useAuthReviewNeedsReviewCount.test.tsx`. The mock boundary is `global.fetch`. Each case uses `renderHook` + `waitFor`:

1. Derives `count` from the server-returned `total` field.
2. Returns `count: 0` when `total` is 0.
3. `isLoading` is true during the fetch and false after settlement.
4. Returns `count: 0` on fetch error (does not propagate or throw); calls `logger.error`.
5. Does not call `fetch` when `orderId` is an empty string; `count` stays 0.
6. Builds the correct URL: contains the orderId path segment, `reviewStatus=needs_review`, `page=1`, `pageSize=1`.

### Unit tests — `OrderDetailsReviewsPanel` badge behavior

All test cases in the new `describe` block in `OrderDetailsReviewsPanel.test.tsx`:

1. Badge `data-testid="badge-Prior Auth"` is present and shows the count when `ORDER_DETAILS_AUTH_REVIEW` flag is on and hook returns `count > 0`.
2. Badge is absent when hook returns `count: 0`.
3. Neither tab label "Prior Auth" nor `badge-Prior Auth` testid is present when flag is off.
4. The hook is called with the orderId extracted from `useParams`.

### Existing tests

Existing `OrderDetailsReviewsPanel` tests must continue to pass. The `beforeEach` in each existing `describe` block must include `useAuthReviewNeedsReviewCountMock.mockReturnValue({ count: 0, isLoading: false })` so existing cases are not affected by the new hook call.

### Manual verification

After the dev server is running (user starts it; no Playwright for this epic):

1. Navigate to any order's dashboard page (e.g. `/orders-tracker/new/<orderId>`) with `ORDER_DETAILS_AUTH_REVIEW` enabled.
2. If the order has prior-auth documents with `needs_review` status, confirm a numeric badge appears on the "Prior Auth" tab.
3. Mark one document as reviewed; navigate back to the order dashboard. Confirm the badge count decreased by 1 (or disappears if it was the last one).
4. Navigate to an order with zero `needs_review` documents. Confirm no badge is shown on the tab.
5. Disable `ORDER_DETAILS_AUTH_REVIEW` in the admin feature flags panel. Confirm the Prior Auth tab is entirely absent (no tab, no badge).

---

## Security Considerations

- **Input validation**: `orderId` comes from `useParams` (Next.js route segment). The endpoint already validates the order scoping server-side. The new hook passes it as a path segment to the existing API route — no additional sanitization needed in the hook.
- **Authorization**: The `GET /api/orders/new/${orderId}/auth-review` route already requires authentication and enforces the existing session/role checks. The hook is a read-only consumer of that route.
- **Feature flag gating**: The badge is implicitly gated at both the server (the existing route is gated) and the UI (the Prior Auth tab only renders when `ORDER_DETAILS_AUTH_REVIEW` is on, so the hook's output is only ever wired to a visible element when the flag is on). No additional feature flag is needed.
- **PHI handling**: The hook fetches only a count (`total`). No patient-identifying data is returned, stored, or logged. `logger.error` on failure logs only `orderId` — not PHI.

---

## Out of Scope

The following are explicitly excluded from this task:

- SignalR or any live-update mechanism — the badge updates on navigation (full page refetch).
- Per-user mark-as-read state — the counter is order-level status only.
- Notification engine changes — no new notification records.
- Changes to the prior-auth detail save flow — saving a review already redirects back to the list, which triggers a page-level refetch.
- Changes to `apps/web/app/orders-tracker/[category]/[orderId]/layout.tsx` — the badge lives inside `OrderDetailsReviewsPanel`, not the layout-level top navigation.
- New env vars — none introduced.
- New feature flags — none introduced.

---

## Acceptance Criteria Verification

| AC | How it is satisfied |
|----|---------------------|
| AC 1: Counter matches the number of auth documents in "Needs Review" status for the open order | `useAuthReviewNeedsReviewCount` fetches with `reviewStatus=needs_review` and reads the server-returned `total`, which is the full count scoped to the order (not the single-item page). |
| AC 2: After an auth doc is "Reviewed", the counter decreases by 1 | Reviewing a document changes its status server-side. The detail page redirects back to the order dashboard, which re-mounts `OrderDetailsReviewsPanel` and re-runs the hook, fetching the updated count. No live update machinery is required. |

---

## Open Questions

None. All architecture decisions are locked as stated in the task brief.
