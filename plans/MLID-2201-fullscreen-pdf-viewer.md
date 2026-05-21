# MLID-2201 — Fullscreen PDF Viewer

## Task Reference

- **Jira**: [MLID-2201](https://localinfusion.atlassian.net/browse/MLID-2201)
- **Story Points**: 2
- **Branch**: `feature/MLID-2201-fullscreen-pdf-viewer`
- **Base Branch**: `develop`
- **Status**: Done

> Note: A PoC branch `feature/MLID-2201-fullscreen-pdf-poc` (commit `fdfd5388c`) exists. It is authoritative for the design pattern only. The real implementation starts from a fresh branch off `develop` with TDD. The PoC has no tests and is not to be merged.

---

## Summary

This task adds a fullscreen toggle to the embedded PDF viewer so users can read documents without being constrained by the layout's column width. The design follows a callback prop + lifted state pattern: `ReactPdfRenderer` receives an optional `onRequestFullscreen` prop and renders a toolbar icon button only when that prop is defined; `EmbeddedFileViewer` owns `isFullscreen` state and a sibling `@repo/ui` Dialog containing a second `ReactPdfRenderer` instance (no fullscreen prop, so its toolbar is clean). The same blob URL is reused between both instances — no second network fetch. Because `EmbeddedFileViewer` is already consumed by approximately nine call sites (intakes, prior auth, clinical reviews, document-intake wizard steps, and the document-records page), the feature propagates to all of them automatically with no call-site changes required.

---

## Codebase Analysis

### Files to touch (four total)

| File | Role |
|------|------|
| `apps/web/components/UI/FileViewer/ReactPdfRenderer.tsx` | Add `onRequestFullscreen` prop + conditional toolbar button |
| `apps/web/components/UI/FileViewer/ReactPdfRenderer.test.tsx` | Extend with fullscreen button tests + `OpenInFullIcon` mock |
| `apps/web/components/UI/FileViewer/EmbeddedFileViewer.tsx` | Add `isFullscreen` state + sibling Dialog + second renderer |
| `apps/web/components/UI/FileViewer/EmbeddedFileViewer.test.tsx` | Update dynamic mock + add Dialog open/close flow tests |

A third test file, `apps/web/components/UI/FileViewer/__tests__/EmbeddedFileViewer.test.tsx` (84 lines, tests an ErrorBoundary crash path), is untouched.

### ReactPdfRenderer.tsx — insertion details

- Current props interface is at lines 16–28. Add `onRequestFullscreen?: () => void` as a new optional field.
- Toolbar structure: `.toolbarLeft` contains `.zoomControls` followed by `.rotationControls` (which holds the rotate button and the conditional Save button). The new fullscreen button goes **after** the `rotationControls` block, still inside `.toolbarLeft`, so it sits visually next to Save.
- Existing CSS class to reuse: `styles.toolbarButton` — no new CSS needed.
- Icon import to add: `import OpenInFullIcon from '@mui/icons-material/OpenInFull';` — currently zero usages anywhere in the repo. Follow the exact same import style as the two existing icon imports on lines 3–4.
- Render pattern: `{onRequestFullscreen !== undefined && (<button className={styles.toolbarButton} title="Open in fullscreen" aria-label="Open in fullscreen" onClick={onRequestFullscreen}><OpenInFullIcon /></button>)}`

### EmbeddedFileViewer.tsx — insertion details

- Current props interface is at lines 19–37. No new props are added to the public interface; the feature is enabled unconditionally for PDF mode.
- Add `const [isFullscreen, setIsFullscreen] = useState<boolean>(false);` alongside existing state declarations.
- The existing inline `<ReactPdfRenderer>` (lines 297–319) receives one new prop: `onRequestFullscreen={() => setIsFullscreen(true)}`.
- Immediately after the existing `pdfContainer` div (still inside the `fileType === 'pdf' && !legacyPdfViewer` guard), add a sibling `<Dialog>` from `@repo/ui`:
  - `open={isFullscreen}`
  - `onClose={() => setIsFullscreen(false)}`
  - `title={fileName}` (defaults to `'Document'` — no change needed)
  - `size="lg"`
  - `maxWidth="xl"`
  - `fixedHeight={96}`
  - Children: an `<ErrorBoundary>` (matching the inline path) wrapping a second `<ReactPdfRenderer>` with `objectUrl`, `pageNumber`, `initialZoom={pdfInitialZoom}`, `fileUrl`, `onRotationSaved={handleRetry}` — and **no** `onRequestFullscreen` prop, so the fullscreen button is automatically hidden inside the dialog.
- Add `import { Dialog } from '@repo/ui';` at the top. No existing `@repo/ui` import is present in this file.

### Dialog import — design-system vs. legacy distinction

- Use `import { Dialog } from '@repo/ui';` — this is the design-system Dialog from `packages/ui`.
- Do **not** use the legacy `@/components/UI/Dialog` component. This is a hard constraint from the Jira ticket.
- The `@repo/ui` Dialog renders `role="dialog"` and has a built-in close (X) button and green title bar via the `title` prop.

### Existing test files — extend, do not create

- `ReactPdfRenderer.test.tsx` (699 lines): Add `jest.mock('@mui/icons-material/OpenInFull', ...)` alongside the existing Rotate90DegreesCw and Save mocks. Append a new `describe` block at the end.
- `EmbeddedFileViewer.test.tsx` (1118 lines): Update the `next/dynamic` mock so the stubbed `ReactPdfRenderer` exposes the `onRequestFullscreen` prop and makes it callable from the test (see Step 2 detail). Append a new `describe` block at the end.

### PoC commit as design reference

Commit `fdfd5388c` on `feature/MLID-2201-fullscreen-pdf-poc` confirms: the Dialog-inside-Dialog scenario on `/patient/[patientId]/document-records` works via MUI's portal system — the new dialog opens above the legacy `FileViewer` Dialog without breakage. This was the primary risk validated by the PoC.

### No feature flag

The feature is unconditional. No feature flag is added. No env-var override is needed.

---

## Implementation Steps

### Step 1 — TDD: ReactPdfRenderer fullscreen toolbar button

**Files**:
- `apps/web/components/UI/FileViewer/ReactPdfRenderer.test.tsx` (RED then GREEN)
- `apps/web/components/UI/FileViewer/ReactPdfRenderer.tsx` (GREEN then REFACTOR)

#### RED — Write failing tests first

Add the following at the very top of the mock block in `ReactPdfRenderer.test.tsx`, alongside the existing icon mocks for `Rotate90DegreesCw` and `SaveIcon`:

```ts
jest.mock('@mui/icons-material/OpenInFull', () => ({
  __esModule: true,
  default: () => <span data-testid="open-in-full-icon">OpenInFull</span>,
}));
```

Append a new `describe` block at the end of `ReactPdfRenderer.test.tsx`:

```ts
describe('fullscreen button', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  it('should not render the fullscreen button when onRequestFullscreen is undefined', () => {
    render(<ReactPdfRenderer objectUrl="blob:test" />);

    expect(screen.queryByTitle('Open in fullscreen')).not.toBeInTheDocument();
    expect(screen.queryByTestId('open-in-full-icon')).not.toBeInTheDocument();
  });

  it('should render the fullscreen button when onRequestFullscreen is defined', () => {
    const handleFullscreen = jest.fn();

    render(<ReactPdfRenderer objectUrl="blob:test" onRequestFullscreen={handleFullscreen} />);

    expect(screen.getByTitle('Open in fullscreen')).toBeInTheDocument();
    expect(screen.getByTestId('open-in-full-icon')).toBeInTheDocument();
  });

  it('should invoke onRequestFullscreen exactly once when the fullscreen button is clicked', () => {
    const handleFullscreen = jest.fn();

    render(<ReactPdfRenderer objectUrl="blob:test" onRequestFullscreen={handleFullscreen} />);
    fireEvent.click(screen.getByTitle('Open in fullscreen'));

    expect(handleFullscreen).toHaveBeenCalledTimes(1);
  });
});
```

Run: `npm run test -- ReactPdfRenderer.test`
Expected: all three new tests fail — `ReactPdfRenderer` does not accept `onRequestFullscreen` yet.

#### GREEN — Minimal implementation

In `ReactPdfRenderer.tsx`:

1. Add import: `import OpenInFullIcon from '@mui/icons-material/OpenInFull';`
2. Add `onRequestFullscreen?: () => void;` to the `ReactPdfRendererProps` interface.
3. Destructure `onRequestFullscreen` from props.
4. In the toolbar JSX, after the `rotationControls` block inside `.toolbarLeft`, add:

```tsx
{onRequestFullscreen !== undefined && (
  <button
    className={styles.toolbarButton}
    title="Open in fullscreen"
    aria-label="Open in fullscreen"
    onClick={onRequestFullscreen}
  >
    <OpenInFullIcon />
  </button>
)}
```

Run: `npm run test -- ReactPdfRenderer.test`
Expected: all three new tests pass.

#### REFACTOR — Polish and quality gates

- Confirm the button sits visually after `rotationControls` and before `.pageInfo` in the DOM — consistent with the mockup placement next to Save.
- Confirm `aria-label` and `title` both read `'Open in fullscreen'` (accessible and test-queryable).
- No new CSS classes introduced; `styles.toolbarButton` is reused.
- Run: `npm run lint:fix` and `npm run types:check` from `apps/web/`.

**Commit:**
```
[MLID-2201] - feat(file-viewer): add fullscreen toggle button to PDF toolbar

Adds an optional onRequestFullscreen callback prop to ReactPdfRenderer.
When defined, an OpenInFullIcon button renders after the rotation controls
in toolbarLeft, reusing the existing toolbarButton CSS class.
When undefined (default), the button is entirely absent from the DOM.
```

---

### Step 2 — TDD: EmbeddedFileViewer fullscreen Dialog

**Files**:
- `apps/web/components/UI/FileViewer/EmbeddedFileViewer.test.tsx` (RED then GREEN)
- `apps/web/components/UI/FileViewer/EmbeddedFileViewer.tsx` (GREEN then REFACTOR)

#### RED — Write failing tests first

**Update the `next/dynamic` mock** so the stubbed `ReactPdfRenderer` captures and exposes the `onRequestFullscreen` prop. The mock must render a trigger button that the test can click to simulate the callback being fired. Replace (or extend) the existing dynamic mock to match this pattern:

```ts
jest.mock('next/dynamic', () => () => {
  const MockReactPdfRenderer = (props: {
    objectUrl?: string | null;
    onRequestFullscreen?: () => void;
    [key: string]: unknown;
  }) => (
    <div data-testid="react-pdf-renderer" data-object-url={props.objectUrl ?? ''}>
      {props.onRequestFullscreen !== undefined && (
        <button
          data-testid="trigger-fullscreen"
          onClick={props.onRequestFullscreen}
        >
          Trigger fullscreen
        </button>
      )}
    </div>
  );
  MockReactPdfRenderer.displayName = 'MockReactPdfRenderer';
  return MockReactPdfRenderer;
});
```

Append a new `describe` block at the end of `EmbeddedFileViewer.test.tsx`:

```ts
describe('fullscreen Dialog', () => {
  const defaultProps = {
    fileUrl: 'https://example.com/test.pdf',
    fileName: 'TestDocument.pdf',
    fileType: 'pdf' as const,
  };

  beforeEach(() => {
    jest.clearAllMocks();
    // Mock axios to return a blob so objectUrl is populated
    // (follow the existing axios mock pattern already in this file)
  });

  it('should pass onRequestFullscreen to the inline ReactPdfRenderer', async () => {
    render(<EmbeddedFileViewer {...defaultProps} />);

    // Wait for objectUrl to resolve (follow existing waitFor pattern in this file)
    await waitFor(() => {
      expect(screen.getByTestId('trigger-fullscreen')).toBeInTheDocument();
    });
  });

  it('should not show a Dialog before the fullscreen callback fires', async () => {
    render(<EmbeddedFileViewer {...defaultProps} />);

    await waitFor(() => {
      expect(screen.getByTestId('trigger-fullscreen')).toBeInTheDocument();
    });

    expect(screen.queryByRole('dialog')).not.toBeInTheDocument();
  });

  it('should render a Dialog containing a second ReactPdfRenderer when fullscreen is triggered', async () => {
    render(<EmbeddedFileViewer {...defaultProps} />);

    await waitFor(() => {
      expect(screen.getByTestId('trigger-fullscreen')).toBeInTheDocument();
    });

    fireEvent.click(screen.getByTestId('trigger-fullscreen'));

    // Dialog opens
    expect(screen.getByRole('dialog')).toBeInTheDocument();

    // Two renderer stubs now in the DOM (inline + dialog)
    expect(screen.getAllByTestId('react-pdf-renderer')).toHaveLength(2);
  });

  it('should hide the Dialog when onClose is called', async () => {
    render(<EmbeddedFileViewer {...defaultProps} />);

    await waitFor(() => {
      expect(screen.getByTestId('trigger-fullscreen')).toBeInTheDocument();
    });

    fireEvent.click(screen.getByTestId('trigger-fullscreen'));
    expect(screen.getByRole('dialog')).toBeInTheDocument();

    // The @repo/ui Dialog renders a close button with aria-label="close"
    fireEvent.click(screen.getByRole('button', { name: /close/i }));

    await waitFor(() => {
      expect(screen.queryByRole('dialog')).not.toBeInTheDocument();
    });
  });

  it('should not pass onRequestFullscreen to the fullscreen Dialog renderer', async () => {
    render(<EmbeddedFileViewer {...defaultProps} />);

    await waitFor(() => {
      expect(screen.getByTestId('trigger-fullscreen')).toBeInTheDocument();
    });

    fireEvent.click(screen.getByTestId('trigger-fullscreen'));

    // Only one "trigger-fullscreen" button exists — the dialog renderer has no callback
    expect(screen.getAllByTestId('trigger-fullscreen')).toHaveLength(1);
  });
});
```

> Note: If `@repo/ui` Dialog requires `LiThemeProvider` to render correctly (manifests as missing `role="dialog"` or style errors in the test output), wrap renders in a `renderWithTheme` helper. Check whether the existing test file already imports or defines such a helper before adding a new one. The pattern used in `apps/web/components/Modules/orders-tracker/switch-context/DrugSwitchContextNoteDialog.test.tsx` is the reference.

Run: `npm run test -- EmbeddedFileViewer.test`
Expected: the five new tests fail — `EmbeddedFileViewer` does not yet have a Dialog or the `onRequestFullscreen` wiring.

#### GREEN — Minimal implementation

In `EmbeddedFileViewer.tsx`:

1. Add import: `import { Dialog } from '@repo/ui';`
2. Add state: `const [isFullscreen, setIsFullscreen] = useState<boolean>(false);`
3. On the existing inline `<ReactPdfRenderer>` inside the `pdfContainer` div, add the prop: `onRequestFullscreen={() => setIsFullscreen(true)}`
4. Immediately after the closing `</div>` of the `pdfContainer` block (still inside the `fileType === 'pdf' && !legacyPdfViewer` guard), add:

```tsx
<Dialog
  open={isFullscreen}
  onClose={() => setIsFullscreen(false)}
  title={fileName}
  size="lg"
  maxWidth="xl"
  fixedHeight={96}
>
  <ErrorBoundary key={errorBoundaryKey} onReset={handleErrorBoundaryReset}>
    <ReactPdfRenderer
      objectUrl={objectUrl}
      pageNumber={pageNumber}
      initialZoom={pdfInitialZoom}
      fileUrl={fileUrl}
      onRotationSaved={handleRetry}
    />
  </ErrorBoundary>
</Dialog>
```

> `ErrorBoundary` and `handleErrorBoundaryReset` are already defined in this component — reuse them exactly as in the inline path. `fileName` defaults to `'Document'` per the existing default prop, so no null guard is needed for the Dialog `title`.

Run: `npm run test -- EmbeddedFileViewer.test`
Expected: all five new tests pass.

#### REFACTOR — Polish and quality gates

- Confirm the Dialog `ReactPdfRenderer` instance forwards the same props as the inline instance (minus `onRequestFullscreen`): `objectUrl`, `pageNumber`, `initialZoom={pdfInitialZoom}`, `fileUrl`, `onRotationSaved={handleRetry}`.
- Confirm `objectUrl` is the existing state variable populated by the blob fetch — no new fetch is triggered. This is the key correctness invariant: one blob fetch, two renderer instances sharing the same URL.
- Confirm the `isFullscreen` state resets to `false` on Dialog close via `onClose`.
- Confirm the Dialog is placed **outside** the `pdfContainer` div but inside the `fileType === 'pdf' && !legacyPdfViewer` guard, so it does not render when showing images or using the legacy viewer.
- Run: `npm run lint:fix` and `npm run types:check` from `apps/web/`.

**Commit:**
```
[MLID-2201] - feat(file-viewer): open fullscreen Dialog on PDF toolbar button click

Adds isFullscreen state to EmbeddedFileViewer. The inline ReactPdfRenderer
now receives an onRequestFullscreen callback that sets isFullscreen=true.
A sibling @repo/ui Dialog (size="lg", maxWidth="xl", fixedHeight=96) renders
a second ReactPdfRenderer instance sharing the same objectUrl — no second
blob fetch. The dialog instance receives no onRequestFullscreen prop so its
toolbar renders without a fullscreen button. The @repo/ui Dialog's built-in
close button sets isFullscreen=false.
```

---

### Step 3 — Quality gates and visual verification

**No new commit unless fixes are required.**

#### Automated checks

Run all from `apps/web/`:

```bash
npm run test -- FileViewer
npm run types:check
npm run lint
npm run format
```

Coverage check: both changed `.tsx` files must remain at or above 80% line/branch coverage. If coverage drops below the threshold, add targeted tests before proceeding.

#### Manual visual verification (browser)

Start the dev server: `npm run dev:web`

Verify each route shows the fullscreen button and the Dialog opens and closes correctly:

| Route | Notes |
|-------|-------|
| `/intakes/[id]/review` | Standard intake review flow |
| `/patient/[patientId]/document-records` | Dialog-inside-Dialog: the fullscreen Dialog should open above the legacy FileViewer Dialog via MUI portal — PoC confirmed this works |
| `/patient/[patientId]/prior-auth/[intakeId]` | Prior auth document view |
| `/patient/[patientId]/clinical-reviews/new` | New clinical review step that embeds a PDF |
| `/patient/[patientId]/clinical-reviews/[reviewId]` | Clinical review detail |

Also confirm for the additional automatic call-site consumers (confirm these get the feature as expected):

- `IntakesPageLegacy` (renders `EmbeddedFileViewer`)
- `ClinicalReviewDisplay.tsx`

**Stop here and wait for the user's browser-side visual verification before any further git operations.** Do not commit, do not proceed to wrap-up, until visual sign-off is received.

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/components/UI/FileViewer/ReactPdfRenderer.tsx` | Modify | Add `onRequestFullscreen?: () => void` prop; import `OpenInFullIcon`; render conditional toolbar button after `rotationControls` inside `.toolbarLeft`, reusing `styles.toolbarButton` |
| `apps/web/components/UI/FileViewer/ReactPdfRenderer.test.tsx` | Modify | Add `jest.mock('@mui/icons-material/OpenInFull', ...)` alongside existing icon mocks; append `describe('fullscreen button', ...)` block with three test cases |
| `apps/web/components/UI/FileViewer/EmbeddedFileViewer.tsx` | Modify | Import `Dialog` from `@repo/ui`; add `isFullscreen` boolean state; pass `onRequestFullscreen` to inline renderer; add sibling `<Dialog>` containing a second `ReactPdfRenderer` instance without the fullscreen callback |
| `apps/web/components/UI/FileViewer/EmbeddedFileViewer.test.tsx` | Modify | Update `next/dynamic` mock to expose and trigger `onRequestFullscreen`; append `describe('fullscreen Dialog', ...)` block with five test cases |

---

## Testing Strategy

### Unit tests

**`ReactPdfRenderer.test.tsx` — three new cases:**

- Button is absent from the DOM when `onRequestFullscreen` is `undefined` (default) — asserts `queryByTitle('Open in fullscreen')` returns null.
- Button is present when `onRequestFullscreen` is a function — asserts `getByTitle('Open in fullscreen')` exists and `getByTestId('open-in-full-icon')` is in the document.
- Click on the button invokes the callback exactly once — `toHaveBeenCalledTimes(1)`.

**`EmbeddedFileViewer.test.tsx` — five new cases:**

- Inline renderer receives a defined `onRequestFullscreen` prop (trigger button appears in the stub).
- No `role="dialog"` element exists before the callback fires.
- After firing the callback, a Dialog renders and the DOM contains two `react-pdf-renderer` stubs (inline + dialog).
- After closing the Dialog (click the built-in close button), `role="dialog"` is gone.
- The dialog renderer stub does not render a `trigger-fullscreen` button (confirming `onRequestFullscreen` was not forwarded to it).

### Integration and E2E tests

No new integration or E2E tests are required. The PoC validated the Dialog-inside-Dialog behavior on the document-records route. The user will visually verify all five named routes and the two additional consumers in Step 3.

### Coverage target

Both `ReactPdfRenderer.tsx` and `EmbeddedFileViewer.tsx` must remain at or above 80% line and branch coverage after the new tests are added.

### Mock boundaries

- `@mui/icons-material/OpenInFull` — mocked at the module level in `ReactPdfRenderer.test.tsx`.
- `next/dynamic` — mocked in `EmbeddedFileViewer.test.tsx`; stub updated to expose `onRequestFullscreen`.
- `axios` — already mocked in `EmbeddedFileViewer.test.tsx`; follow existing patterns to resolve the blob URL so `objectUrl` state is populated before Dialog assertions run.
- `@repo/ui` Dialog — not mocked; rendered directly. If `LiThemeProvider` is required, use the `renderWithTheme` helper referenced in `DrugSwitchContextNoteDialog.test.tsx`.

---

## Security Considerations

- **PHI handling**: The Dialog `title` prop uses `fileName` (defaulting to `'Document'`). File names at these call sites may include patient-identifiable strings (e.g., a document named after a patient). This is not a new exposure — `fileName` is already displayed in the inline viewer's UI. No change in PHI surface.
- **No new network calls**: The fullscreen instance reuses the `objectUrl` blob URL already fetched by the inline instance. No second request is made to storage, the EHR, or any external service.
- **No new authorization paths**: The Dialog is a pure UI state toggle. The underlying document fetch and authorization happen identically to the existing embedded viewer flow.
- **No input validation required**: `onRequestFullscreen` is an internal callback with no external input.
- **No feature flag**: The feature is unconditional — every PDF rendered by `EmbeddedFileViewer` in PDF mode gets the fullscreen button. This is consistent with the ticket's intent. No env-var or MongoDB-backed flag is needed.

---

## Open Questions

### Resolved

1. **Independent zoom/rotation state — RESOLVED: independent.** The inline instance and the fullscreen Dialog instance maintain independent zoom and rotation state. This is the easiest path: each `ReactPdfRenderer` already manages its own zoom/rotation internally via `useState`, so mounting a second instance requires zero extra wiring. Shared state would require lifting `zoom`, `rotation`, and `currentPage` into `EmbeddedFileViewer`, converting `ReactPdfRenderer` to controlled mode for those props, and adding `onChange` callbacks — a non-trivial refactor of an already-tested component. Confirmed by user on 2026-05-12.

2. **Image mode (`fileType="image"`) — RESOLVED: PDF only.** Confirmed by user on 2026-05-12. The fullscreen Dialog stays gated inside `fileType === 'pdf' && !legacyPdfViewer`. Image mode is out of scope and will not receive the fullscreen button.

3. **Additional automatic consumers — RESOLVED: intended.** Confirmed by user on 2026-05-12. The goal of the feature is that every consumer of `EmbeddedFileViewer` (PDF mode) gets the fullscreen toggle automatically, regardless of where the component is mounted. `IntakesPageLegacy` and `ClinicalReviewDisplay` are expected to receive the feature without modification. Visual QA still includes them as a sanity check.

### Resolved during implementation

4. **`LiThemeProvider` in tests — RESOLVED: needed.** The `@repo/ui` Dialog's styled component reads from Emotion theme tokens (`t.radii.xl`, `t.elevation.lg`, etc.) on every render, so it crashes outside a `LiThemeProvider`. Both `EmbeddedFileViewer.test.tsx` files now alias `render` from `@testing-library/react` to auto-wrap in `LiThemeProvider` via the `wrapper` option — no changes to the 45 existing render call sites. No shared test utility was created; the helper is local to each file (the existing `DrugSwitchContextNoteDialog` precedent does the same).
