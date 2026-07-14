# MLID-2306 — Why some reviewed auth documents disappear from the Order Details list

## Reported by QA (Daria)

> I noticed different behavior when reviewing auth documents in Order Details.
>
> In one case, after I reviewed a document, it disappeared from the list
> immediately (approval, drug - Avsola). However, in another case, after
> reviewing a different document, it remained visible in the list (denial, drug -
> Venofer).
>
> If so, what determines whether a reviewed document stays in the list or
> disappears?

## Short answer

What determines it is **whether the LISA drug that the reviewer assigns to the
document during review matches the order's own drug**.

The Order Details "Auth Review" list filters prior-auth documents by drug. That
filter behaves in two completely different ways depending on whether the
document has been reviewed:

- **Before review** the document has no assigned drug id (`liDrugId` is empty).
  The filter falls back to a permissive rule: if the document has no OCR
  brand/generic drug name at all, it is shown under **every** order regardless
  of the order's drug (a "passthrough"). It can also be shown by a fuzzy
  drug-name text match.
- **After review** the reviewer's drug selection is written to the document as
  `liDrugId`. Once that id is present, the permissive fallback no longer applies
  and the filter switches to a **strict drug-id equality** rule: the document is
  kept only if its `liDrugId` equals the order's drug id, and is otherwise
  removed.

So reviewing a document changes how it is matched. A document that was only
showing because of the permissive passthrough will disappear the moment review
pins it to a drug that is different from the order's drug. A document whose
assigned drug equals the order's drug stays, because it satisfies the strict
equality rule too.

The two cases Daria saw are textbook examples of each path:

- **Avsola approval (disappeared):** the order's drug is **Panzyga**, but the
  document was reviewed and mapped to **Avsola**. Different drug, so after review
  the strict rule removed it. Before review it had no OCR drug name, so the
  passthrough had been showing it under the Panzyga order even though the
  document is not about Panzyga.
- **Venofer denial (stayed):** the order's drug is **Venofer** and the document
  was reviewed and mapped to **Venofer**. Same drug, so the strict equality rule
  keeps it.

## Where this happens in the code

- Endpoint: `apps/web/app/api/orders/new/[orderId]/auth-review/route.ts`
- Pipeline / match: `buildOrderScopedPriorAuthMatch` in
  `apps/web/services/priorAuth/query.ts`
- Where `liDrugId` is written on review: the review wizard
  (`usePriorAuthReviewWizard.ts`) sends the selected `liDrugId` in the PATCH
  body, and `applyPriorAuthReview` in `apps/web/services/priorAuth/detail.ts`
  persists it into `additionalInfo.priorAuthData.liDrugId`.

The order-scoped drug condition in `buildOrderScopedPriorAuthMatch` has two
mutually exclusive branches:

1. **Hard id branch** — only contributes when the order has a drug id. It keeps
   the document when `additionalInfo.priorAuthData.liDrugId.value` equals the
   order's drug id.
2. **Fallback branch** — only active when `liDrugId` is **null or empty**. It
   keeps the document when any of these is true:
   - complete passthrough: the document has no OCR brand name and no OCR generic
     name (nothing to compare against), OR
   - the order's drug name fuzzy-matches the document's OCR brand name, OR
   - the order's generic name fuzzy-matches the document's OCR generic name, OR
   - the not-enough-evidence passthrough.

The key detail is that the fallback branch is gated on `liDrugId` being empty
(`{ $in: [liDrugIdValue, [null, '']] }`). Reviewing the document populates
`liDrugId`, which switches off the entire fallback branch. From that point on,
the only way the document can stay is the hard id branch, i.e. exact drug-id
equality with the order.

Note that the date condition is **not** the differentiator here. Both documents
pass the date check:

- Avsola: no `dateOfDecision`, so it falls back to the intake `createdAt`
  (2026-05-22), which is after the order's `createdAt` (2026-03-17).
- Venofer: `dateOfDecision` 2026-01-21, which is after the order's `createdAt`
  (2026-01-19).

## Is this a bug?

The "disappearance" itself is the filter working as designed: after review the
Avsola document is correctly recognized as a different drug than the Panzyga
order and is removed. The real inconsistency is on the **other** side: before
review, the permissive passthrough was showing the Avsola document under the
Panzyga order even though it is not a Panzyga document. In other words, the
document should arguably not have been in the Panzyga list to begin with. The
passthrough rule (show documents that have no OCR drug evidence under every
order) is what makes the pre-review and post-review behavior look
contradictory to a reviewer.

