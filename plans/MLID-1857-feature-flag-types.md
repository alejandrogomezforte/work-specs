# MLID-1857 — Feature flag + types

## Task Reference

- **Jira**: [MLID-1857](https://localinfusion.atlassian.net/browse/MLID-1857)
- **Story Points**: 1
- **Branch**: `feature/MLID-1857-feature-flag-types`
- **Base Branch**: `epic/MLID-1492-order-eligibility`
- **Status**: To Do
- **Epic Plan**: [MLID-1492-plan-progress.md](./MLID-1492-plan-progress.md)

---

## Summary

Add `ORDER_ELIGIBILITY` feature flag and eligibility type fields to `FullOrder<K>`. This is the foundation task (D1-T1) that all other tasks in the epic depend on.

---

## Codebase Analysis

- **Feature flag file** (`apps/web/utils/featureFlags/featureFlags.ts`):
  - `FeatureFlag` enum at lines 18–67; new entry goes after `HOSPITAL_SYSTEMS` (line 59) or in a new logical group
  - `DEFAULT_FEATURE_FLAGS` at lines 73–127; new entry with `false` default
  - Existing test file at `apps/web/utils/featureFlags/featureFlags.test.tsx` — follows pattern of testing `getDefaultFeatureFlagValue` per flag

- **Orders types** (`apps/web/types/orders.ts`):
  - `FullOrder<K>` uses `WithRelationsAnd<OrderDTO<K>, Relations, Extra>` (lines 109–128)
  - The `Extra` block (3rd type param, lines 118–127) already uses `K extends 'lead' ? ... : never` pattern for conditional fields
  - No existing test file for types — type correctness verified via `types:check`

---

## Implementation Steps

### Step 1 — RED: Write failing test for ORDER_ELIGIBILITY default

- **File**: `apps/web/utils/featureFlags/featureFlags.test.tsx`
- **What**: Add test asserting `getDefaultFeatureFlagValue(FeatureFlag.ORDER_ELIGIBILITY)` returns `false`. This will fail because the enum member doesn't exist yet.

### Step 2 — GREEN: Add feature flag enum + default

- **File**: `apps/web/utils/featureFlags/featureFlags.ts`
- **What**:
  - Add `ORDER_ELIGIBILITY = 'ORDER_ELIGIBILITY'` to `FeatureFlag` enum
  - Add `[FeatureFlag.ORDER_ELIGIBILITY]: false` to `DEFAULT_FEATURE_FLAGS`

### Step 3 — Add eligibility type fields

- **File**: `apps/web/types/orders.ts`
- **What**: Add two optional fields to the `Extra` type parameter in `FullOrder<K>`:
  ```typescript
  hospital340BEligible?: K extends 'new' ? string : never;
  pharmacyEligible?: K extends 'new' ? string : never;
  ```
  This ensures the fields are only available on `FullOrder<'new'>` and typed as `never` for maintenance/lead.

### Step 4 — Verify quality

- Run tests: `npm run test -- --testPathPattern="featureFlags"`
- Run type check: `npm run types:check`
- Run lint on changed files only: `npx eslint --fix <changed files>`
- Run format: `npm run format`

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/utils/featureFlags/featureFlags.ts` | Modify | Add `ORDER_ELIGIBILITY` to enum + defaults |
| `apps/web/utils/featureFlags/featureFlags.test.tsx` | Modify | Add test for `ORDER_ELIGIBILITY` default value |
| `apps/web/types/orders.ts` | Modify | Add `hospital340BEligible` and `pharmacyEligible` to `FullOrder<K>` Extra |

---

## Testing Strategy

- **Unit test**: `getDefaultFeatureFlagValue(FeatureFlag.ORDER_ELIGIBILITY)` returns `false`
- **Type verification**: `npm run types:check` confirms:
  - `FullOrder<'new'>` has both optional string fields
  - `FullOrder<'maintenance'>` and `FullOrder<'lead'>` have them as `never`
  - Existing code compiles without modification

---

## Security Considerations

- **No security impact** — this task only adds a feature flag (defaulting to `false`) and type definitions. No runtime behavior changes.

---

## Open Questions

- None — scope is fully defined in the epic plan.
