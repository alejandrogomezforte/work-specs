# MLID-2684 — PDF preview title for `documents` collection shows wrong text

Bug. Assignee: Alejandro Gomez.

> Companion to `MLID-2684.md` (which covers why the preview *content* differs
> between the Prior Auth table and the Documents tab). This file tracks a
> separate finding: the PDF preview **title** rendered for a `documents`
> collection object does not match the expected text.

## Problem statement

On **Order Details → Order Dashboard → Prior Auth** sub-tab
(`/orders-tracker/new/[id]/auth-review`), clicking the preview icon on a
prior-auth row opens a side preview panel. The panel **title** shows a strange
string instead of the drug name.

- **Surface:** `/orders-tracker/new/[id]/auth-review` (order id
  `6a58c06cb4ef88e69ceaad18`), Prior Auth sub-tab, preview side panel header.
- **Expected title:** the drug name, `PAPZIMEOS`.
- **Actual title:** `J3404` (the drug's HCPCS J-code).

> Note: despite the ticket framing, the title does **not** come from the
> `documents` collection. The PDF *content* in the panel comes from the
> `documents` split page; the *title* comes from the parent **intake**'s OCR
> data. See root cause below.

## Context (ids & collections)

Staging database: `local-infusion-db`.

| Item | Value |
|---|---|
| `intakes._id` | `6a57c04d85d689c573fc4baa` |
| `neworders._id` (order) | `6a58c06cb4ef88e69ceaad18` |
| `patients._id` (patient) | `6a1860e18c4205d33177d52f` |

Note: in the `documents` collection, `intakeId` and `patientId` are stored as
**strings** (not `ObjectId`), so query them without an `ObjectId()` wrapper.

## Investigation

### Query 1 — documents belonging to the intake

Find every `documents` row whose `intakeId` matches the intake, returning id +
category.

Mongo shell:

```js
db.documents.find(
  { intakeId: "6a57c04d85d689c573fc4baa" },
  { _id: 1, category: 1, blobPath: 1, linkedOrderIds: 1 }
)
```

Result — **2 documents**:

| `_id` | category | blobPath (filename) | linkedOrderIds |
|---|---|---|---|
| `6a58c244b4ef88e69ceab030` | **Prior Authorization** | `insurance-authorization_pages-2-2.pdf` | `[]` |
| `6a58c245b4ef88e69ceab061` | Other | `other_pages-1-1.pdf` | `[]` |

The document we care about (category **Prior Authorization**) is
`6a58c244b4ef88e69ceab030`.

### Query 2 — full data of the Prior Authorization document

Mongo shell:

```js
db.documents.findOne({ _id: ObjectId("6a58c244b4ef88e69ceab030") })
```

Result:

```json
{
  "_id": "6a58c244b4ef88e69ceab030",
  "intakeId": "6a57c04d85d689c573fc4baa",
  "patientId": "6a1860e18c4205d33177d52f",
  "category": "Prior Authorization",
  "description": "Humana approval for PAPZIMEOS with authorization dates and reference number",
  "blobPath": "https://listageapp.blob.core.windows.net/documents/intakes/2026/07/15/6a57c04d85d689c573fc4baa/insurance-authorization_pages-2-2.pdf",
  "pageCount": 1,
  "createdBy": "6a0d9d87e1e2dc4083b5ee00",
  "ocrStatus": "completed",
  "ocrData": { "labs": [] },
  "linkedOrderIds": [],
  "createdAt": "2026-07-16T11:36:36.387Z",
  "ocrCompletedAt": "2026-07-16T11:36:42.765Z",
  "ocrJsonPath": "intakes/2026/07/15/6a57c04d85d689c573fc4baa/insurance-authorization_pages-2-2_ocr.json",
  "ocrMarkdownPath": "intakes/2026/07/15/6a57c04d85d689c573fc4baa/insurance-authorization_pages-2-2_ocr.md",
  "__v": 0
}
```

#### Observation relevant to the title bug

This document has **no** `title` / `name` / `fileName` / `documentType` field.
The only human-readable strings are:

- `category`: `"Prior Authorization"`
- `description`: `"Humana approval for PAPZIMEOS with authorization dates and reference number"`
- `blobPath` filename: `insurance-authorization_pages-2-2.pdf`

So whatever the preview renders as the title must be derived from one of these
(or a lookup elsewhere). Next step: trace how the preview title is computed for
`documents`-collection objects and compare against the expected text.

### Query 3 — where the title string "J3404" comes from

The preview panel title is `previewLetter.drug` (see code trace below). For this
order-scoped route (`/api/orders/new/[orderId]/auth-review`), the letter is
built from the **`intakes`** collection via `transformIntakeToPriorAuthLetter`,
with all OCR fields nested under `additionalInfo`. The `drug` field resolves in
this precedence:

1. `additionalInfo.priorAuthData.drugName` (verified value) — **absent here**
2. `resolveOcrDrugName(additionalInfo.ocrData.service)` — used here
3. `''`

Mongo shell — inspect the OCR `service` block on the intake:

```js
db.intakes.findOne(
  { _id: ObjectId("6a57c04d85d689c573fc4baa") },
  {
    "additionalInfo.priorAuthData.drugName": 1,
    "additionalInfo.priorAuthData.liDrugId": 1,
    "additionalInfo.ocrData.service": 1
  }
)
```

Result:

```json
{
  "_id": "6a57c04d85d689c573fc4baa",
  "additionalInfo": {
    "ocrData": {
      "service": {
        "cptHcpcsCode": { "value": "J3404", "page": [2] },
        "serviceRequested": { "value": "PAPZIMEOS (J3404)", "page": [2] }
      }
    }
  }
}
```

Key facts:
- `priorAuthData.drugName` — **absent** (so precedence step 1 is skipped).
- `ocrData.service.brandDrugName` / `genericDrugName` — **absent**.
- The only drug text is `service.serviceRequested.value = "PAPZIMEOS (J3404)"`.

## Root cause

`drug` falls through to
`resolveOcrDrugName(service)` → `normalizeServiceDrugName("PAPZIMEOS (J3404)")`
(`apps/web/services/priorAuth/normalizeServiceDrugName.ts`).

That function has a **"prefer the parenthesized brand" heuristic**: when the
service line contains parentheses, it assumes the parentheses hold the most
specific brand name (designed for inputs like
`"IMMUNE GLOBULIN (PRIVIGEN) 10%"` → `"Privigen"`) and returns the normalized
contents of the parentheses if non-empty.

Here the parentheses hold the **HCPCS J-code**, not a brand:

```
normalizeServiceDrugName("PAPZIMEOS (J3404)")
  parenMatch -> "J3404"
  normalizeServiceDrugName("J3404")
    "J3404" is not a strength token (does not start with a digit),
    not a stopword, not a unit -> kept
    -> "J3404"
  fromParens = "J3404" (truthy) -> returns "J3404"   ← wrong
```

The real drug name `PAPZIMEOS` sits **outside** the parentheses and is discarded.
So the heuristic backfires whenever the parenthesized token is a procedure/HCPCS
code (`J####`) rather than a brand name.

### Code path summary

| Step | File | Detail |
|---|---|---|
| Title rendered | `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/AuthReviewIndex.tsx` | `previewLetter.drug \|\| 'Prior Auth Letter'` (panel header + `EmbeddedFileViewer fileName`) |
| Letter built | `apps/web/app/api/orders/new/[orderId]/auth-review/route.ts` | `Intake.aggregate` → `transformIntakeToPriorAuthLetter` |
| `drug` resolved | `apps/web/services/priorAuth/transforms.ts` | `resolveVerifiedFieldValue(priorAuthData?.drugName) \|\| resolveOcrDrugName(ocrData?.service) \|\| ''` |
| OCR drug name | `apps/web/services/priorAuth/resolveOcrDrugName.ts` | brand → generic → `normalizeServiceDrugName(serviceRequested)` |
| Bug location | `apps/web/services/priorAuth/normalizeServiceDrugName.ts` | parenthesized-brand preference extracts the `(J3404)` J-code |

## Conclusion

The title `J3404` is **not** stored on any document. It is derived at request
time from the parent intake's OCR field
`intakes.additionalInfo.ocrData.service.serviceRequested.value`
(`"PAPZIMEOS (J3404)"`), normalized by `normalizeServiceDrugName`, whose
"prefer the parenthesized brand" rule wrongly treats the parenthesized HCPCS
J-code `(J3404)` as the brand name and drops the real name `PAPZIMEOS`.

Object to inspect in Compass: **intake** `6a57c04d85d689c573fc4baa`, field
`additionalInfo.ocrData.service.serviceRequested`.
(The PDF blob shown in the panel is the `documents` split page
`6a58c244b4ef88e69ceab030`, but that is only the file content, not the title.)
