# MLID-2306 — Stage 2: Unresolved Drug-Match Issue (pending PO)

**Status:** UNRESOLVED — blocks finalizing Stage 2 drug scoping.
**Owner decision needed from:** PO (discussion scheduled next day, 2026-06-04).
**Where the logic lives:** `buildOrderScopedPriorAuthMatch` in
`apps/web/services/priorAuth/query.ts` (the order-scoped `$match` appended in
`buildPriorAuthListPipeline`).

---

## Summary of the problem

The Auth Review list scopes a prior-auth document to an order by two AND-ed
conditions: a **date** condition and a **drug** condition. The date condition is
fine. The **drug** condition is the problem.

The prior-auth document stores the drug in **three different fields**, while the
order stores the drug in **one** field (a LISA drug id). Only one of the three
document fields is an id; the other two are free-text names. So today the drug
condition can only compare id-to-id, and that id is only present after a human
reviews the document. Result: for un-reviewed documents (almost all of them
right now), the drug condition matches nothing meaningful and every document
falls through a null/empty passthrough — so the drug filter effectively does not
restrict anything.

---

## The data reality

### The order has one drug field

- `neworders.drug` — a **LISA drug id** (`lidrugs._id` stored as a hex string).
- To get the order's drug **name**, you must look up `lidrugs` by that id
  (`lidrugs.name`). The order document itself stores only the id, not the name.

### The prior-auth document has three drug fields

| # | Full path | Kind | Populated when | Example |
|---|-----------|------|----------------|---------|
| 1a | `additionalInfo.ocrData.service.brandDrugName.value` | text (OCR) | at document processing (pre-review) | `"TEZSPIRE"` |
| 1b | `additionalInfo.ocrData.service.genericDrugName.value` | text (OCR) | at document processing (pre-review) | `"tezepelumab-ekko"` |
| 2 | `additionalInfo.priorAuthData.drugName.value` | text (verified) | only after human review (or MLID-2363) | `"Gammagard Liquid"` |
| 3 | `additionalInfo.priorAuthData.liDrugId.value` | **id** (`lidrugs._id` hex) | only after human review (or MLID-2363) | `"693897569879dc15c6a1de6a"` |

All four are wrapped in an `OCRField` shape: `{ value: string, page: number[] }`.

Notes that make this messy:
- The OCR names have **inconsistent casing** across documents (`TEZSPIRE` vs
  `Tezspire`).
- There is a **brand vs generic** split (two OCR name fields).
- Fields 2 and 3 are empty on un-reviewed documents.
- The OCR `service` object can be empty `{}` on reviewed documents (the verified
  fields take over). So a given document usually has *either* OCR names *or*
  verified fields, not always both.

---

## What is currently implemented (and the bug already fixed)

Current drug condition (after the fix below):

```
$or: [
  { $eq: ['$additionalInfo.priorAuthData.liDrugId.value', <order.drug id>] },   // hard id match
  { $in: [ { $ifNull: ['$additionalInfo.priorAuthData.liDrugId.value', null] }, // null/missing passthrough
           [null, ''] ] }
]
```

This whole `$or` is AND-ed with the date condition. It is **not** an OR between
date and drug — date is always required.

**Bug fixed during Stage 2 testing (2026-06-03):** the null branch originally
used `$in: ['$...liDrugId.value', [null, '']]`, but MongoDB's `$in` does not
match a *missing* field path against `null`. Un-reviewed documents have
`liDrugId` entirely absent (not null), so they were silently excluded and every
order showed zero rows. Fixed by coalescing first with `$ifNull` (shown above).

**Remaining limitation (the unresolved part):** the only positive comparison is
the hard id match on field 3. Because field 3 is empty pre-review, real drug
filtering does not happen yet — un-reviewed documents only pass via the
null/empty passthrough, regardless of the OCR drug name on the letter. So a
Panzyga order shows Ocrevus, Vyepti, Leqvio, etc. documents.

---

## Proposed solution (Alejandro — to confirm with PO)

Extend the concept of "match" so the single order drug is compared against **all
three** document drug representations: a **hard** match on the id, plus **soft**
matches on each of the three name fields, plus the existing null/empty fallback.

Field relations (the idea, not literal field names):

- `additionalInfo.ocrData.service.brandDrugName.value`  → **soft** match to order drug **name**
- `additionalInfo.ocrData.service.genericDrugName.value` → **soft** match to order drug **name**
- `additionalInfo.priorAuthData.drugName.value`          → **soft** match to order drug **name**
- `additionalInfo.priorAuthData.liDrugId.value`          → **hard** match to order drug **id**

Resulting drug condition (show the document if any of these is true):

```
IF  (OCR brand drug name  soft-equals  order drug name)
OR  (OCR generic drug name soft-equals order drug name)
OR  (verified drug name    soft-equals order drug name)
OR  (verified liDrugId      hard-equals order drug id)
OR  (all of the above doc fields are null/empty)        // un-reviewed passthrough
THEN show the document in the table
```

This is still AND-ed with the date condition.

---

## Open questions for the PO

1. **Definition of "soft match".** Exact case-insensitive equality? Trim
   whitespace? Normalized (strip biosimilar suffixes like `-ekko`, `BIOSIMILAR`,
   brand vs generic equivalence)? Substring/contains? The looser it is, the more
   false positives.
