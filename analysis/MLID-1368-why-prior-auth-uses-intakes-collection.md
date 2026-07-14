# Why Prior Auth reads from the `intakes` collection (not `documents`)

Investigation into the original design decision behind the Prior Auth feature, to inform the MLID-2684 preview bug.

Source epics/tickets: MLID-1368 (Prior Auth tab epic), MLID-1005 (OCR for PA letters), and the epic's child tasks. Original developer: Egor Dezhic (foundational build tickets MLID-1590 and MLID-1591, implemented 2026-01-28; his Atlassian account now shows as "Former user").

## Short answer

Using `intakes` was **specified in the epic**, not an ad-hoc developer choice. The structured, human-reviewed prior-auth letter data only ever existed on the intake document, so there was no `documents`-collection alternative to read the letter fields from. The `documents` collection never held the OCR letter schema or the reviewer's edits.

## Evidence

### 1. The foundational ticket named intakes as the source

MLID-1590 ("Prior Auth Tab for the patients") — the first implementation ticket, built by Egor — specifies the data source directly:

- "Add a new `Prior Auth` tab after `Financial Calculator` in the patient's view (`/patient/{ID}/prior-auth`). This tab should contain the table of **intakes** with `authorization` among document definitions."
- "Possible values for `Payor` and `Drug` should come straight from available values in the DB because they can vary based on OCR results."
- "For `Letter Type` you should filter by the value of `letterMetadata.documentType` for one of the values `Approval | Denial | Request | Receipt | Reminder | Duplicate | Unnecessary | Other`."
- "In the table view when `letterMetadata.documentType` is `Other` you should display the `letterMetadata.documentTypeCustom` which will be produced by LLM arbitrarily."
- "Here is an example of `ocrData` for a doc with prior auth:" — followed by a JSON sample whose shape is `authorization` (approvedQuantity, authorizationNumber, dateOfDecision, effectiveDate, expirationDate, status), `letterMetadata`, `patient`, `payor`.

That `ocrData` / `authorization` / `letterMetadata` shape is the intake OCR payload (`intake.additionalInfo.ocrData`), not anything the `documents` collection stores.

### 2. The data pipeline puts the reviewed letter data on the intake

The upstream flow (from the OCR epic MLID-1005 and the removal-design ticket MLID-1715):

```
Spruce fax  ->  OCR classifies it as a PA letter  ->  VA confirms/corrects in the Intake Analyzer  ->  intake.additionalInfo.ocrData + priorAuthData
```

- MLID-1715: "We use OCR to identify Prior Auth related faxes, and then the virtual assistant (VA) users will further confirm or make correction in the **Intake Analyzer**." It also frames the "remove wrongly-identified PA letter" action as "another feedback loop to enhance our OCR model performance."
- MLID-1005 ("OCR for Prior Authorization Decision Letters v1", Done) defines the full structured field schema (Approval/Denial/Request-for-Info fields, letter metadata, payor, service/drug details) and says to "Store extracted data in structured fields in our database." These are the fields that land in `intake.additionalInfo.ocrData`.

Because both the machine extraction and the human correction happen at the intake/Intake Analyzer layer, the reviewed authorization data lives on the intake.

### 3. Edits are written back to intakes

MLID-1803 ("Dynamic field updates in Prior Auth table based on review"):

- "When a user updates specific fields while reviewing a document in the Prior Auth tab, the **Intakes table must reflect those updated values upon save/change**."

MLID-2364 ("Change Drug Name field to Dropdown…"):

- The Verify Data screen edits "the **OCR processed data**"; the field "may start with NULL if the OCR isn't able to process the drug name."

So the write target was intakes by design, matching the read source.

### 4. The `documents` collection was never the source of the letter fields

The `documents` collection (`apps/web/models/Document.ts`) stores per-file metadata: `intakeId`, `patientId`, `category`, `description`, `blobPath`, `pageCount`, OCR/embedding status, `linkedOrderIds`. Its `ocrData` sub-schema is lab-results only. It has no letter type, no `authorization`/`letterMetadata`/`payor` OCR shape, no review status, and none of the 19 verify-data fields. There was nothing to read the prior-auth letter data from on `documents`.

## Confirming the code matches the specification

The current implementation follows the tickets exactly:

- `apps/web/services/priorAuth/query.ts` — every `$match` path is on the intake: `patient.patientId`, `status: 'processed'`, `additionalInfo.ocrData.letterMetadata.documentType.value`, `additionalInfo.priorAuthData.*`. The pipeline is executed with `Intake.aggregate(...)`.
- `apps/web/services/priorAuth/transforms.ts` — `transformIntakeToPriorAuthLetter(intake: IIntake)` flattens the intake into the UI row (comment: "flattened from Intake model").
- `apps/web/services/priorAuth/detail.ts` — reads and writes `intake.additionalInfo.priorAuthData` via `Intake.findById` / `intake.save()`, then mirrors to Postgres with `mirrorIntakeToPostgres`.

## A related early symptom: MLID-1917

MLID-1917 ("Few auth documents failed to be added to Prior Auth tab"): "Patient WeInfuse ID 8529 has 8 Prior Authorization documents in the account, but Prior Auth tab contains only 6." This is early evidence that an intake-sourced view can under-count relative to the actual documents on the account — the same intakes-versus-documents seam the current preview bug (MLID-2684) sits on. Noted here as context, not as part of MLID-2684's scope.

## Implication for MLID-2684 (preview shows the whole fax)

Olha's business rule — show the parsed prior-auth document in the preview — does not conflict with the intakes architecture, because the letter data and the preview file come from different places:

| What | Correct source | Why |
| --- | --- | --- |
| Letter data + editable fields (type, verify data, review status, summary note) | `intakes` (`additionalInfo.ocrData` / `priorAuthData`) | Only place it exists; edits are written back here (MLID-1590 / 1803) |
| The preview PDF shown to the user | the parsed/split document blob | Currently wrongly pulled from `intake.documents[0]` (the whole fax) |

Therefore "show and edit from documents instead of intakes" is a false dichotomy:

- The edit **cannot** move off intakes without a large migration, because `documents` has none of the letter fields — and the epic confirms intakes is the system of record by design.
- The edit **does not need to** move. MLID-2684 is only that the preview blob pointer targets the whole fax instead of the parsed split document. The parsed document is already available both as a typed entry in `intake.documents[]` and as a row in the `documents` collection.

This supports keeping all data and edits on intakes and fixing only the preview file pointer (Option A in `docs/agomez/preplans/MLID-2684.md`).

## Ticket map

| Ticket | Type | Role in this decision |
| --- | --- | --- |
| MLID-1368 | Epic | Prior Auth tab epic; scope says table fields "come from the OCR processed data from the auth letters" |
| MLID-1005 | Epic | OCR for PA letters; defines the structured field schema stored on the intake |
| MLID-1590 | Task | Foundational build; names the "table of **intakes**" and the `ocrData`/`letterMetadata` shape |
| MLID-1591 | Task | Prior Auth letter review; behavior keyed to `letterMetadata.documentType` |
| MLID-1715 | Task | Remove wrongly-identified PA letter; describes OCR + Intake Analyzer confirmation flow |
| MLID-1803 | Story | Review edits must write back to the Intakes table |
| MLID-2364 | Task | Verify Data screen edits the OCR-processed data |
| MLID-1917 | Bug | Early evidence the intake-sourced view can under-count vs documents on the account |
