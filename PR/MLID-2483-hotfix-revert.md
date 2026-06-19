# [MLID-2483] revert(order-documents): restore previous document filter on orders page

## Summary

- Reverts PR #1608 (MLID-2483) on the documents filter in `GET /api/orders/details`. PR #1608 over-tightened the filter and dropped legitimate documents from the Documents tab of the orders page in production.
- The revert restores the pre-PR-1608 rule: **Order-category documents are filtered by `linkedOrderIds`; every non-Order document is always included.**
- The follow-up scoping rules originally targeted by MLID-2483 will be redesigned in a separate ticket once the requirements are pinned down.

**Branch:** `hotfix/MLID-2483` → `develop`

**Reverted commits (now on develop, merged via PR #1608):**

- `ff70bfedb` — `[MLID-2483] - fix(order-documents): scope document list to linked order instead of all patient documents`
- `76c1a727c` — `[MLID-2483] - refactor(order-documents): extract filter variables before payload to fix no-shadow lint error`

---

## Why we are reverting

PR #1608 changed the documents filter from:

```ts
// pre-PR-1608 (what this hotfix restores)
documents.filter(
  (doc) =>
    doc.category !== 'Order' ||
    (doc.linkedOrderIds ?? []).includes(String(trackerOrder._id))
);
```

to a stricter rule that required either:

1. an explicit `linkedOrderIds` match against the tracker order id, OR
2. a shared `intakeId` with an Order-category document already linked to the tracker order.

In production this excluded non-Order documents (Intake, Authorization, Referral Copy, Lab Results, etc.) that previously surfaced through their intake-sibling Order. Users reported missing documents on the orders page Documents tab the morning after the merge.

The intake-id "bridge" assumption — that every legitimate non-Order document either carries the tracker order id directly or shares an intake with a linked Order doc — does not hold for the existing data set. The cleanest unblock is to restore the previous behavior and re-scope MLID-2483 with clearer rules.

---

## Changes

| File | Change |
|------|--------|
| `apps/web/app/api/orders/details/route.ts` | Restored to the pre-PR-1608 inline filter: `doc.category !== 'Order' \|\| (doc.linkedOrderIds ?? []).includes(String(trackerOrder._id))`. The `linkedIntakeIds` set + `orderDocuments` variable extraction from the lint-fix commit are also removed. |
| `apps/web/app/api/orders/details/route.test.ts` | Restored the original "document filtering by linkedOrderIds (Phase 4)" describe block (3 tests). Removed the 4 cases added by PR #1608 that asserted the stricter intake-bridge behavior. |

### NOT touched

- No other route, service, or UI file. The scope of the revert matches the scope of PR #1608 exactly.
- No new tests added — the restored 3 test cases describe the behavior we are recommitting to.
- No data migration / no DB changes.

---

## Verification

| Check | Result |
|-------|--------|
| `npx jest app/api/orders/details/route.test.ts` (apps/web) | 39/39 passed |
| `npm run types:check` (apps/web) | Clean |
| `npx eslint app/api/orders/details/route.ts app/api/orders/details/route.test.ts` | Clean |
| `git diff ff70bfedb~1 HEAD -- <both files>` | Empty — files exactly match pre-PR-1608 state |
| Manual verification on the orders page | User-confirmed: documents tab again shows the documents that disappeared after PR #1608 |

### Restored test cases (in `apps/web/app/api/orders/details/route.test.ts`)

Inside the `document filtering by linkedOrderIds (Phase 4)` describe block:

- `includes Order-category documents that are linked to the current order`
- `excludes Order-category documents not linked to the current order`
- `always includes non-Order-category documents regardless of linkedOrderIds`

These three cases collectively pin down the rule we are reverting to.

---

## Reviewer checklist

- [ ] Pull `hotfix/MLID-2483`, run `npx jest app/api/orders/details/route.test.ts` and confirm 39/39 pass.
- [ ] Open the orders page Documents tab for an order that lost documents after PR #1608 — confirm the previously-missing documents are listed again.
- [ ] Confirm the documents tab still scopes Order-category documents to the current order via `linkedOrderIds` (an Order doc explicitly linked to a different tracker order should not appear).
- [ ] Confirm no other behavior on `GET /api/orders/details` changed (everything outside the documents-filter block is byte-identical).

---

## Follow-up

- MLID-2483 itself needs to be re-scoped with the PO. The original ticket intent — "scope the document list to documents linked to the tracker order" — is still valid; the intake-id bridge approach is what broke. The redesign should pick up after this revert lands on develop.

---

## Jira

- [MLID-2483](https://localinfusion.atlassian.net/browse/MLID-2483) — original ticket. Will need to be reopened (or a successor ticket created) for the redesign.
