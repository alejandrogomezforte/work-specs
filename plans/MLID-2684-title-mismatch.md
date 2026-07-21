# [MLID-2684] fix(prior-auth): use document description as the auth-review preview title

**Status: In QA** — merged to `develop` via PR from
`feature/MLID-2684-preview-title-mismatch` (commits `0349dbadf`, `07349095b`).
Full suite green, types clean, coverage ≥87% on all changed source files.
PR draft: `docs/agomez/PR/MLID-2684-title-mismatch.md`.

## Problem

On **Order Details → Order Dashboard → Prior Auth**
(`/orders-tracker/new/[id]/auth-review`), clicking the preview icon on a row
opens a side preview panel. The panel title (and the full-screen expand dialog
title) currently shows the drug value, which for some letters is a raw HCPCS
J-code such as `J3404` instead of a readable name.

Root cause (see `docs/agomez/postmortem/MLID-2684-pdf-preview-title.md`): the
title is `previewLetter.drug`, resolved from the intake OCR field
`additionalInfo.ocrData.service.serviceRequested.value` (`"PAPZIMEOS (J3404)"`)
through `normalizeServiceDrugName`, whose "prefer the parenthesized brand"
heuristic wrongly picks the parenthesized J-code.

## Prior work (why this is still open)

- PR 1896 / commit `49f7aec` (MLID-2684) fixed the preview **content**
  (`documentUrl` → the split `Prior Authorization` blob instead of the whole
  fax) and added `selectPriorAuthSplitDocument`. It did not touch any title.
- The `documentDescription` plumbing on the detail screen — the
  `PriorAuthDetailResponse.documentDescription` field, the detail-page title
  binding, and the `documents[0]?.description` source — came from MLID-2306
  (commit `0abe3b9ad`, "add order Auth Review detail screen Stage 3").
- Net: the side-panel title still uses `drug`, `PriorAuthLetter` has no
  `documentDescription`, and the detail source is `documents[0]` (wrong
  document). This plan is the remaining, non-overlapping work.

## Decision

