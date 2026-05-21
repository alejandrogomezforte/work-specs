# [MLID-2192] chore(orders): add 340B test coverage and orders eligibility explainer docs

## Summary

- Follow-up to MLID-2192. Adds a defensive structural test that pins the `hospital340BEligible` `$switch` to the same `patient.primaryInsurance` field that pharmacy eligibility reads from — so a future refactor cannot silently disconnect 340B from the shared `patient_insurances` lookup that MLID-2192 widened.
- Introduces a new `docs/orders/` directory with two reference documents (340B eligibility, pharmacy eligibility) that explain both rules end-to-end for engineers and AI agents — what the rule answers, decision-flow diagrams, data sources, full pipeline-stage code chunks, edge cases.
- No production code change. `apps/web/services/mongodb/ordersTracker.ts` is untouched.

## Background

After MLID-2192 merged, the Product Owner asked whether the same `Undetermined`-status acceptance should also benefit the 340B Eligibility column. Investigation confirmed it already does — both `$switch` blocks (340B and pharmacy) consume the same `$patient.primaryInsurance` field, populated by the single `patient_insurances` `$lookup` MLID-2192 widened. There was no production code change to make.

But the link between the shared lookup and the 340B `$switch` was implicit — never asserted by any test. A future refactor moving 340B's branch off `$patient.primaryInsurance.payer_type_name` (for example, into a separate temp variable or a new lookup) would silently disconnect the status filter from 340B and reintroduce the bug. This PR adds the missing structural assertion.

The new `docs/orders/` files address a separate but related gap: there is no team-shared explanation of how either eligibility rule actually works. The internal analysis docs in `docs/agomez/analysis/` predate the current code shape and contain architectural drift (they describe a `_primaryInsurance` shared temp lookup that no longer exists). The new docs reflect the current production code.

## Changes Overview

- **Files changed**: 3 (1 modified, 2 new)
- **Lines added**: +783
- **Lines removed**: 0

## Files Changed

| File | Change | Description |
|------|--------|-------------|
| `apps/web/services/mongodb/__tests__/ordersTracker.test.ts` | Modified | Adds a new `describe('hospital340BEligible $switch reads from patient.primaryInsurance')` block inside `with includeEligibility`. Two `it()` cases extract the `hospital340BEligible` `$switch` from the executed pipeline, JSON-stringify its branches, and assert that both `$patient.primaryInsurance` and `$patient.primaryInsurance.payer_type_name` appear in the structure. Style and helpers match the `patient_insurances lookup status filter` block from MLID-2192. |
| `docs/orders/340b-eligibility.md` | New | Engineer-and-AI-agent explainer for the `hospital340BEligible` rule. Covers: what the rule answers, possible result values, decision-flow ASCII diagram (Gates 1–3), data-source tables for all 5 collections, data-flow diagram, full pipeline-stage code chunks, edge cases, UI/cache integration, related rules, AI-agent quick-reference. |
| `docs/orders/pharmacy-eligibility.md` | New | Same structure for `pharmacyEligible`. Covers Gates 1–5 with two paths (Medicare auto-pass vs. Commercial/Medicaid + plan check), all 6 collections, full pipeline code, edge cases, recent rule changes (MLID-2192, MLID-2194, plus the 2026-03 changes), and AI-agent quick-reference. |

## What Changed and Why

### `ordersTracker.test.ts` — 340B structural assertion

Existing tests in `with includeEligibility` check stage presence by `from`/field name but never walk into the `hospital340BEligible` `$switch` body. None of them would catch a refactor that disconnects 340B from `$patient.primaryInsurance`.

The new block extracts the `$switch` and asserts the field-path strings appear:

```ts
describe('hospital340BEligible $switch reads from patient.primaryInsurance', () => {
  async function get340BBranches() { /* ...same shape as MLID-2192 helper... */ }

  it('should reference $patient.primaryInsurance in its $switch (shared lookup with pharmacy)', async () => {
    const branches = await get340BBranches();
    expect(JSON.stringify(branches)).toContain('$patient.primaryInsurance');
  });

  it('should regex-match payer_type_name read from $patient.primaryInsurance', async () => {
    const branches = await get340BBranches();
    expect(JSON.stringify(branches)).toContain(
      '$patient.primaryInsurance.payer_type_name'
    );
  });
});
```