This document only describes the behavior and its cause. It does not prescribe a
fix.

## Order and document data

### Order 1 — document disappeared after review

Order (`neworders`):

```
_id:        69b9a19726b2f297cf1e9e62
displayId:  05Ep
createdAt:  2026-03-17T18:46:47.122Z
orderType:  New Start
patient:    6971e73af002e9d03851a4bd   (we_infuse_id_IV: "7719")
drug:       6938992b9879dc15c6a1de6e   -> Panzyga (no genericName)
```

The reviewed document (`intakes`) that disappeared:

```
_id:               6a108bf07040d5ee75402d72
status:            processed
reviewStatus:      reviewed
createdAt:         2026-05-22T17:01:36.234Z
documentType:      Approval
dateOfDecision:    (none)
brandDrugName:     (none)
genericDrugName:   (none)
liDrugId.value:    69de2a73db1282f388db64a2   -> Avsola   <-- assigned during review
priorAuthDrugName: "Avsola"
```

Why it disappeared:

- Drug id condition: order drug is Panzyga (`6938992b...`); the document's
  reviewed `liDrugId` is Avsola (`69de2a73...`). Not equal, so the hard id
  branch fails.
- Because `liDrugId` is now set, the fallback branch is switched off entirely.
- Result: the document no longer matches and drops out of the list.
- Before review, `liDrugId` was empty and the document had no OCR brand/generic
  drug name, so the complete-passthrough clause had been showing it under the
  Panzyga order.

### Order 2 — document stayed after review

Order (`neworders`):

```
_id:        6a3cfffdf86aefc0df04abe6
displayId:  05rw
createdAt:  2026-01-19T10:16:29.172Z
orderType:  New Start
patient:    69cba6b202323d384fe6213b   (we_infuse_id_IV: 8808)
drug:       6a3cffb9f86aefc0df04abd0   -> Venofer (genericName: "Iron Sucrose")
```

The reviewed document (`intakes`) that stayed:

```
_id:               697194bf2f0f60efb4e3626d
status:            processed
reviewStatus:      reviewed
createdAt:         2026-01-22T03:08:47.810Z
documentType:      Denial
dateOfDecision:    2026-01-21
brandDrugName:     "VENOFER"
genericDrugName:   "Venofer"
liDrugId.value:    6a3cffb9f86aefc0df04abd0   -> Venofer   <-- assigned during review
priorAuthDrugName: "Venofer"
```

Why it stayed:

- Drug id condition: order drug is Venofer (`6a3cffb9...`); the document's
  reviewed `liDrugId` is also Venofer (`6a3cffb9...`). Equal, so the hard id
  branch keeps it.
- It would also have matched before review via the fuzzy brand-name match
  (order name "Venofer" contained in OCR brand name "VENOFER"), so this document
  shows consistently both before and after review.

## Summary table

| | Order 1 (disappeared) | Order 2 (stayed) |
|---|---|---|
| Order id (`neworders`) | `69b9a19726b2f297cf1e9e62` | `6a3cfffdf86aefc0df04abe6` |
| Reviewed document id (`intakes`) | `6a108bf07040d5ee75402d72` | `697194bf2f0f60efb4e3626d` |
| Open document in `/intakes` | `/intakes/6a108bf07040d5ee75402d72` | `/intakes/697194bf2f0f60efb4e3626d` |
| Order drug | Panzyga (`6938992b...`) | Venofer (`6a3cffb9...`) |
| Document type | Approval | Denial |
| Document `liDrugId` after review | Avsola (`69de2a73...`) | Venofer (`6a3cffb9...`) |
| Drug id equals order drug? | No | Yes |
| OCR brand drug name on doc (`ocrData.service.brandDrugName.value`) | (none) | "VENOFER" |
| OCR generic drug name on doc (`ocrData.service.genericDrugName.value`) | (none) | "Venofer" |
| Shown before review because | complete passthrough — no OCR brand or generic name, nothing to compare | fuzzy brand-name match — order "Venofer" contained in OCR brand "VENOFER" |
| Shown after review because | nothing — hard id mismatch, fallback off | hard id match |
| Net result | removed after review | kept after review |

## The determinant, in one sentence

A reviewed prior-auth document stays in an order's Auth Review list only if the
LISA drug the reviewer assigned to it (`liDrugId`) is the same drug as the
order; if the reviewer maps it to a different drug, reviewing it switches the
match from the permissive name/passthrough fallback to strict drug-id equality
and the document drops out.
