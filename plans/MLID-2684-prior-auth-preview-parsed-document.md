# MLID-2684 — Prior Auth PDF preview shows the whole fax instead of the parsed document

## Task Reference

- **Jira**: [MLID-2684](https://localinfusion.atlassian.net/browse/MLID-2684)
- **Story Points**: 2
- **Branch**: `fix/MLID-2684-prior-auth-preview-parsed-document`
- **Base Branch**: `develop`
- **Status**: Done

---

## Summary

The Patient card Prior Auth tab and the Order Details Auth Review tab both preview the wrong PDF: they open the entire inbound fax instead of the split, parsed prior-authorization document that the Documents tab already shows. Both surfaces render through the same shared `transformIntakeToPriorAuthLetter` transform, which currently always points `documentUrl` at `intake.documents[0].pdfDocument` (the whole fax).

The fix is a plain presence check: **if the intake has a `documents` row with `category === "Prior Authorization"`, preview that row's `blobPath`; otherwise preview the whole intake PDF (from page 1).** No tiebreaker heuristics, no page-jump logic. This mirrors what the Documents tab already does and keeps a safe fallback for older/un-split intakes so nothing regresses.

---

## Codebase Analysis

- **Root cause** — `apps/web/services/priorAuth/transforms.ts`, `transformIntakeToPriorAuthLetter(intake: IIntake): PriorAuthLetter`, is pure/synchronous and builds `documentUrl` from `intake.documents?.[0]?.pdfDocument` (the whole fax). This transform is shared by both the patient prior-auth surface and the order-scoped auth-review surface, so a single fix here covers both.
- **`documentPage` is being dropped, not re-sourced** — today the transform also computes `documentPage` from `documentDefinitions.find(def => def.type === 'prior-authorization')`. Per the agreed design, the split prior-auth document is page 1, and the whole-intake fallback opens on page 1 as well, so `documentPage` is always undefined after this change and the entire `documentDefinitions`/page-extraction block is removed. Note this also removes a latent bug: production `additionalInfo.documentDefinitions[].type` for prior-auth is written as `'Insurance Authorization'`, never the literal `'prior-authorization'` the current code checks for, so the page-jump is already silently non-functional against real intakes today (masked only by test fixtures). Removing the block makes the page-1 behavior explicit instead of accidental. No reuse of `getPriorAuthPageList` / no change to `priorAuthReprocessDecision.ts` is needed.
- **`PriorAuthLetter` type** (`apps/web/types/priorAuth.ts`) already declares `documentUrl?: string` and `documentPage?: number` (both optional) — no type change needed. It also declares an already-present-but-unused `blobPath?: string`; this plan intentionally leaves `blobPath` unpopulated (out of scope — the UI only consumes `documentUrl`). The returned object simply stops setting `documentPage`.
- **Detail path** — `apps/web/services/priorAuth/detail.ts`, `getPriorAuthDetail(intakeId, patient)` already calls `getDocumentsByIntakeId(intakeId)` (returns ALL `documents` rows for the intake, unfiltered by category, sorted `createdAt: -1`) and today only reads `documents[0]?.description` for `documentDescription`. It then calls `transformIntakeToPriorAuthLetter(intake)`. Fix: pass the already-fetched `documents` array straight through as the transform's new second argument — the transform's own category filter (inside the new selection helper) does the narrowing, so `detail.ts` does not need to filter first. No new query in the detail path. `documentUrl`/`documentPage` on `PriorAuthDetailResponse` are read straight from `letter.documentUrl`/`letter.documentPage`, so no other change in `detail.ts`.
- **List routes** — `apps/web/app/api/patient/[patientId]/prior-auth/route.ts` and `apps/web/app/api/orders/new/[orderId]/auth-review/route.ts` are structurally identical for this purpose: both run `Intake.aggregate(buildPriorAuthListPipeline(...))` (pagination already applied inside `$facet.data` via `$skip`/`$limit`, so `intakes` is at most one page, ≤100 rows), then `intakes.map(intake => transformIntakeToPriorAuthLetter(intake))`. The batched documents lookup slots in right after `const intakes = result.data as IIntake[]` and before the `.map(...)` in both files — one extra query per page, not per row.
- **`IDocument`** (`apps/web/models/Document.ts`) has `intakeId: string`, `category?: string`, `blobPath: string` (required), `pageCount?: number`, `createdAt: Date`. Indexed on `intakeId` and `blobPath`; there is a composite `{intakeId, ocrStatus}` index but none on `{intakeId, category}` — not needed here since batches are ≤100 intake ids per page and per-intake document counts are small.
- **`getDocumentsByIntakeId`** (`apps/web/services/mongodb/document.ts`) is `(intakeId: string) => Promise<IDocument[]>`, does `DocumentModel.find({ intakeId }).sort({ createdAt: -1 }).lean().exec()`. It is reused as-is for the detail path; the list-route batched query is a new, separate function (different shape: `$in` over many intake ids, filtered by category at the query level).
- **Multiple matches (rare)** — the data has one known intake with two `category === 'Prior Authorization'` documents. The selection rule picks the **most recently created** match (`createdAt` descending, take first). The helper sorts internally so the result is deterministic regardless of the caller's input order.
- **Category mapping** — canonical persisted `Document.category` is `'Prior Authorization'` (via `mapDocumentTypeToWeInfuse`). A locked product decision (confirmed acceptable in the preplan) is to match `category === 'Prior Authorization'` only; the small number of rows literally mislabeled `'Insurance Authorization'` at the `Document.category` level will keep falling back to the whole fax. That is a separate, smaller data-cleanup concern, not this bug.
- **EmbeddedFileViewer** — `ReactPdfRenderer` defaults `pageNumber` to `1`; JS default parameters treat `undefined` and `1` identically. So passing `documentPage = undefined` (always, now) renders page 1 in every case with no special-casing and no UI changes. Consuming components (`PatientPriorAuth.tsx`, order `AuthReviewDetail.tsx`) pass `documentUrl`/`documentPage` straight through — out of scope for this fix.
- **Test conventions to follow**:
  - `apps/web/services/priorAuth/transforms.test.ts` — sibling file, `createMockIntake(overrides)` builder, `as unknown as IIntake` cast.
  - `apps/web/services/priorAuth/__tests__/detail.test.ts` — `__tests__` subfolder (this file's own existing convention), `jest.mock('@/models/Intake')`, factory-mocks `getDocumentsByIntakeId` and `logger`.
  - `apps/web/app/api/patient/[patientId]/prior-auth/route.test.ts` and `apps/web/app/api/orders/new/[orderId]/auth-review/route.test.ts` — mock `Intake.aggregate` directly, assert on the JSON response body.
  - Mongoose query chain mocks in this codebase mock each chained method (`find` → `{ lean: () => ({ exec: jest.fn().mockResolvedValue(...) }) }`), see `apps/web/services/mongodb/document.test.ts`.
  - `beforeEach(() => jest.clearAllMocks())` in every describe block; logger always mocked; no `any`.

---

## Implementation Steps

Each step follows RED (failing test) → GREEN (minimal implementation) → REFACTOR internally, and lands as one commit. Commit messages follow `[MLID-2684] - fix(prior-auth): <description>`.

### Step 1 — Pure selection helper: pick the split Prior Authorization document

- **Files**:
  - `apps/web/services/priorAuth/selectPriorAuthDocument.ts` (create)
  - `apps/web/services/priorAuth/selectPriorAuthDocument.test.ts` (create)
- **What**: Add one pure, synchronous function with no DB access:
  - `selectPriorAuthSplitDocument(documents: IDocument[] | undefined): string | undefined` — filters `documents` to `category === 'Prior Authorization'`. Returns `undefined` when there are zero matches (caller falls back to the whole fax). When there is one match, returns its `blobPath`. When there are multiple matches, returns the `blobPath` of the most recently created one (sort candidates by `createdAt` descending, take the first) — deterministic regardless of input order.
  - No `documentDefinitions` argument, no page logic, no dependency on `priorAuthReprocessDecision.ts`.
- **RED test cases** (`selectPriorAuthDocument.test.ts`):
  - "should return undefined when documents is undefined"
  - "should return undefined when documents is empty"
  - "should return undefined when no documents have category 'Prior Authorization'"
  - "should return the blobPath of the single matching document when exactly one 'Prior Authorization' document exists"
  - "should ignore documents of other categories (e.g. 'Demographics') when a 'Prior Authorization' document also exists"
  - "should return the blobPath of the most recently created document when multiple 'Prior Authorization' documents exist"
  - "should be deterministic regardless of input order when multiple matches exist" (same two docs, reversed input array → same blobPath returned)

### Step 2 — Wire the selection helper into `transformIntakeToPriorAuthLetter`

- **Files**:
  - `apps/web/services/priorAuth/transforms.ts` (modify)
  - `apps/web/services/priorAuth/transforms.test.ts` (modify)
- **What**:
  - Change the signature to `transformIntakeToPriorAuthLetter(intake: IIntake, documents: IDocument[]): PriorAuthLetter` — `documents` is a required second parameter (no default) so every call site must explicitly decide what to pass, preventing a call site from silently reverting to the whole-fax fallback by omission.
  - Remove the old `documentDefinitions` / `priorAuthDef` / `documentPage` extraction lines and the `pdfDocument` line.
  - Replace with: `const splitBlobPath = selectPriorAuthSplitDocument(documents);` then `const documentUrl = splitBlobPath ?? intake.documents?.[0]?.pdfDocument ?? undefined;`.
  - Remove the `documentPage` property from the returned object entirely (it is optional on `PriorAuthLetter` and is no longer set — the preview always opens on page 1).
  - Import `IDocument` type from `@/models/Document` and `selectPriorAuthSplitDocument` from `./selectPriorAuthDocument`.
- **Test file updates**:
  - Add a local test-only helper at the top of `transforms.test.ts`: `const transform = (intake: IIntake, documents: IDocument[] = []) => transformIntakeToPriorAuthLetter(intake, documents);` and replace all existing `transformIntakeToPriorAuthLetter(intake)` call sites with `transform(intake)` — this keeps every existing assertion about drug/payor/dates/letterType/reviewStatus unchanged (those tests pass `documents: []`, the correct no-hit-fallback case, identical to today's behavior).
  - Relabel the existing "should build documentUrl from pdfDocument" test to make the branch explicit: "should fall back to the whole-fax pdfDocument when no 'Prior Authorization' documents are provided" — same assertion, `transform(intake)` (i.e. `documents: []`).
  - **Delete** the two now-obsolete `documentPage` extraction tests ("should extract documentPage from prior-authorization definition" and "should extract documentPage to undefined when no prior-authorization definition exists") — that behavior is removed. Replace them with a single "should not set documentPage (always page 1)" test asserting `result.documentPage` is `undefined` in both the matched and fallback branches.
  - Add new RED test cases:
    - "should use the matched 'Prior Authorization' document's blobPath as documentUrl when one is provided"
    - "should ignore intake.documents[0].pdfDocument entirely when a split match exists" (assert the whole-fax URL is NOT returned)
    - "should return undefined documentUrl when there is neither a split match nor a whole-fax pdfDocument"

### Step 3 — Thread `documents` through `getPriorAuthDetail`

- **Files**:
  - `apps/web/services/priorAuth/detail.ts` (modify)
  - `apps/web/services/priorAuth/__tests__/detail.test.ts` (modify)
- **What**: In `getPriorAuthDetail`, change `const letter = transformIntakeToPriorAuthLetter(intake);` to `const letter = transformIntakeToPriorAuthLetter(intake, documents);`, reusing the `documents` array already fetched a few lines above via `getDocumentsByIntakeId(intakeId)`. No other change — `documentDescription` still reads `documents[0]?.description` from the same array as today.
- **RED test cases** (`detail.test.ts`, inside the existing `describe('getPriorAuthDetail', ...)` block):
  - "should use the 'Prior Authorization' document's blobPath as documentUrl when a matching Document record exists" — `mockGetDocumentsByIntakeId.mockResolvedValue([{ category: 'Prior Authorization', blobPath: 'https://blob/split-doc.pdf', createdAt: new Date('2024-01-16') }])`; assert `result.data.documentUrl === 'https://blob/split-doc.pdf'` and `result.data.documentPage` is `undefined`.
  - "should ignore Document rows for other categories and fall back to the whole-fax pdfDocument" — `mockGetDocumentsByIntakeId.mockResolvedValue([{ category: 'Demographics', blobPath: 'https://blob/demographics.pdf' }])`; assert `result.data.documentUrl` is the intake's whole-fax `pdfDocument` (`https://example.com/doc.pdf`), never the Demographics `blobPath`. (PHI/cross-contamination guard — see Security Considerations.)
  - "should pick the most recently created document when multiple 'Prior Authorization' Document records exist for the intake" — two rows with different `createdAt` and different `blobPath`; assert the newer row's `blobPath` wins.
  - The existing "should return found:true with detail data when intake belongs to patient" test already asserts `documentUrl` equals the whole-fax URL with the default `mockGetDocumentsByIntakeId.mockResolvedValue([])` from `beforeEach` — no change needed; it now exercises the no-hit fallback branch and continues to pass unmodified.

### Step 4 — Shared batched-fetch helper + wire both list routes

- **Files**:
  - `apps/web/services/priorAuth/documents.ts` (create)
  - `apps/web/services/priorAuth/documents.test.ts` (create)
  - `apps/web/app/api/patient/[patientId]/prior-auth/route.ts` (modify)
  - `apps/web/app/api/patient/[patientId]/prior-auth/route.test.ts` (modify)
  - `apps/web/app/api/orders/new/[orderId]/auth-review/route.ts` (modify)
  - `apps/web/app/api/orders/new/[orderId]/auth-review/route.test.ts` (modify)
- **What**:
  - Add `getPriorAuthDocumentsByIntakeIds(intakeIds: string[]): Promise<Map<string, IDocument[]>>` in `documents.ts`. Implementation: return an empty `Map` immediately when `intakeIds.length === 0` (no query). Otherwise run `DocumentModel.find({ intakeId: { $in: intakeIds }, category: 'Prior Authorization' }).lean().exec()` and group the results into a `Map<intakeId, IDocument[]>`. Wrap in try/catch: on failure, `logger.error(...)` and return an empty map — the query is a best-effort optimization; failure must degrade to the transform's whole-fax fallback for every intake on the page, not fail the entire list response.
  - In both route files, right after `const intakes = result.data as IIntake[];` (patient route) / `const intakes = (result?.data ?? []) as IIntake[];` (order route), add: `const documentsByIntakeId = await getPriorAuthDocumentsByIntakeIds(intakes.map((intake) => String(intake._id)));`. Inside the existing `.map(...)` callback, change `transformIntakeToPriorAuthLetter(intake)` to `transformIntakeToPriorAuthLetter(intake, documentsByIntakeId.get(String(intake._id)) ?? [])`. The `letter.letterMetadata.documentTypeByAi = ...` line directly below is unchanged.
- **RED test cases** (`documents.test.ts`):
  - "should return an empty map without querying the database when intakeIds is empty"
  - "should query DocumentModel.find with $in intakeIds and category 'Prior Authorization'"
  - "should group multiple documents under the same intakeId key"
  - "should return an empty map and log an error when the query rejects, without throwing"
- **RED test cases** (both route test files — mirror in each):
  - Add `jest.mock('@/services/priorAuth/documents', () => ({ getPriorAuthDocumentsByIntakeIds: jest.fn() }))` and a `mockGetPriorAuthDocumentsByIntakeIds` reference; default it to resolve an empty `Map` in `beforeEach`.
  - "should call getPriorAuthDocumentsByIntakeIds with the intake ids returned by the aggregation"
  - "should use the matched 'Prior Authorization' document's blobPath as documentUrl for a letter when the batched lookup returns a match for its intake id"
  - "should fall back to the whole-fax pdfDocument for a letter when the batched lookup has no entry for its intake id"

### Step 5 — Verification pass

- **Files**: none (no source changes)
- **What**: Run the full targeted test suite and type-check for every file touched above:
  - `npm run test -- --testPathPattern="priorAuth|prior-auth"` (from `apps/web/`)
  - `npm run types:check` (from `apps/web/`)
  - `npm run lint:fix` (from `apps/web/`) and confirm no new warnings on touched files
  - Confirm coverage on modified files with existing test files stays at or above the repo's 80% threshold
  - No commit for this step unless verification surfaces a fix — in that case it becomes its own small commit.

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/services/priorAuth/selectPriorAuthDocument.ts` | Create | Pure helper: filter to `category === 'Prior Authorization'`, return most-recent match's `blobPath` |
| `apps/web/services/priorAuth/selectPriorAuthDocument.test.ts` | Create | Unit tests for the selection helper |
| `apps/web/services/priorAuth/transforms.ts` | Modify | `transformIntakeToPriorAuthLetter` takes `documents` as a required second argument, uses the helper for `documentUrl`, and stops setting `documentPage` |
| `apps/web/services/priorAuth/transforms.test.ts` | Modify | Add join/fallback cases; delete obsolete `documentPage` extraction tests; update call sites to pass `documents` |
| `apps/web/services/priorAuth/detail.ts` | Modify | Pass the already-fetched `documents` array into `transformIntakeToPriorAuthLetter` |
| `apps/web/services/priorAuth/__tests__/detail.test.ts` | Modify | Add matched/fallback/other-category/most-recent test cases |
| `apps/web/services/priorAuth/documents.ts` | Create | Batched `$in` + category-filtered fetch, grouped by intakeId, for list routes |
| `apps/web/services/priorAuth/documents.test.ts` | Create | Unit tests for the batched fetch helper |
| `apps/web/app/api/patient/[patientId]/prior-auth/route.ts` | Modify | Wire the batched helper before mapping intakes to letters |
| `apps/web/app/api/patient/[patientId]/prior-auth/route.test.ts` | Modify | Mock the batched helper; assert documentUrl resolution |
| `apps/web/app/api/orders/new/[orderId]/auth-review/route.ts` | Modify | Wire the batched helper before mapping intakes to letters |
| `apps/web/app/api/orders/new/[orderId]/auth-review/route.test.ts` | Modify | Mock the batched helper; assert documentUrl resolution |

---

## Testing Strategy

- **Unit tests**:
  - `selectPriorAuthDocument.test.ts` — no match, single match, other-category ignored, most-recent selection among multiple, order-independence.
  - `transforms.test.ts` — matched-document path sets `documentUrl` to the split `blobPath` and does not leak the whole-fax URL; no-hit fallback path returns the whole-fax URL; `documentPage` is always `undefined`; all pre-existing behavior assertions (drug, payor, dates, letterType, reviewStatus) continue to pass with `documents: []`.
  - `documents.test.ts` — empty-input short-circuit, correct Mongoose query shape (`$in` + category filter), grouping by intakeId, and non-throwing failure degradation.
- **Integration tests**:
  - `__tests__/detail.test.ts` — full `getPriorAuthDetail` flow with a mocked `getDocumentsByIntakeId`, covering matched-category, other-category (ignored), and multiple-match (most-recent) scenarios, plus the existing not_found/ownership_error/isManualUpload cases (unchanged).
  - Both list-route `route.test.ts` files — mocked `Intake.aggregate` plus mocked `getPriorAuthDocumentsByIntakeIds`, asserting the batched helper is invoked with the intake ids from the aggregation page and that each letter's `documentUrl` reflects either the matched blobPath or the whole-fax fallback.
- **Manual verification** (after the dev server is running and connected to a database with real intakes that have both whole-fax `intake.documents` and split `documents` rows categorized `Prior Authorization`):
  - Patient card → Prior Auth tab → Preview icon → confirm only the parsed pages render, matching the Documents tab.
  - Order Details → Documents/Auth Review → open the same auth doc → confirm the same parsed-only preview.
  - Open an older/un-split intake (no matching `documents` row) on both surfaces → confirm the whole intake PDF still previews (from page 1), no regression.

---

## Security Considerations

- **PHI / cross-patient leakage**: the join is always scoped to documents belonging to the same intake(s) already resolved for that patient/order — the detail path filters by a single `intakeId` (already ownership-checked via `intakeBelongsToPatient`), and the list-route batched query's `$in` list is built exclusively from the intake ids already returned by that same patient/order-scoped aggregation page. No new patient- or intake-crossing query is introduced.
- **Category-scoped guard**: the selection helper filters to `category === 'Prior Authorization'` before ever considering a `blobPath`, so a `Demographics` or `Insurance Card` document row belonging to the same intake can never be surfaced as the prior-auth preview (covered explicitly by the "should ignore Document rows for other categories" test in Step 3).
- **Fail-safe degradation**: if the batched list-route query throws (e.g. transient DB error), `getPriorAuthDocumentsByIntakeIds` logs and returns an empty map rather than propagating — every letter on that page then resolves via the existing whole-fax fallback, so a lookup failure degrades to today's (already-shipped) behavior instead of breaking the page or exposing a different document.
- **Feature flags**: no new feature flag is introduced. The order-scoped route remains gated by its existing flag; this fix does not change that gate.
- **Input validation**: no new user-controlled input is introduced — `intakeIds` passed to the batched query are server-derived (from the already-validated aggregation result), not request parameters.

---

## Open Questions

- **Resolved — fallback page**: when there is no `Prior Authorization` document, the whole intake PDF previews from page 1. `documentPage` is dropped entirely (always undefined); `ReactPdfRenderer` renders page 1 with no special-casing or UI change.
- **One product decision to reconfirm before merge**: rows where `Document.category` is literally `'Insurance Authorization'` (a data mislabel, distinct from any field on the Intake) will not match `category === 'Prior Authorization'` and will therefore keep showing the whole-intake fallback rather than the parsed document. This was locked in the preplan as acceptable scope for this bug fix (a separate data-cleanup effort would relabel those rows), but is worth a final confirmation with product/QA before merge since it means the fix will not cover 100% of intakes on day one.
- None other.
</content>
</invoke>