The JSON-stringify-and-`toContain` shape is intentional: strict enough to fail when 340B is rewired off the shared lookup, loose enough to survive cosmetic reordering of `$switch` branches.

**Alternatives considered:**

- Walk the `$switch.branches` array structurally and assert specific shape — rejected. More fragile against legitimate refactors of the branch order or null-handling.
- Add a behavioral end-to-end test (run the pipeline against seeded fixtures, assert the resulting `hospital340BEligible` value) — rejected for this PR. The structural test catches the regression we're worried about; behavioral coverage is better added when there's a non-trivial branch shape change to validate.

### `docs/orders/340b-eligibility.md` and `docs/orders/pharmacy-eligibility.md` — new explainers

Both documents follow a consistent structure so a reader can switch between them without re-orienting:

1. Frontmatter (computed field name, file location, function, computation strategy)
2. What the rule answers (plain-English)
3. Possible result values (with table)
4. Decision flow (ASCII diagram)
5. Data sources (per-collection field-level table + data-flow diagram)
6. Implementation walkthrough (actual pipeline-stage code chunks)
7. Edge cases (null handling, missing fields, casing, anchoring)
8. How the UI gets this value (cache + change-stream notes, with stale-cache caveat)
9. Recent rule changes / Related rules
10. Quick reference for AI agents

Two corrections the new docs make over the older `docs/agomez/analysis/` material:

- `patient.primaryInsurance` is populated by an upstream `$lookup` inside `expandIdsAggregate`, not by a shared `_primaryInsurance` temp field that gets unset after eligibility computes. Both `$switch` blocks consume the same flattened subdocument, which is also what the New Orders "Insurance" display column reads.
- The patient_insurances filter accepts `Undetermined` (MLID-2192) and the pharmacy regex includes `Medicare Supplement` ([MLID-2194](https://localinfusion.atlassian.net/browse/MLID-2194)). Pharmacy doc has a "Recent rule changes" section recording these as dated entries alongside the older 2026-03 changes (drug check inversion + treatment gate).

## Commits

| Hash | Message |
|------|---------|
| `aa1950f1` | `[MLID-2192] - test(orders-tracker): lock down hospital340BEligible $switch contract` |
| `1af14d08` | `[MLID-2192] - docs(orders): add 340B and pharmacy eligibility explainers` |

## Test Plan

### Automated

- `cd apps/web && npx jest --testPathPattern="ordersTracker" --no-coverage` → 140/140 pass (138 pre-existing + 2 new)
- `cd apps/web && npx tsc --noEmit` → clean
- `cd apps/web && npx eslint services/mongodb/__tests__/ordersTracker.test.ts` → clean
- `cd apps/web && npx prettier --check services/mongodb/__tests__/ordersTracker.test.ts ../../docs/orders/340b-eligibility.md ../../docs/orders/pharmacy-eligibility.md` → clean

### Manual

No manual QA required. This is a test + docs PR with no production code change. The new test asserts current behavior (passes on first run with no underlying change), so it cannot regress any deploy.

Recommended reviewer checks:

- [ ] Open `apps/web/services/mongodb/__tests__/ordersTracker.test.ts` and confirm the new describe block is inside `with includeEligibility`, not at the top level. Co-locating with the other eligibility-pipeline tests is intentional.
- [ ] Skim the `Decision flow` ASCII diagrams in both new doc files. They are the most synopsis-level summary of each rule and the part most easily verified against the code in `apps/web/services/mongodb/ordersTracker.ts`.
- [ ] Sanity-check the line-number references in both doc files against `getEligibilityPipelineStages()` (the docs cite `lines 25–224` and `lines 362–386` for the upstream insurance lookup — these match `develop` at the time of writing).

## Jira

- [MLID-2192](https://localinfusion.atlassian.net/browse/MLID-2192) — Add Patient Insurance If the Status is Undetermined / Order Tracker + Pharmacy Calculation (parent ticket)
- Parent epic: [MLID-2097](https://localinfusion.atlassian.net/browse/MLID-2097) — Random Tasks (bucket epic; standalone task, no architectural plan)
