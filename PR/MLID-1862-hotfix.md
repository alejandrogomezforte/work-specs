# [MLID-1862] fix(pharmacy-eligibility): anchor Medicare regex to reject partial matches

## Summary

- **Fix**: Pharmacy eligibility Medicare regex was unanchored, causing payer types like "Medicare Supplement" and "Medicare Part D" to incorrectly pass eligibility
- **Root cause**: The regex `medicare(\s+advantage)?` matched any string *starting with* "medicare" instead of matching exactly "Medicare" or "Medicare Advantage"
- **Solution**: Anchored regex to `^\s*medicare(\s+advantage)?\s*$` -- rejects partial matches while tolerating leading/trailing whitespace

## Changes

| File | Change |
|------|--------|
| `services/mongodb/ordersTracker.ts` | Anchored Medicare regex with `^$` and `\s*` whitespace tolerance |
| `services/mongodb/__tests__/ordersTracker.test.ts` | Added 7 tests for regex anchoring, partial match rejection, and whitespace handling |

## Test plan

- [x] "Medicare" matches (passes eligibility)
- [x] "Medicare Advantage" matches (passes eligibility)
- [x] "Medicare Supplement" does NOT match (fails eligibility)
- [x] "Medicare Part D" does NOT match (fails eligibility)
- [x] Leading/trailing whitespace on "Medicare" and "Medicare Advantage" still matches
- [x] All 52 ordersTracker tests pass
- [x] Coverage: 98.1% statements, 92.3% branches, 100% functions

## Jira

[MLID-1862](https://localinfusion.atlassian.net/browse/MLID-1862) (subtask of MLID-1492)
