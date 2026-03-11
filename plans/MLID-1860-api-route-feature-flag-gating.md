# MLID-1860 — API Route Feature Flag Gating

## Task Reference

- **Jira**: [MLID-1860](https://localinfusion.atlassian.net/browse/MLID-1860)
- **Story Points**: 2
- **Branch**: `feature/MLID-1860-api-route-feature-flag-gating`
- **Base Branch**: `epic/MLID-1492-order-eligibility`
- **Status**: Done

---

## Summary

Gate the `GET /api/orders/new` endpoint with the `ORDER_ELIGIBILITY` feature flag. When the flag is enabled, pass `includeEligibility: true` to `getOrders()` so the aggregation pipeline includes 340B and pharmacy eligibility stages. When disabled, pass `false` (current behavior).

---

## Codebase Analysis

Key findings from investigation:

- **API route**: `apps/web/app/api/orders/new/route.ts` — GET handler (lines 31–58) calls `getOrders()` with `category: 'new'` and pagination params. Currently does NOT pass `includeEligibility`.
- **`getOrders` signature**: Already accepts `includeEligibility?: boolean` in `GetOrdersParams` (line 917 in `ordersTracker.ts`). The pipeline conditionally includes eligibility stages when `category === 'new' && includeEligibility` (line 853).
- **FeatureFlagService pattern**: Used throughout the codebase via `FeatureFlagService.getFeatureFlag(FeatureFlag.FLAG_NAME)`. Import: `@/services/featureFlags/featureFlagService`.
- **FeatureFlag enum**: `ORDER_ELIGIBILITY` already exists in `utils/featureFlags/featureFlags.ts` (added in D1-T1).
- **Existing tests**: `route.test.ts` mocks `getOrders`, `getServerSession`, and other deps. The GET tests verify 401 for unauthenticated requests and 200 with mock data. No existing feature flag mocking.
- **Test mocking pattern**: `jest.mock('@/services/featureFlags/featureFlagService')` + cast `FeatureFlagService.getFeatureFlag as jest.MockedFunction<...>`.

---

## Implementation Steps

### Step 1 — Write failing tests (RED)

- **Files**: `apps/web/app/api/orders/new/route.test.ts`
- **What**: Add mock for `FeatureFlagService` and two new test cases:
  1. `'should pass includeEligibility: true when ORDER_ELIGIBILITY flag is enabled'` — mock `getFeatureFlag` returning `true`, call GET, assert `getOrders` was called with `includeEligibility: true`
  2. `'should pass includeEligibility: false when ORDER_ELIGIBILITY flag is disabled'` — mock `getFeatureFlag` returning `false`, call GET, assert `getOrders` was called with `includeEligibility: false`
- **New mocks needed**:
  - `jest.mock('@/services/featureFlags/featureFlagService')`
  - `const mockGetFeatureFlag = FeatureFlagService.getFeatureFlag as jest.MockedFunction<typeof FeatureFlagService.getFeatureFlag>`
- **Default mock behavior**: In `beforeEach`, set `mockGetFeatureFlag.mockResolvedValue(false)` so existing tests aren't affected (flag off = no eligibility = current behavior)

### Step 2 — Implement feature flag check (GREEN)

- **Files**: `apps/web/app/api/orders/new/route.ts`
- **What**:
  1. Add imports: `FeatureFlagService` and `FeatureFlag`
  2. In the GET handler, after auth check and before `getOrders()` call:
     ```typescript
     const includeEligibility = await FeatureFlagService.getFeatureFlag(
       FeatureFlag.ORDER_ELIGIBILITY
     );
     ```
  3. Pass `includeEligibility` to the `getOrders()` call

### Step 3 — Refactor & verify (REFACTOR)

- **Files**: Both route.ts and route.test.ts
- **What**: Clean up, ensure no `any` types, run quality checks
- **Commands**: `npm run test`, `npm run types:check`, `npx prettier --write` and `npx eslint --fix` on changed files only

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/app/api/orders/new/route.ts` | Modify | Add feature flag check, pass `includeEligibility` to `getOrders()` |
| `apps/web/app/api/orders/new/route.test.ts` | Modify | Add FeatureFlagService mock, add 2 test cases for flag on/off |

---

## Testing Strategy

- **Unit tests**: 2 new tests verifying `getOrders` receives correct `includeEligibility` value based on feature flag state
- **Existing tests**: Default mock to `false` ensures all existing GET tests continue to pass (they don't assert on `includeEligibility`)
- **Manual verification**: Toggle `ORDER_ELIGIBILITY` flag in admin panel, verify network tab shows eligibility fields in response when enabled

---

## Security Considerations

- **Authorization**: No change — existing auth check remains first
- **Feature flag**: Server-side gating ensures eligibility computation only runs when explicitly enabled
- **PHI handling**: No new PHI exposure — eligibility fields are derived/computed, not patient data

---

## Open Questions

None — all patterns are established and the `includeEligibility` param is already wired in the service layer.
