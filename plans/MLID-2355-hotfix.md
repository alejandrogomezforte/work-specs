# MLID-2355 — Hotfix: badge undercounts "Needs Review" (context save)

> **Status:** Documented, NOT yet implemented. Picking back up after a break.
> The badge feature itself (MLID-2355) is implemented, tested, and committed on
> branch `feature/MLID-2355-auth-review-needs-review-counter`. This file captures
> a separate defect found during browser testing that makes the badge number
> disagree with the table.

---

## Symptom (observed in the browser)

- Test order: `698db48da52e0630a255ea82` (displayId `05DC`, patient WeInfuse `8081`,
  "LISA TEST Q"). Open at `/orders-tracker/new/698db48da52e0630a255ea82`.
- The Prior Auth documents table shows **3 rows with "Needs Review"** status
  (and 1 "Reviewed"; 4 rows total).
- The Prior Auth tab **badge shows 2**.
- Expected (AC #1): the badge must match the number of documents in "Needs
  Review" status → should be **3**, not 2.

---

## Root cause: two definitions of "needs review" that disagree

The app decides "needs review" in two different places, and they do not agree.

### 1. Table coloring (display) — `apps/web/services/priorAuth/transforms.ts`

The transform that builds each `PriorAuthLetter` sets:

```js
reviewStatus:
  intake.additionalInfo?.priorAuthData?.reviewStatus === 'reviewed'
    ? 'reviewed'
    : 'needs_review';
```

So a row is shown as **"Needs Review"** whenever the stored field is **anything
other than `'reviewed'`** — this includes documents where the field is **absent**
AND documents where the field is explicitly the string `'needs_review'` (or null,
or empty).

### 2. Badge / status-filter query — `apps/web/services/priorAuth/query.ts`

Inside `buildPriorAuthListPipeline`, the `needs_review` filter is:

```js
if (filters.reviewStatus === 'needs_review') {
  baseMatch['additionalInfo.priorAuthData.reviewStatus'] = { $exists: false };
} else if (filters.reviewStatus === 'reviewed') {
  baseMatch['additionalInfo.priorAuthData.reviewStatus'] = 'reviewed';
}
```

So `needs_review` counts **only documents where the field is absent**. A document
whose field is explicitly set to `'needs_review'` (the field exists) is matched by
**neither** the `needs_review` branch nor the `reviewed` branch — it falls into a
gap. The two filter branches are NOT complements of each other, even though the
display treats them as complements.

The badge (MLID-2355 hook `useAuthReviewNeedsReviewCount`) faithfully reuses the
`reviewStatus=needs_review` filter, so it inherits this undercount. **The defect is
in `query.ts` (MLID-2306 code), not in the badge hook.**

---

## Evidence (MongoDB, `local-infusion-db.intakes`, patient `8081`)

Grouping this patient's prior-auth documents by the raw `reviewStatus` value and
its BSON type:

| `reviewStatus` value | BSON type | count |
|----------------------|-----------|-------|
| `"reviewed"`         | string    | 10    |
| *(field absent)*     | missing   | 8     |
| `"needs_review"`     | string    | 1     |

Exactly **one** document stores the field explicitly as the string
`"needs_review"`; the rest that need review have no field at all.

For order `05DC`'s 4 in-scope docs: 1 `reviewed`, and of the 3 shown as orange,
**2 have the field absent** (counted by the badge) and **1 has the field set to
`"needs_review"`** (not counted). Hence **badge = 2, table = 3**.

> You can reproduce this without the badge at all: in the table's status
> dropdown, choose **"Needs Review"**. The list also drops to 2 rows, because the
> list filter uses the same `query.ts` logic. So the badge is consistent with the
> filter — both disagree with the table coloring.

---

## Suggested fix

Make the filter mirror the display: "needs review" means "not reviewed".

In `apps/web/services/priorAuth/query.ts`, inside `buildPriorAuthListPipeline`,
change the `needs_review` branch:

```js
// before
baseMatch['additionalInfo.priorAuthData.reviewStatus'] = { $exists: false };

// after
baseMatch['additionalInfo.priorAuthData.reviewStatus'] = { $ne: 'reviewed' };
```

In MongoDB, `{ $ne: 'reviewed' }` matches both **missing** fields and any
**non-`'reviewed'`** value, so it becomes the exact complement of the `reviewed`
branch — matching the `transforms.ts` display rule. The badge would then show
**3** for order `05DC`, satisfying AC #1.

---

## Blast radius (decide before implementing)

`buildPriorAuthListPipeline` / `query.ts` is **shared**. The one-line change also:

- Fixes the **order auth-review table's own "Needs Review" status filter**
  (it would show 3 instead of 2). Correct, but it changes MLID-2306 behavior.
- Affects the **patient-level** prior-auth list route, which uses the same
  pipeline. Also correct, also a behavior change beyond the badge.

This is the intended, consistent behavior, but it is broader than the badge.

---

## Open decision (to resume after the break)

1. **Fold the fix into MLID-2355** — the badge's AC #1 depends on it; add it to
   `query.ts` TDD-style on the current branch, with a unit test in
   `apps/web/services/priorAuth/__tests__/query.listPipeline.test.ts` covering
   that `needs_review` now matches both absent and explicit `'needs_review'`.
2. **Separate ticket** against the MLID-2306 prior-auth filter, since it changes
   shared filter behavior beyond the badge.
3. Leave as-is for now.

**Recommendation:** Option 1 — without it the badge does not meet its own AC #1,
and the change is small and well-scoped. Add a focused list-pipeline test so the
intent is locked.

---

## Quick resume checklist

- [ ] Pick option (1 / 2 / 3) above.
- [ ] If implementing: RED test in `query.listPipeline.test.ts` asserting the
      `needs_review` filter produces `{ $ne: 'reviewed' }` (and/or an integration
      assertion that an explicit `'needs_review'` doc is included).
- [ ] GREEN: change the `needs_review` branch in `query.ts` to `{ $ne: 'reviewed' }`.
- [ ] Re-verify in the browser on order `698db48da52e0630a255ea82`: badge → 3,
      and the table's "Needs Review" status filter → 3 rows.
- [ ] Confirm the `reviewed` filter still returns the 10 reviewed docs (unchanged).
