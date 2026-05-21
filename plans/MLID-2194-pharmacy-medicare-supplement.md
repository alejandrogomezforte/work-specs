# MLID-2194 — Pharmacy Eligibility: Add "Medicare Supplement" to Allow-List

## Task Reference

- **Jira**: [MLID-2194](https://localinfusion.atlassian.net/browse/MLID-2194)
- **Story Points**: 4
- **Sprint**: MLID Sprint 26
- **Branch**: `feature/MLID-2194-pharmacy-medicare-supplement`
- **Base Branch**: `develop` (PR target — branch was cut from develop, not the epic, since D1+D2 are already merged to develop)
- **Status**: Done — committed `05e7e0bb`, awaiting PR
- **Epic plan**: `docs/agomez/plans/MLID-1492-plan-progress.md`
- **PR doc**: `docs/agomez/PR/MLID-2194.md`

---

## Summary

Product Management identified a missing plan type: `payer_type_name = "Medicare Supplement"` should auto-pass the pharmacy eligibility check alongside the existing "Medicare" and "Medicare Advantage" values. The fix is a one-character regex change inside the existing `$switch` pipeline stage. No type changes, no schema changes, no feature flag wiring, no UI impact.

---

## Codebase Analysis

- **Pipeline stage:** `getEligibilityPipelineStages()` in `apps/web/services/mongodb/ordersTracker.ts`, line 25–224.
- **Target line:** line 174 — the `$regexMatch.regex` inside `branches[2].case.$and[2].$or[0]` of the `pharmacyEligible` `$switch`.
- **Current regex:** `'^\\s*medicare(\\s+advantage)?\\s*$'` — matches `Medicare` and `Medicare Advantage` only.
- **Required regex:** `'^\\s*medicare(\\s+(advantage|supplement))?\\s*$'` — additionally matches `Medicare Supplement`.
- **Existing test describe block:** `'pharmacyEligible Medicare regex anchoring'` at line 841 of `apps/web/services/mongodb/__tests__/ordersTracker.test.ts`. Contains 6 tests; one must be flipped, one must be added.
- **Feature flag:** `ORDER_ELIGIBILITY` already wired at API + UI level. No changes needed.

---

## Implementation Steps

### Step 1 (RED) — Flip the failing test and add the whitespace test

**File:** `apps/web/services/mongodb/__tests__/ordersTracker.test.ts`

**Edit 1 — flip lines 899–904:** rename and change expectation from `false` to `true`.

Before (lines 899–904):
```typescript
it('should NOT match "Medicare Supplement"', async () => {
  const regex = await getMedicareRegex();
  expect(new RegExp(regex, 'i').test('Medicare Supplement')).toBe(
    false
  );
});
```

After:
```typescript
it('should match "Medicare Supplement"', async () => {
  const regex = await getMedicareRegex();
  expect(new RegExp(regex, 'i').test('Medicare Supplement')).toBe(
    true
  );
});
```

**Edit 2 — add whitespace test after line 921** (inside the same `describe` block, before the closing `}`):
```typescript
it('should match "Medicare Supplement" with leading/trailing whitespace', async () => {
  const regex = await getMedicareRegex();
  expect(new RegExp(regex, 'i').test(' Medicare Supplement ')).toBe(
    true
  );
});
```

Run `npm run test -- ordersTracker` from `apps/web/`. Expected: 2 failures (the flipped test + the new test).

---

### Step 2 (GREEN) — Update the regex

**File:** `apps/web/services/mongodb/ordersTracker.ts`

**Edit line 174:**

Before:
```typescript
regex: '^\\s*medicare(\\s+advantage)?\\s*$',
```

After:
```typescript
regex: '^\\s*medicare(\\s+(advantage|supplement))?\\s*$',
```

Run `npm run test -- ordersTracker` from `apps/web/`. Expected: all tests green.

---

### Step 3 (REFACTOR) — Verify quality gates

No refactoring needed. Run from `apps/web/`:

```bash
npm run types:check
npm run lint:fix
```

Expected: clean.

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/services/mongodb/ordersTracker.ts` | Modify | Extend regex on line 174 to include `supplement` in the Medicare allow-list |
| `apps/web/services/mongodb/__tests__/ordersTracker.test.ts` | Modify | Flip line 899–904 test expectation; add one new whitespace test after line 921 |

---

## Testing Strategy

**Existing tests that must remain green (no changes):**
- `'should anchor Medicare regex with ^ and $'` — verifies the regex is still anchored.
- `'should match "Medicare" exactly'` — bare Medicare still passes.
- `'should match "Medicare Advantage"'` — existing allow-list entry still passes.
- `'should NOT match "Medicare Part D"'` — non-listed plan type still blocked.
- `'should match "Medicare" with leading/trailing whitespace'`
- `'should match "Medicare Advantage" with leading/trailing whitespace'`

**Tests being changed:**
- Line 899 test: flipped from `toBe(false)` → `toBe(true)`, renamed to `'should match "Medicare Supplement"'`.

**New test:**
- `'should match "Medicare Supplement" with leading/trailing whitespace'` — mirrors the existing whitespace tests for the other Medicare variants.

No integration test or manual browser verification needed; this is a pure backend regex change behind an already-deployed feature flag.

---

## Security Considerations

None. This is a read-only aggregation pipeline change behind the existing `ORDER_ELIGIBILITY` feature flag. No PHI is written or exposed. No new input surfaces.

---

## Open Questions

None.

---

## Optional Cleanup (Low Priority)

Two local working docs (gitignored) describe the old allow-list and will be stale after this change:
- `docs/agomez/analysis/pharmacy-eligibility.md` — lines ~108, ~131, ~236/240 reference the two-item Medicare list.
- `docs/agomez/PR/MLID-1862-hotfix.md` — lines 6, 19, 20 explicitly state Supplement does NOT match.

Update these in the same commit or a follow-up tidy-up — not blocking.
