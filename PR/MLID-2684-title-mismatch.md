# [MLID-2684] fix(prior-auth): use document description as the auth-review preview title

## Summary
- The Order Details → Prior Auth preview panel titled the PDF preview with the
  letter's drug value, which for some letters is a raw HCPCS J-code (for example
  `J3404`) instead of a readable name.
- The parsed "Prior Authorization" document already carries a human-readable
  `description`. This change uses that description as the preview title in the
  embedded side panel, the full-screen expand dialog, and the detail takeover.
- The detail takeover was also sourcing its description from the wrong document
  (the newest document of any category); it now uses the same selected
  prior-auth document, so all surfaces agree.

## Root Cause
The preview title was `previewLetter.drug`. The drug is derived from the intake
OCR field `additionalInfo.ocrData.service.serviceRequested.value` (for example
`"PAPZIMEOS (J3404)"`) through `normalizeServiceDrugName`. That function prefers
a parenthesized brand name (designed for inputs like
`"IMMUNE GLOBULIN (PRIVIGEN)"`), so when the parentheses contain the HCPCS J-code
it returns `J3404` and discards the real name `PAPZIMEOS`. The drug field is
fine for the table's Drug column but is the wrong source for a document title.

Separately, the detail route computed its description from `documents[0]`, and
`getDocumentsByIntakeId` sorts newest-first, so the newest document of any
category won (for example an `Other` split with description `"other"`) rather
than the prior-auth document.

> Follow-up to the earlier MLID-2684 PR (preview parsed document instead of whole
> fax), which fixed the preview content but not the title.

## Changes Overview
- **Files changed**: 9 (5 source, 4 test)
- **Lines added**: +288
- **Lines removed**: -18

## Files Changed

| File | Change | Description |
|------|--------|-------------|
| `apps/web/services/priorAuth/selectPriorAuthDocument.ts` | Modified | Add `selectPriorAuthDocument` returning the chosen "Prior Authorization" document; `selectPriorAuthSplitDocument` now wraps it and returns `.blobPath`. |
| `apps/web/types/priorAuth.ts` | Modified | Add optional `PriorAuthLetter.documentDescription`. |
| `apps/web/services/priorAuth/transforms.ts` | Modified | Select the prior-auth document once; derive both `documentUrl` and the trimmed `documentDescription` from it. |
| `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/AuthReviewIndex.tsx` | Modified | Preview panel header and `EmbeddedFileViewer` `fileName` use `documentDescription \|\| 'Prior Auth Letter'`. |
| `apps/web/services/priorAuth/detail.ts` | Modified | Source `documentDescription` from `letter.documentDescription ?? documents[0]?.description`. |
| `apps/web/services/priorAuth/selectPriorAuthDocument.test.ts` | Modified | Cases for the new document selector. |
| `apps/web/services/priorAuth/transforms.test.ts` | Modified | `documentDescription` set / trimmed / undefined. |
| `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/AuthReviewIndex.test.tsx` | Modified | Preview title and viewer `fileName` (description and fallback). |
| `apps/web/services/priorAuth/__tests__/detail.test.ts` | Modified | Prior-auth description wins over newest doc; fallback when no prior-auth doc; plus verify-data coverage. |

## What Changed and Why

### 1. One document, one source of truth
`selectPriorAuthSplitDocument` previously returned only the `blobPath`. It now
delegates to a new `selectPriorAuthDocument`, which returns the whole chosen
document (most recently created `Prior Authorization` row). The transform reads
both the preview URL and the preview title from that single document, so the
title always describes the exact PDF being rendered.

### 2. Title comes from the document, not the drug
`transformIntakeToPriorAuthLetter` now sets `documentDescription` from the
selected document's trimmed `description` (undefined when absent or empty). The
`AuthReviewIndex` preview panel uses it for both the header and the
`EmbeddedFileViewer` `fileName`:

```
Before: {previewLetter.drug || 'Prior Auth Letter'}          -> "J3404"
After:  {previewLetter.documentDescription || 'Prior Auth Letter'}
        -> "Humana approval for PAPZIMEOS with authorization dates and reference number"
```

The single `fileName` change also fixes the full-screen expand dialog, which
uses `fileName` as its title. The fallback is the neutral `'Prior Auth Letter'`
label rather than the drug, so a missing description never resurfaces the wrong
J-code.

### 3. Detail takeover uses the same selection
`getPriorAuthDetail` now derives its description from
`letter.documentDescription ?? documents[0]?.description`. It prefers the
selected prior-auth document; the old `documents[0]` value is kept only as a
fallback for intakes that have no prior-auth split document, so no description
is lost in that case.

### Alternatives considered
- Fixing `normalizeServiceDrugName` to skip parenthesized HCPCS codes was
  rejected: the drug field feeds the table's Drug column and is a separate
  concern from the document title. The document already carries the right text.

## Commits
| Hash | Message |
|------|---------|
| `0349dbadf` | [MLID-2684] - fix(prior-auth): use document description as the auth-review preview title |
| `07349095b` | [MLID-2684] - test(prior-auth): cover getPriorAuthDetail verify-data and applyPriorAuthReview write path |

## Test Plan
- [ ] Open an order's Order Dashboard → Prior Auth sub-tab; click the preview
      icon on a row whose drug shows as a J-code. The side-panel title reads the
      document description, not the J-code.
- [ ] Click the full-screen expand icon in that preview; the dialog title reads
      the document description.
- [ ] Click "Open" to the detail takeover; the preview title reads the document
      description (not a sibling document's description such as "other").
- [ ] A letter whose intake has no parsed prior-auth document falls back to the
      neutral "Prior Auth Letter" title without error.
- Automated: full suite green (1384 suites, 20,205 tests), types clean, coverage
  ≥ 87% on all changed source files (`detail.ts` 94.98%).

## Jira
- [MLID-2684](https://localinfusion.atlassian.net/browse/MLID-2684)