2. **Where does the order's drug name come from?** The order stores only the id.
   We must resolve `lidrugs.name` (and possibly a generic name) for the order's
   drug. Does `lidrugs` carry both a brand and a generic name to soft-match
   against both OCR name fields?
3. **Keep the null/empty fallback?** With soft name matching in place, most
   un-reviewed documents *do* have an OCR name, so the fallback would only catch
   documents with no drug information at all. Do we still want those to show, or
   should "no drug info" documents be hidden?
4. **Is MLID-2363 still the long-term answer?** If MLID-2363 reliably backfills
   `liDrugId`, the hard id match becomes the primary path and the soft matches
   become a transitional measure. Confirm timeline and whether soft matching is
   meant to be permanent or temporary.

---

## Implementation impact (once the rule is agreed)

- `buildOrderScopedPriorAuthMatch` (`apps/web/services/priorAuth/query.ts`) gains
  the soft-name comparisons. Soft matching in MongoDB aggregation likely needs
  `$toLower` / `$trim` on both sides, or a `$regexMatch`.
- The order-scoped routes
  (`apps/web/app/api/orders/new/[orderId]/auth-review/route.ts` and
  `.../filter-options/route.ts`) must pass the order's drug **name(s)**, not just
  the id. That means resolving `lidrugs` for the order's drug — extend
  `getNewOrderById` (or a sibling helper) to also return the drug name(s), or do
  a separate `lidrugs` lookup, and inject the name as a literal into the pipeline.
- Unit tests to update: `apps/web/services/priorAuth/__tests__/query.orderScoped.test.ts`
  and the two route test files.

---

## Where the four drug fields are written (confirmed in code, 2026-06-04)

Confirmed by reading both repositories during conversational investigation:
the li-call-processor monorepo and the separate `li-intake` (Python) repo.

### Field group A — OCR drug names (written during OCR processing)

Fields:
- `additionalInfo.ocrData.service.brandDrugName.value`
- `additionalInfo.ocrData.service.genericDrugName.value`

Who writes them, and when:
- The values are produced by the OCR/AI step in the **li-intake** repository
  (Python), returned to this monorepo, and **persisted here**.
- Persistence lives in `@repo/shared`:
  `packages/shared/src/ocr/processOcrDataAndUpdateIntake.ts` — it takes the raw
  OCR data and writes `additionalInfo.ocrData` via `deps.updateIntake(...)`
  (around lines 180-208), setting the intake status to `needsReview`.
- It is invoked by the intake processing job
  `apps/web/services/jobs/definitions/processIntake/processIntakeJob.ts`
  (around line 298, after `intakeAiService.analyzeDoc(...)`), and also from
  `processIntake/pollForCachedOcrResults.ts`.
- li-call-processor stores the names exactly as li-intake returns them; it does
  NOT modify the drug-name values on the li-call-processor side.

li-intake LISA-name alignment (this is what "MLID-2363" refers to):
- `li-intake/src/core/order_rules.py` -> `_align_prior_auth_service_drug_names`,
  added by commit `56abf9b` "MLID-2363 map PA drug names to LISA drugs"
  (2026-05-15). It matches the OCR brand (then generic) name to a LISA medication
  configuration name via `match_medication_configuration_name_only`
  (`li-intake/src/core/page_processors/order/medication_matcher.py`, which returns
  the matched configuration `name`). On a match it overwrites BOTH brand and
  generic with that single LISA name; on no match it sets BOTH to null.
- Data reality (checked in Mongo `intakes`, 2026-06-04): the alignment is NOT
  consistently reflected. Of processed prior-auth documents: 97 still carry raw,
  differing brand/generic names (newest 2026-06-03), 85 are both-null, and only 9
  show the aligned "both equal a LISA name" pattern. So we cannot assume the OCR
  name fields hold a clean LISA name today.

### Field group B — verified drug fields (written during form verification)

Fields:
- `additionalInfo.priorAuthData.drugName.value`
- `additionalInfo.priorAuthData.liDrugId.value`

Who writes them, and when:
- Written when a user verifies the drug in the 3-step prior-auth review form in
  `/patient/[id]/prior-auth` (the same write path is reused by the order Auth
  Review screen `/orders-tracker/new/[id]/auth-review`).
- The form
  `apps/web/components/Modules/patient/components/patientPriorAuth/VerifyDataForm.tsx`
  sets `drugName = selectedLiDrug.name` and `liDrugId = selectedLiDrug._id` when
  the user selects a LISA drug (around lines 117-119). It prefills `drugName` from
  the OCR brand/generic name and re-resolves the LISA drug on re-open via
  `apps/web/services/priorAuth/resolveLisaDrugFromCatalog.ts`.
- The PATCH route
  `apps/web/app/api/patient/[patientId]/prior-auth/[intakeId]/route.ts` persists
  them into `additionalInfo.priorAuthData` along with `reviewStatus: 'reviewed'`
  (around lines 455-461).
- li-intake does NOT set `liDrugId`; it is attached only at verification time
  here.

### Net summary

A prior-auth document carries the drug in three name fields plus one id field; the
order carries only a single LISA drug id (`neworders.drug` = `lidrugs._id`). The
only id-comparable field is `priorAuthData.liDrugId`, set only after human
verification. The OCR name fields are intended (by the li-intake MLID-2363
alignment) to be the LISA name or null, but in current data they are mostly still
raw text.

---

## Decision log

- _(to fill after PO discussion 2026-06-04)_ —