Stop deriving the preview title from OCR drug parsing. The split
`Prior Authorization` document in the `documents` collection already carries a
human-readable `description` (e.g. "Humana approval for PAPZIMEOS with
authorization dates and reference number"). Show **that** field as the preview
title, in both the embedded side-panel header and the full-screen expand dialog.

The full-screen **detail** takeover (`AuthReviewDetail.tsx`) already renders
`data.documentDescription`, but that value is sourced from the wrong document
(the newest document of any category, not the prior-auth one), so it shows the
wrong text too. Both the side panel (`AuthReviewIndex.tsx`) and the detail
route's description source are corrected here.

## Scope

In scope:
- Surface the selected prior-auth document's `description` on the shared
  `PriorAuthLetter` transform so the list/preview has it.
- Use it as the side-panel preview title and as the `EmbeddedFileViewer`
  `fileName` (which drives the full-screen expand dialog title).
- Correct the detail route's `documentDescription` source so the takeover
  page and removal-panel prefill use the prior-auth document, not the newest
  document of any category.

Out of scope:
- Changing `normalizeServiceDrugName` / drug resolution (still used for the
  table's Drug column — that is a separate concern, not the title).
- The patient-card prior-auth surface (no equivalent preview-panel title there;
  only `AuthReviewIndex.tsx` uses the `drug`-as-title pattern).

## Data / source of truth

- Collection: `documents`; field: `description`.
- The relevant document is the split page with `category === 'Prior Authorization'`
  for the intake — already selected by
  `apps/web/services/priorAuth/selectPriorAuthDocument.ts`
  (`selectPriorAuthSplitDocument`, "most recently created Prior Authorization
  document"). We reuse that exact selection so the title always describes the
  same PDF the panel renders.
- Example object (staging): document `6a58c244b4ef88e69ceab030`,
  `description = "Humana approval for PAPZIMEOS with authorization dates and reference number"`.

## Implementation (TDD, red → green → refactor)

### Step 1 — Select the prior-auth document once (`selectPriorAuthDocument.ts`)

Refactor so both the blob path and the description come from the *same* chosen
document, avoiding two divergent selectors.

- Add `selectPriorAuthDocument(documents): IDocument | undefined` that returns
  the chosen document (the existing "filter by category `Prior Authorization`,
  reduce to most-recent `createdAt`" logic).
- Re-express `selectPriorAuthSplitDocument` as a thin wrapper:
  `selectPriorAuthDocument(documents)?.blobPath`.

Tests (`selectPriorAuthDocument.test.ts`): keep existing blob-path cases green;
add cases asserting `selectPriorAuthDocument` returns the most-recent
`Prior Authorization` document and `undefined` when none exists.

### Step 2 — Carry `documentDescription` on the letter (`transforms.ts` + type)

- `apps/web/types/priorAuth.ts`: add optional field to `PriorAuthLetter`:
  `documentDescription?: string` (JSDoc: "Description of the selected prior-auth
  document; used as the preview title").
- `apps/web/services/priorAuth/transforms.ts`
  (`transformIntakeToPriorAuthLetter`): compute the selected document once via
  `selectPriorAuthDocument(documents)`; set `documentUrl` from its `blobPath`
  (unchanged behavior) and set `documentDescription` from its `description`
  (trim; omit when empty/absent).

Tests (`transforms.test.ts`, RED first):
- Given documents including a `Prior Authorization` doc with a `description`,
  the letter's `documentDescription` equals that description.
- Given no `Prior Authorization` document, `documentDescription` is `undefined`
  and `documentUrl` falls back to the intake PDF (existing behavior preserved).

Note: both list routes (`/api/orders/new/[orderId]/auth-review` and
`/api/patient/[patientId]/prior-auth`) already pass the intake's documents into
this transform, so the new field is populated with no route changes.

### Step 3 — Use the description as the preview title (`AuthReviewIndex.tsx`)

In the preview side panel:
- Header title (currently `previewLetter.drug || 'Prior Auth Letter'`) →
  `previewLetter.documentDescription || 'Prior Auth Letter'`.
- `EmbeddedFileViewer` `fileName` prop (currently `previewLetter.drug || 'Prior Auth Letter'`) →
  `previewLetter.documentDescription || 'Prior Auth Letter'`.

Fallback stays `'Prior Auth Letter'` (a neutral label) rather than the drug, so
a missing description never resurfaces the wrong J-code. This single `fileName`
change fixes both the embedded panel and the full-screen expand dialog, since
`EmbeddedFileViewer` passes `fileName` to its full-screen `Dialog title` and to
`ReactPdfRenderer`.

Tests (`AuthReviewIndex.test.tsx`): rendering the preview panel for a letter with
`documentDescription` shows the description in the panel header; falls back to
`'Prior Auth Letter'` when absent.

### Step 4 — Align detail route source (in scope)

`apps/web/services/priorAuth/detail.ts` currently derives
`documentDescription` from `documents[0]?.description`. Because
`getDocumentsByIntakeId` sorts `createdAt: -1`, `documents[0]` is the *most
recently created* document of **any** category — not necessarily the prior-auth
one. For the reference intake `6a57c04d85d689c573fc4baa` the newest document is
the `Other` split (description `"other"`, created ~1s after the prior-auth doc),
so the detail takeover title currently shows `"other"` — the same class of bug
on a sibling surface.

Replace the source with the letter's selected-document description, keeping the
old value as a fallback:

```ts
const documentDescription =
  letter.documentDescription ?? documents[0]?.description;
```

(Compute `letter` first, then derive `documentDescription` from it.)

Consequences:
- Shared by both detail routes — patient
  (`/api/patient/[patientId]/prior-auth/[intakeId]`) and order
  (`/api/orders/new/[orderId]/auth-review/[letterId]`).
- Fixes the detail takeover title (`AuthReviewDetail.tsx`): `"other"` →
  the prior-auth document's description, matching the side panel.
- Also changes the "This is not a Prior Authorization letter" removal-panel
  description prefill to the prior-auth document's description.
- The `?? documents[0]?.description` fallback preserves current behavior for
  intakes that have no `Prior Authorization` split document (letter falls back
  to the whole fax), so no description is lost in that case.

Tests (`detail.test.ts`, RED first):
- When the newest document is not the prior-auth doc, `documentDescription`
  equals the prior-auth document's description (not the newest doc's).
- When no prior-auth document exists, `documentDescription` falls back to
  `documents[0]?.description`.

## Files

| File | Change |
|---|---|
| `apps/web/services/priorAuth/selectPriorAuthDocument.ts` | add `selectPriorAuthDocument`; wrap `selectPriorAuthSplitDocument` |
| `apps/web/services/priorAuth/selectPriorAuthDocument.test.ts` | new cases for the document selector |
| `apps/web/types/priorAuth.ts` | add `PriorAuthLetter.documentDescription?` |
| `apps/web/services/priorAuth/transforms.ts` | populate `documentDescription` from selected doc |
| `apps/web/services/priorAuth/transforms.test.ts` | assert `documentDescription` set / undefined |
| `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/AuthReviewIndex.tsx` | title + `fileName` use `documentDescription` |
| `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/AuthReviewIndex.test.tsx` | preview-title assertions |
| `apps/web/services/priorAuth/detail.ts` | use `letter.documentDescription ?? documents[0]?.description` |
| `apps/web/services/priorAuth/__tests__/detail.test.ts` | assert prior-auth description wins; fallback when no PA doc |

## Verification

1. `cd apps/web && npm run test -- transforms selectPriorAuthDocument AuthReviewIndex`
2. `npm run types:check`
3. Browser (user): open the order's Prior Auth preview for the `J3404` row and
   confirm the title reads the document description in both the embedded panel
   and the full-screen expand.
