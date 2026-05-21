# Pre-existing failing tests on `develop`

**Date observed:** 2026-05-19
**Branch observed on:** `develop` at commit `3f645aa00` (also reproduced on `feature/MLID-2186-pdf-viewer-download-button` — confirmed identical)
**Discovered during:** `/verify` gate for [MLID-2186](https://localinfusion.atlassian.net/browse/MLID-2186) (PDF viewer Download button)
**Confirmed pre-existing:** yes — same failure counts and same root errors on bare `develop`

## Summary

| | Count |
|---|---|
| Total test suites in `apps/web` | 926 |
| Failing test suites | 12 |
| Failing tests | 118 |
| Passing tests | 15,627 |
| Skipped tests | 2 |
| Time to full run | ~100s (jest --no-coverage) |

## Failure groups

Failures cluster into **two root causes**:

- **Group A — `@repo/ui` theme not provided in test render (6 suites, ~110 tests).** Emotion-styled components from `@repo/ui` (`Card`, `CircularProgress`) read `t.colors.*` at render time, where `t` is the theme object injected by `LiThemeProvider`. When these components are rendered in a test without that provider wrapping, `t` is `undefined` and the styled function throws `TypeError: Cannot read properties of undefined (reading 'colors')`. All Group A suites fail with the same shape of error from `Card.styles.ts:47` or `CircularProgress.styles.ts:17`. Fixing this is either (a) wrap the test render helpers with `LiThemeProvider`, or (b) make the `@repo/ui` styled functions tolerate a missing theme.
- **Group B — independent failures unrelated to UI/theme (6 suites, ~12 tests).** Each suite has its own root cause: stale test mocks, API contract drift, missing Azure credentials, or a worker memory crash. Listed for completeness; none have anything to do with MLID-2186.

---

## Group A — `@repo/ui` theme provider missing in test render

### `app/orders-tracker/[category]/[orderId]/layout.test.tsx`

- **Failing tests (1):**
  - `OrderDetailsLayout › loads and renders order header metadata`
- **Root error:** `TestingLibraryElementError: Unable to find an element with the text: LISA Order` — caused upstream by a thrown render from a `@repo/ui` child component, which prevents the layout from completing its render. Console also shows `act(...)` warnings about `OrderDetailsLayout` state updates.
- **Impact on MLID-2186:** None — this layout mounts `EmbeddedFileViewer` only indirectly through child pages, and our `fileName` forwarding cannot trigger this failure.

### `app/orders-tracker/[category]/[orderId]/clinical-reviews/_components/__tests__/NewReviewPage.test.tsx`

- **Failing tests (4):**
  - `NewReviewPage › Preview Panel Visibility › should NOT show preview panel when drug is not selected`
  - `NewReviewPage › Preview Panel Visibility › should show preview panel after drug is selected`
  - `NewReviewPage › Referral Date Picker › should disable future dates on referral date picker`
  - `NewReviewPage › Form Layout › should render with full width form panel when drug is not selected`
- **Root error:** `TypeError: Cannot read properties of undefined (reading 'colors')` at `packages/ui/src/components/Card/Card.styles.ts:47` — `Card` styled function reads `t.colors.surface.muted` from an undefined theme.
- **Impact on MLID-2186:** This file is a downstream consumer of `EmbeddedFileViewer`. Failure is pre-existing and unrelated to the `fileName` forwarding change.

### `app/orders-tracker/[category]/[orderId]/clinical-reviews/_components/NewReviewPage.test.tsx`

- **Failing tests (9):**
  - `NewReviewPage - AI Integration › should search documents when the order drug is selected by context`
  - `NewReviewPage - AI Integration › should not rerun document search when selecting a document`
  - `NewReviewPage - AI Integration › should call createReview and analyzeReview when Run Review is clicked`
  - `NewReviewPage - AI Integration › should NOT show mock data when Run Review is clicked`
  - `NewReviewPage - AI Integration › should show error when createReview fails`
  - `NewReviewPage - AI Integration › Document Pre-selection › should pre-select Supporting Medical Documents alongside Lab Results`
  - `NewReviewPage - AI Integration › Document Pre-selection › should allow deselecting a pre-selected Supporting Medical Document`
  - `NewReviewPage - AI Integration › Document Selection - Bug Fixes › should independently select documents with same intakeId and documentType but different pdfUrls`
  - `NewReviewPage - AI Integration › Document Selection - Bug Fixes › should update PDF preview when clicking different documents with same intakeId and documentType`
- **Root error:** Same as above — `Card.styles.ts:47` reading `t.colors.surface.muted` on undefined theme.
- **Impact on MLID-2186:** Downstream consumer; pre-existing.

### `app/orders-tracker/[category]/[orderId]/clinical-reviews/_components/ClinicalReviewPage.test.tsx`

- **Failing tests (~45):** all tests in suite — examples:
  - `ClinicalReviewPage › should show loading state initially`
  - `ClinicalReviewPage › should display clinical review results when status is in_progress`
  - `ClinicalReviewPage › should display results when status is approved`
  - `ClinicalReviewPage › should show error when analysis failed`
  - `ClinicalReviewPage › should show analyzing status when analysis is in progress`
  - `ClinicalReviewPage › should show error when review not found`
  - `ClinicalReviewPage › Rejection Flow ›` *(5 tests)*
  - `ClinicalReviewPage › Complete Review Button ›` *(4 tests)*
  - `ClinicalReviewPage › Optional Notes Persistence ›` *(13 tests)*
  - `ClinicalReviewPage › Error Revert ›` *(3 tests)*
  - `ClinicalReviewPage › Polling Feedback Initialization ›` *(1 test)*
  - `ClinicalReviewPage › Debounced Note Submission ›` *(1 test)*
  - `ClinicalReviewPage › Value Requirement Fields Display ›` *(2 tests)*
  - `ClinicalReviewPage › Source Display in Result Column ›` *(5 tests)*
  - `ClinicalReviewPage › LISA Result Labels ›` *(2 tests)*
  - `ClinicalReviewPage › MLID-1272: Header Display ›` *(1 test)*
  - `ClinicalReviewPage › document dropdown metadata ›` *(4 tests)*
  - `ClinicalReviewPage › Source Link Toggle and Navigation ›` *(5 tests)*
  - `ClinicalReviewPage › MLID-1815: Infusion timeframe display in left panel ›` *(2 tests)*
- **Root error:** `TypeError: Cannot read properties of undefined (reading 'colors')` at `packages/ui/src/components/CircularProgress/CircularProgress.styles.ts:17` — `CircularProgress` styled function reads `t.colors.interactive.primary` on undefined theme.
- **Impact on MLID-2186:** Downstream consumer of `EmbeddedFileViewer`. Pre-existing.

### `app/orders-tracker/[category]/[orderId]/clinical-reviews/_components/components/ReviewHistoryList.test.tsx`

- **Failing tests (~25):** examples:
  - `ReviewHistoryList › Header › should render "Results history" title`
  - `ReviewHistoryList › Header › should render New Review button`
  - `ReviewHistoryList › Card Layout ›` *(9 tests)*
  - `ReviewHistoryList › Status Badges › should render status chips with correct text`
  - `ReviewHistoryList › Actions ›` *(4 tests)*
  - `ReviewHistoryList › Loading State › should show loading spinner when isLoading is true`
  - `ReviewHistoryList › Empty State › should show empty message when no reviews`
  - `ReviewHistoryList › Copy Functionality ›` *(6 tests)*
  - `ReviewHistoryList › Pagination ›` *(3 tests)*
- **Root error:** Same as above — `Card.styles.ts:47` on undefined theme.
- **Impact on MLID-2186:** Does NOT import `EmbeddedFileViewer`. Pre-existing, unrelated to this task.

### `app/orders-tracker/[category]/[orderId]/clinical-reviews/_components/components/DrugSelector.test.tsx`

- **Failing tests (1+):**
  - `DrugSelector › should show loading state when isLoading is true` — and additional tests in the suite
- **Root error:** `TestingLibraryElementError: Unable to find an element with the text: /loading orders/i` — likely caused upstream by the same theme issue preventing the component from rendering its loading state.
- **Impact on MLID-2186:** Does NOT import `EmbeddedFileViewer`. Pre-existing, unrelated.

---

## Group B — Independent failures

### `services/documents/documentSearchService.test.ts`

- **Failing tests (3):**
  - `documentSearchService › llmRelevanceSearchDocuments (search query) › should call categorySearchDocuments first, then evaluate with LLM`
  - `documentSearchService › llmRelevanceSearchDocuments (search query) › should exclude documents without ocrJsonPath from LLM evaluation`
  - `documentSearchService › llmRelevanceSearchDocuments (search query) › should skip documents whose markdown loading fails`
- **Root cause:** `mockEvaluateDocumentsRelevance` is being called with an extra positional argument `{limiter: undefined}` that the test does not expect. Looks like a recent change to `searchDocuments` (or its callee) added a new options object to the call, but the test expectations were not updated.
- **Failing function:** assertions on `mockEvaluateDocumentsRelevance.toHaveBeenCalledWith(...)` inside `searchDocuments` test setup (test file lines 235, 298, 362).
- **Impact on MLID-2186:** None — does not import any of the modified files.

### `services/jobs/pulse.test.ts`

- **Failing tests (1):**
  - `services/jobs/pulse › should create Pulse instance with correct db config`
- **Root cause:** `Pulse` constructor received an unexpected `defaultLockLifetime: 3600000` and `maxConcurrency: 100` (test expected `maxConcurrency: 20` and no `defaultLockLifetime`). Test asserts on the constructor args from `getPulse()` (pulse.ts).
- **Failing function:** `expect(Pulse).toHaveBeenCalledWith(objectContaining({...}))` at `services/jobs/pulse.test.ts:69`.
- **Impact on MLID-2186:** None.

### `app/api/clinical-reviews/[id]/__tests__/route.test.ts`

- **Failing tests (2):**
  - `GET /api/clinical-reviews/[id] › should return 404 if review is not found`
  - `GET /api/clinical-reviews/[id] › should return the clinical review for authenticated user`
- **Root cause:** Route handler is now returning HTTP 400 instead of 404 (not-found case) or 200 (success case). Validation or auth check upstream of the lookup logic is rejecting the request before it can run. Either the route handler logic or the test's mocked request shape is out of sync.
- **Failing function:** `expect(response.status).toBe(404)` / `.toBe(200)` at test file lines 83 and 119.
- **Impact on MLID-2186:** None.

### `services/clinicalReviews/analyzeClinicalReviewFromDocuments.test.ts`

- **Failing tests (9):**
  - `mapLLMResponseToClinicalReviewLabResults › should include lisaAnalysis with found, passed, and confidence`
  - `mapLLMResponseToClinicalReviewLabResults › should include sourceTestName in lisaAnalysis when provided`
  - `mapLLMResponseToClinicalReviewLabResults › should map sourceDocumentId when pageNumber matches source context`
  - `mapLLMResponseToClinicalReviewLabResults › should resolve sourceDocumentId to the correct document when pages overlap across documents`
  - `mapLLMResponseToClinicalReviewLabResults › sourceHighlights from sourceSubTests › should include polygon in sourceHighlights entry when findPolygonForText succeeds`
  - `mapLLMResponseToClinicalReviewLabResults › sourceHighlights from sourceSubTests › should cache OCR JSON to avoid redundant downloads for same document`
  - `mapLLMResponseToClinicalReviewLabResults - page validation › should treat hallucinated page number (not in source context) as not found`
  - `mapLLMResponseToClinicalReviewLabResults - page validation › should use localPage from source context (not global page) for sourcePageNumber`
  - `mapLLMResponseToClinicalReviewLabResults - page validation › should resolve sourceDocumentId and sourcePageNumber from source context`
- **Root cause:** `mapLLMResponseToClinicalReviewLabResults` is returning `lisaAnalysis` with `found: false`, `sourceTestName: undefined`, `sourceDocumentId: undefined`, `sourcePageNumber: undefined`. The fields that map source context (test name, document id, page number) and the polygon resolution are not being populated. Likely a regression in the mapping logic in `services/clinicalReviews/analyzeClinicalReviewFromDocuments.ts`.
- **Failing function:** `mapLLMResponseToClinicalReviewLabResults` in `analyzeClinicalReviewFromDocuments.ts` — its return value no longer matches what the test asserts on.
- **Impact on MLID-2186:** None.

### `services/azure/documentIntelligence.test.ts`

- **Failing tests (8):**
  - `extractDocumentMarkdown › should return DocumentOcrResult with markdown content on success`
  - `extractDocumentMarkdown › should throw error when analysis polling fails`
  - `extractDocumentMarkdown › should split markdown by page using page spans`
  - `extractDocumentMarkdown › should handle empty pages array gracefully`
  - `extractDocumentMarkdown › should throw error when analyzeResult is undefined`
  - `extractDocumentMarkdown with stream input › should use application/octet-stream content type when given a stream`
  - `extractDocumentMarkdown with stream input › should use urlSource when given a string URL`
  - `extractDocumentMarkdown with stream input › should return DocumentOcrResult on success with stream input`
- **Root cause:** Tests are hitting the real Azure Document Intelligence endpoint with invalid/unconfigured credentials: `401 Access denied due to invalid subscription key or wrong API endpoint`. The tests are not properly mocking the Azure client, OR they depend on a local environment variable set that's missing for me. This is an integration-style test masquerading as a unit test.
- **Failing function:** `extractDocumentMarkdown` in `services/azure/documentIntelligence.ts` line 116 throws because the Azure call returned 401.
- **Impact on MLID-2186:** None.

### `app/api/jobs/analytics-process-intake-overview/export/route.test.ts`

- **Failing tests:** `Test suite failed to run` — the suite could not initialize at all.
- **Root cause:** `Jest worker encountered 4 child process exceptions, exceeding retry limit` with a V8 crash trace (`v8::base::Thread::StartSynchronously`, `node::TriggerNodeReport`). Likely an out-of-memory or native binding issue while loading the test file's imports (often jsdom + heavy module graph).
- **Failing function:** N/A — process-level crash before any test runs.
- **Impact on MLID-2186:** None.

---

## Flakiness note

Two prior `/verify` runs against the same `develop` commit showed slightly different summaries:

- Run 1: 12 failed suites / **111** failed tests, included `services/apiKeys/generateKey.test.ts`
- Run 2: 12 failed suites / **118** failed tests, included `services/azure/documentIntelligence.test.ts` instead of `generateKey.test.ts`

The set of failing suites is fluid at the margins — `generateKey.test.ts` and `documentIntelligence.test.ts` appear to flake. Suite count is stable at 12.

---

## What this means for MLID-2186

- All 14 tests that import `ReactPdfRenderer` or `EmbeddedFileViewer` directly **pass** on the feature branch.
- The 3 failing suites that downstream-consume `EmbeddedFileViewer` (`ClinicalReviewPage`, `NewReviewPage` ×2) fail for theme-provider reasons that exist identically on `develop`. They are not regressions from MLID-2186.
- The remaining 9 failing suites have no path to be affected by adding a Download button to the PDF viewer.

A separate branch will be opened to address the Group A failures that touch our subtree (the orders-tracker clinical-reviews suites). The Group B failures are out of scope.
