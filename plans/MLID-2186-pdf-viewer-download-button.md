# MLID-2186 — PDF Viewer Download Button

## Task Reference

- **Jira**: [MLID-2186](https://localinfusion.atlassian.net/browse/MLID-2186)
- **Story Points**: 4
- **Branch**: `feature/MLID-2186-pdf-viewer-download-button`
- **Base Branch**: `develop`
- **Status**: In Development
- **Branch workflow**: Branch off **local** `develop` after `git pull`. If `develop` moves during development: `git checkout develop && git pull`, then `git merge develop` into the feature branch. No rebase. Do not delete this feature branch after merge.

---

## Summary

LISA users have no way to download PDF documents from the in-app viewer. `ReactPdfRenderer` is the single PDF preview component used across all preview surfaces (Intakes, Patient Documents, Prior Authorizations, Clinical Review, Orders). Adding a Download button directly to its toolbar — using the existing browser anchor pattern already present in the codebase — delivers the capability on every surface with a single component change. A companion change in `EmbeddedFileViewer` forwards its existing `fileName` prop to both render sites so the downloaded file carries a meaningful name rather than a browser-generated blob identifier.

---

## Codebase Analysis

### Files affected

| File | Role |
|------|------|
| `apps/web/components/UI/FileViewer/ReactPdfRenderer.tsx` | Toolbar lives here (lines 287–346); new button added here |
| `apps/web/components/UI/FileViewer/ReactPdfRenderer.module.css` | Add `.downloadButton` class |
| `apps/web/components/UI/FileViewer/EmbeddedFileViewer.tsx` | Forwards `fileName` prop to both `ReactPdfRenderer` render sites |
| `apps/web/components/UI/FileViewer/ReactPdfRenderer.test.tsx` | Unit tests for the new button |
| `apps/web/components/UI/FileViewer/EmbeddedFileViewer.test.tsx` | Tests for `fileName` forwarding |

### Toolbar pattern

All existing toolbar controls in `ReactPdfRenderer` are plain HTML `<button>` elements styled via CSS module classes (not MUI `IconButton`). They each carry a `title` attribute (used by tests and screen readers) and an `aria-label`. Icons come from `@mui/icons-material/<Name>`. The Fullscreen button (lines 327–334) is the direct template for the new button.

### Blob URL flow

1. Caller passes Azure Blob URL as `fileUrl` to `EmbeddedFileViewer`.
2. `EmbeddedFileViewer` fetches it via `axios GET /api/files/stream?url=...` (`responseType: 'blob'`), wraps the response in `new Blob([data], { type: 'application/pdf' })`, calls `URL.createObjectURL(blob)`, and stores the result as `objectUrl`.
3. `objectUrl` (a `blob:` URL already resident in browser memory) is passed to `ReactPdfRenderer` as the `objectUrl` prop.

Because the blob is already materialized in the browser, **no additional fetch is required for download** — the anchor can point directly at `objectUrl`.

### Download implementation

Use the hand-rolled anchor pattern already present in the codebase (`downloadOcrFile` in `patientDocumentRecords.tsx`):

```typescript
const anchor = document.createElement('a');
anchor.href = objectUrl;
anchor.download = downloadFileName;
document.body.appendChild(anchor);
anchor.click();
document.body.removeChild(anchor);
```

No new dependencies (`file-saver` or otherwise).

### Filename normalization rule

- New prop `fileName?: string` on `ReactPdfRendererProps`.
- `downloadFileName` = `fileName` with `.pdf` appended if the name does not already end in `.pdf` (case-insensitive check).
- When `fileName` is `undefined` or empty, fall back to `'document.pdf'`.
- Several callers pass document-type strings such as `'Lab Result'`; these must receive the `.pdf` suffix.

### Disabled state

The Download button is disabled when `objectUrl === null`, matching the existing `!objectUrl` guard already used by the renderer at line 280.

### Call sites — no changes

All 11 callers already pass a useful `fileName` string or omit it (falling back to `'document.pdf'`). No caller modifications are required.

| Call site file | Passes `fileName`? |
|---|---|
| `apps/web/components/Modules/patient/components/patientDocumentRecords/patientDocumentRecords.tsx` | Yes |
| `apps/web/components/Modules/patient/components/patientPriorAuthorizations/patientPriorAuthorizations.tsx` | Yes |
| `apps/web/components/Modules/patient/components/patientClinicalReviews/patientClinicalReviews.tsx` | Yes |
| `apps/web/components/Modules/orders/components/orderDocuments/orderDocuments.tsx` | Yes |
| `apps/web/components/Modules/orders/components/orderPriorAuthorizations/orderPriorAuthorizations.tsx` | Yes |
| `apps/web/components/Modules/orders/components/orderClinicalReviews/orderClinicalReviews.tsx` | Yes |
| `apps/web/components/Modules/intakes/components/intakeDocuments/intakeDocuments.tsx` | Yes |
| `apps/web/app/patient/[id]/documents/page.tsx` (deprecated surface) | Yes / omitted |
| Other call sites (3) | Yes / omitted |

---

## Implementation Steps

Each step is one RED -> GREEN -> REFACTOR cycle and one commit.

---

### Step 1 — Add `fileName` prop and render a disabled Download button

**Files**
- `apps/web/components/UI/FileViewer/ReactPdfRenderer.test.tsx` (RED)
- `apps/web/components/UI/FileViewer/ReactPdfRenderer.tsx` (GREEN + REFACTOR)
- `apps/web/components/UI/FileViewer/ReactPdfRenderer.module.css` (GREEN)

**RED — write failing tests first**

In `ReactPdfRenderer.test.tsx`:

1. Add an icon mock for `DownloadIcon` at the top of the mock block:
   ```typescript
   jest.mock('@mui/icons-material/Download', () => ({
     __esModule: true,
     default: () => <span data-testid="download-icon" />,
   }));
   ```

2. Add two test cases (both will fail because the button does not exist yet):
   - `'should render a Download button in the toolbar'` — renders with `objectUrl` set; asserts `screen.getByTitle('Download')` is in the document.
   - `'should render the Download button as disabled when objectUrl is null'` — renders with `objectUrl={null}`; asserts `screen.getByTitle('Download')` has `disabled` attribute.

Run `npm run test -- ReactPdfRenderer.test.tsx`. Expected: RED (button not found).

**GREEN — minimal implementation**

In `ReactPdfRenderer.tsx`:
- Add `fileName?: string` to `ReactPdfRendererProps`.
- Import `DownloadIcon` from `@mui/icons-material/Download`.
- Insert a `<button>` after the Fullscreen button, patterned exactly on the Fullscreen button:
  ```tsx
  <button
    className={styles.downloadButton}
    title="Download"
    aria-label="Download"
    disabled={!objectUrl}
    onClick={handleDownload}
  >
    <DownloadIcon />
  </button>
  ```
- Add a stub `handleDownload` that does nothing yet (empty function body — sufficient to make Step 1 tests pass; Step 2 fills it in).

In `ReactPdfRenderer.module.css`:
- Add `.downloadButton` with the same rules as `.rotateButton` (inspect that class and duplicate; do not share a class between unrelated semantic buttons).

Run `npm run test -- ReactPdfRenderer.test.tsx`. Expected: GREEN.

**REFACTOR**

- If there are any magic string literals introduced, extract to named constants inside the file.
- Run `npm run types:check` and `npm run lint:fix` from `apps/web/`.

**Commit**: `[MLID-2186] - feat(pdf-viewer): add fileName prop and disabled Download button to ReactPdfRenderer toolbar`

---

### Step 2 — Wire click handler: anchor download with filename normalization

**Files**
- `apps/web/components/UI/FileViewer/ReactPdfRenderer.test.tsx` (RED)
- `apps/web/components/UI/FileViewer/ReactPdfRenderer.tsx` (GREEN + REFACTOR)

**RED — write failing tests first**

In `ReactPdfRenderer.test.tsx`, add a `describe('Download button click', ...)` block. Before each test, mock `document.createElement` for the `'a'` case:

```typescript
let fakeAnchor: { href: string; download: string; click: jest.Mock; remove: jest.Mock };

beforeEach(() => {
  fakeAnchor = { href: '', download: '', click: jest.fn(), remove: jest.fn() };
  const originalCreateElement = document.createElement.bind(document);
  jest.spyOn(document, 'createElement').mockImplementation((tag: string) => {
    if (tag === 'a') return fakeAnchor as unknown as HTMLElement;
    return originalCreateElement(tag);
  });
  jest.spyOn(document.body, 'appendChild').mockImplementation(() => fakeAnchor as unknown as Node);
  jest.spyOn(document.body, 'removeChild').mockImplementation(() => fakeAnchor as unknown as Node);
});

afterEach(() => {
  jest.restoreAllMocks();
});
```

Write three test cases (all RED until `handleDownload` is implemented):

1. `'should trigger anchor download with supplied fileName when objectUrl is set'`
   - Render with `objectUrl="blob:http://localhost/abc"` and `fileName="Lab Result"`.
   - `fireEvent.click(screen.getByTitle('Download'))`.
   - Assert `fakeAnchor.href === 'blob:http://localhost/abc'`.
   - Assert `fakeAnchor.download === 'Lab Result.pdf'` (suffix appended).
   - Assert `fakeAnchor.click` was called once.

2. `'should fall back to document.pdf when fileName is not provided'`
   - Render with `objectUrl="blob:http://localhost/abc"` and no `fileName`.
   - Click the button.
   - Assert `fakeAnchor.download === 'document.pdf'`.

3. `'should not double-append .pdf when fileName already ends in .pdf'`
   - Render with `objectUrl="blob:http://localhost/abc"` and `fileName="Summary.PDF"`.
   - Click the button.
   - Assert `fakeAnchor.download === 'Summary.PDF'` (unchanged, case preserved).

Run tests. Expected: RED (handler is a stub).

**GREEN — implement `handleDownload` and `normalizeDownloadFileName`**

In `ReactPdfRenderer.tsx`:

Add a pure helper function (module-level, not exported) named `normalizeDownloadFileName`:

```typescript
function normalizeDownloadFileName(fileName: string | undefined): string {
  if (!fileName) return 'document.pdf';
  return fileName.toLowerCase().endsWith('.pdf') ? fileName : `${fileName}.pdf`;
}
```

Implement `handleDownload` inside the component (or as a `useCallback` — keep it simple; no memoization unless justified):

```typescript
function handleDownload(): void {
  if (!objectUrl) return;
  const downloadFileName = normalizeDownloadFileName(fileName);
  const anchor = document.createElement('a');
  anchor.href = objectUrl;
  anchor.download = downloadFileName;
  document.body.appendChild(anchor);
  anchor.click();
  document.body.removeChild(anchor);
}
```

Run tests. Expected: GREEN.

**REFACTOR**

- Confirm `normalizeDownloadFileName` is tested via the three cases above. No additional extraction needed.
- Run `npm run types:check` and `npm run lint:fix` from `apps/web/`.

**Commit**: `[MLID-2186] - feat(pdf-viewer): implement download click handler with filename normalization`

---

### Step 3 — Forward `fileName` from `EmbeddedFileViewer` to both `ReactPdfRenderer` render sites

**Files**
- `apps/web/components/UI/FileViewer/EmbeddedFileViewer.test.tsx` (RED)
- `apps/web/components/UI/FileViewer/EmbeddedFileViewer.tsx` (GREEN + REFACTOR)

**RED — write failing tests first**

In `EmbeddedFileViewer.test.tsx`, locate the existing mock for `ReactPdfRenderer`. Extend the mock to capture the props it receives:

```typescript
let capturedRendererProps: Record<string, unknown> = {};

jest.mock('@/components/UI/FileViewer/ReactPdfRenderer', () => ({
  ReactPdfRenderer: (props: Record<string, unknown>) => {
    capturedRendererProps = props;
    return <div data-testid="react-pdf-renderer" />;
  },
}));
```

Add two test cases (both RED until the prop pass-through is added):

1. `'should forward fileName to ReactPdfRenderer in inline mode'`
   - Render `<EmbeddedFileViewer fileUrl="..." fileName="Lab Result" />` in a state where inline rendering is active (i.e., not in fullscreen dialog mode).
   - Assert `capturedRendererProps.fileName === 'Lab Result'`.

2. `'should forward fileName to ReactPdfRenderer in fullscreen dialog mode'`
   - Render `<EmbeddedFileViewer fileUrl="..." fileName="Lab Result" />`, then trigger the fullscreen open action (click the fullscreen button or set the appropriate state via the existing test pattern in the file).
   - Assert `capturedRendererProps.fileName === 'Lab Result'` in dialog mode.

Run `npm run test -- EmbeddedFileViewer.test.tsx`. Expected: RED (`fileName` not forwarded).

**GREEN — pass `fileName` to both `ReactPdfRenderer` render sites**

In `EmbeddedFileViewer.tsx`:
- Locate the inline `<ReactPdfRenderer ... />` (around line 335) and the Dialog/fullscreen `<ReactPdfRenderer ... />` (around line 362).
- Add `fileName={fileName}` to both.

No new prop additions are needed — `EmbeddedFileViewer` already accepts `fileName?: string`.

Run `npm run test -- EmbeddedFileViewer.test.tsx`. Expected: GREEN.

**REFACTOR**

- Verify no duplication was introduced beyond the two prop additions.
- Run `npm run types:check` and `npm run lint:fix` from `apps/web/`.

**Commit**: `[MLID-2186] - feat(pdf-viewer): forward fileName prop to ReactPdfRenderer in both inline and fullscreen modes`

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/components/UI/FileViewer/ReactPdfRenderer.tsx` | Modify | Add `fileName` prop, `DownloadIcon` button in toolbar, `handleDownload` function, `normalizeDownloadFileName` helper |
| `apps/web/components/UI/FileViewer/ReactPdfRenderer.module.css` | Modify | Add `.downloadButton` class matching the visual style of existing icon-only toolbar buttons |
| `apps/web/components/UI/FileViewer/EmbeddedFileViewer.tsx` | Modify | Add `fileName={fileName}` to both `ReactPdfRenderer` render sites (inline and fullscreen Dialog) |
| `apps/web/components/UI/FileViewer/ReactPdfRenderer.test.tsx` | Modify | Add `DownloadIcon` mock; add tests for button presence, disabled state, click handler, and filename normalization |
| `apps/web/components/UI/FileViewer/EmbeddedFileViewer.test.tsx` | Modify | Add tests asserting `fileName` is forwarded to `ReactPdfRenderer` in both inline and fullscreen modes |

---

## Testing Strategy

### Unit tests

**`ReactPdfRenderer.test.tsx`**

| Test case | Behavior |
|---|---|
| `'should render a Download button in the toolbar'` | Button with `title="Download"` appears when `objectUrl` is set |
| `'should render the Download button as disabled when objectUrl is null'` | Button has `disabled` attribute when `objectUrl={null}` |
| `'should trigger anchor download with supplied fileName when objectUrl is set'` | Anchor `href` equals `objectUrl`; `download` equals `fileName + '.pdf'`; `anchor.click` called once |
| `'should fall back to document.pdf when fileName is not provided'` | `anchor.download === 'document.pdf'` |
| `'should not double-append .pdf when fileName already ends in .pdf'` | `anchor.download` equals the original `fileName` unchanged |

**`EmbeddedFileViewer.test.tsx`**

| Test case | Behavior |
|---|---|
| `'should forward fileName to ReactPdfRenderer in inline mode'` | Captured `ReactPdfRenderer` props include `fileName` equal to the value passed to `EmbeddedFileViewer` |
| `'should forward fileName to ReactPdfRenderer in fullscreen dialog mode'` | Same assertion after fullscreen is opened |

### Manual verification (user must perform before the final commit)

1. Start the dev server (`npm run dev:web`).
2. Navigate to an orders-tracker order detail page that has at least one PDF document (e.g., `/orders-tracker/.../documents`).
3. Open a PDF document in the viewer.
4. Confirm the Download button appears in the toolbar to the right of the Fullscreen button.
5. Click the Download button.
6. Confirm the browser downloads the file and the saved filename ends in `.pdf`.
7. Open the fullscreen dialog for the same document and repeat steps 4–6 to verify both viewer modes work.
8. Optionally test from a patient documents surface and an intakes surface to confirm cross-surface coverage.

**Do not commit Step 3 until the user has completed manual verification.**

---

## Security Considerations

- **PHI in filenames**: The `fileName` prop originates from document metadata supplied by callers (document type strings, user-uploaded filenames). These values may include patient-facing descriptions. The download handler must not log the `downloadFileName` value at any level — no `logger.*` calls, no `console.*` calls inside `handleDownload`. The downloaded filename is between the browser and the user's file system only.
- **Sanitization**: Sanitizing the filename is the caller's responsibility. The renderer accepts whatever string is passed; it only appends the `.pdf` extension if missing. The anchor `download` attribute is set in the browser and does not reach the server.
- **Authorization**: No new API endpoints are introduced. The `objectUrl` is a `blob:` URL created from data already fetched by the authorized `EmbeddedFileViewer` session. No additional auth check is needed.
- **Input validation**: N/A — no user-controlled input enters the handler directly.
- **Feature flags**: No new feature flag required; the Download button is always visible. The surfaces where the viewer appears may individually be gated by existing flags — those are unaffected.

---

## Open Questions

None. All design decisions are resolved by the investigation summary and the constraints above.
