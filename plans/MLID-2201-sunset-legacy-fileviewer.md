# MLID-2201 (Part 2) — Sunset Legacy FileViewer Wrapper

## Task Reference

- **Jira**: [MLID-2201](https://localinfusion.atlassian.net/browse/MLID-2201)
- **Story Points**: +2 (additive to Part 1's 2 SP → 4 SP total)
- **Branch**: `feature/MLID-2201-fullscreen-pdf-viewer` (same branch as Part 1 — already 2 commits ahead of develop)
- **Base Branch**: `develop`
- **Status**: Done
- **Predecessor**: [MLID-2201 Part 1 — Fullscreen PDF Viewer](MLID-2201-fullscreen-pdf-viewer.md)

> This file is the addendum plan for the legacy-wrapper sunset work. Part 1 (the fullscreen toggle feature) is already committed on the branch. This plan covers the bundled UX bug fix: the Dialog-inside-Dialog stacking that surfaces on `/patient/[id]/document-records` and `/patient/[id]/documents` after Part 1 lands.

---

## Summary

Part 1 of MLID-2201 added a fullscreen toggle to the embedded PDF viewer via a sibling `@repo/ui` Dialog. The PoC explicitly flagged one edge case as "known but acceptable": on `/patient/[id]/document-records`, `EmbeddedFileViewer` is *already* rendered inside the legacy `FileViewer` wrapper (which is itself a `react-modal`-based Dialog from `@/components/UI/Dialog`). Clicking the fullscreen button stacks the new `@repo/ui` Dialog above the legacy one — two close buttons, two title bars, two scroll surfaces.

Rather than living with the stack, this part sunsets the legacy `FileViewer` wrapper entirely. `EmbeddedFileViewer` gains optional `open` + `onClose` props; when those are defined, the component renders its body inside an `@repo/ui` Dialog directly and suppresses its own fullscreen toggle (no popup-inside-popup). The two consumers (`patientDocumentRecords.tsx` and `patientDocuments.tsx`) are migrated to use this dialog mode, and the legacy wrapper + its test file are deleted.

Outcome: a single dialog system across all PDF/document viewing flows, no stacking, and one fewer legacy artifact in `apps/web/components/UI/FileViewer/`.

---

## Codebase Analysis

### Legacy wrapper (to be deleted)

- `apps/web/components/UI/FileViewer/index.tsx` — 62 lines. Wraps `EmbeddedFileViewer` in `<Dialog isOpen onClose title content={...} contentStyles={{ width: '90%', height: '85vh' }}>` from `@/components/UI`. Re-exports `ContentLibraryFileViewer` and `EmbeddedFileViewer`.
- `apps/web/components/UI/FileViewer/index.test.tsx` — tests the legacy wrapper, fully isolated (mocks both `EmbeddedFileViewer` and the legacy `Dialog`).

### Consumers of the legacy wrapper (4 total — found after a more careful scan)

| File | Line | Props used |
|------|------|------------|
| `apps/web/components/Modules/patient/components/patientDocumentRecords/patientDocumentRecords.tsx` | 548 | `isOpen`, `onClose`, `fileUrl`, `fileName` |
| `apps/web/components/Modules/patient/components/patientDocuments/patientDocuments.tsx` | 162 | `isOpen`, `onClose`, `fileUrl`, `fileName`, `fileType` |
| `apps/web/app/intakes/[id]/page.tsx` | 657 | `isOpen`, `onClose`, `fileUrl`, `fileName`, `fileType` |
| `apps/web/components/Modules/patient/components/patientPriorAuth/PatientPriorAuth.tsx` | 612 | `isOpen`, `onClose`, `fileUrl`, `fileName`, `pageNumber`, `onOpen` |

Multi-line imports caused the first three consumers to surface in an initial single-line regex pass but the other two slipped through. Verified after the migration started.

### `onOpen` dead surface (user decision)

`PatientPriorAuth.tsx` is the only consumer of the legacy wrapper's `onOpen` prop. It hooks `openInNewTab`, which navigates to `/patient/<patientId>/prior-auth/<intakeId>` — the full prior-auth review page. User direction (2026-05-12): **drop this behavior entirely**, do not migrate the "Open" header button. Justification: clicking the row (instead of the preview icon button) already navigates to the same full prior-auth page, which embeds the inline `EmbeddedFileViewer` for the same document. The dialog's "Open" button is therefore redundant. Removing it lets the legacy `onOpen` affordance die with the wrapper, no replacement needed in `EmbeddedFileViewer` dialog mode.

### `ContentLibraryFileViewer` co-location

`ContentLibraryFileViewer.tsx` is independent of the legacy wrapper — co-located in `apps/web/components/UI/FileViewer/` and re-exported through `index.tsx` (line 60). It must continue to be exported from the module after the wrapper is removed. Solution: replace `index.tsx` with a small barrel `index.ts` that re-exports both `EmbeddedFileViewer` and `ContentLibraryFileViewer`.

### Test surface

- `apps/web/components/UI/FileViewer/index.test.tsx` — delete entirely (tests the wrapper we're removing).
- `apps/web/components/Modules/patient/components/patientDocumentRecords/__tests__/patientDocumentRecords.test.tsx` — mocks `FileViewer` at lines 53–73, asserts opening on row click at lines 505 and 649. The mock and assertions must swap to `EmbeddedFileViewer`. Mechanical change, ~3 spots.
- `apps/web/components/Modules/patient/components/patientDocuments/` — **no test file exists**. Nothing to update there.
- `apps/web/components/UI/FileViewer/EmbeddedFileViewer.test.tsx` — extend with a new `Dialog mode` describe block (4 tests) covering the new `open` / `onClose` behavior.

### `EmbeddedFileViewer` blob-fetch lifecycle

`EmbeddedFileViewer` fetches the blob in a `useEffect` on mount and on `fileUrl` change. Today the legacy wrapper passes `fileUrl={selectedFile?.url || ''}` even when closed — the legacy Dialog hides the content via CSS, so the empty-URL fetch fires and silently errors. Acceptable today but ugly. After the migration, the fetch must be gated on `open !== false` so a closed dialog with an empty URL does not fire a bogus request.

### Visual dimensions (per user direction)

The legacy wrapper used `width: 90%, height: 85vh`. We ignore those — the `@repo/ui` Dialog with `size="lg" maxWidth="xl" fixedHeight={96}` (already used by Part 1's fullscreen Dialog) is the new visual standard. No attempt to match legacy dimensions.

### No remaining users of the legacy `@/components/UI/Dialog`?

Out of scope. The legacy Dialog is still used elsewhere (e.g. admin `JobsManagement.tsx`). We are only sunsetting the `FileViewer` wrapper, not the legacy Dialog primitive.

---

## Implementation Steps

Three commits, each a single TDD cycle or atomic refactor. Same branch (`feature/MLID-2201-fullscreen-pdf-viewer`).

### Step 4 — TDD: EmbeddedFileViewer dialog mode

**Files**:
- `apps/web/components/UI/FileViewer/EmbeddedFileViewer.test.tsx` (RED then GREEN)
- `apps/web/components/UI/FileViewer/EmbeddedFileViewer.tsx` (GREEN then REFACTOR)

#### RED — Write failing tests first

Append a new `describe` block at the end of the main `EmbeddedFileViewer.test.tsx`:

```ts
describe('Dialog mode', () => {
  const defaultProps = {
    fileUrl: 'https://example.com/test.pdf',
    fileName: 'TestDocument.pdf',
    fileType: 'pdf' as const,
  };

  beforeEach(() => {
    mockAxios.mockResolvedValue({
      status: 200,
      data: new Blob(['PDF content'], { type: 'application/pdf' }),
    });
  });

  it('should not render a Dialog when open prop is undefined (inline mode)', async () => {
    render(<EmbeddedFileViewer {...defaultProps} />);

    await waitFor(() => {
      expect(screen.getByTestId('react-pdf-renderer')).toBeInTheDocument();
    });

    expect(screen.queryByRole('dialog')).not.toBeInTheDocument();
  });

  it('should render the viewer inside a Dialog when open=true', async () => {
    const handleClose = jest.fn();

    render(
      <EmbeddedFileViewer
        {...defaultProps}
        open={true}
        onClose={handleClose}
      />
    );

    expect(screen.getByRole('dialog')).toBeInTheDocument();

    await waitFor(() => {
      expect(screen.getByTestId('react-pdf-renderer')).toBeInTheDocument();
    });
  });

  it('should not render the Dialog content when open=false', () => {
    const handleClose = jest.fn();

    render(
      <EmbeddedFileViewer
        {...defaultProps}
        open={false}
        onClose={handleClose}
      />
    );

    expect(screen.queryByRole('dialog')).not.toBeInTheDocument();
  });

  it('should skip the blob fetch when open=false', () => {
    const handleClose = jest.fn();

    render(
      <EmbeddedFileViewer
        {...defaultProps}
        open={false}
        onClose={handleClose}
      />
    );

    // Run any queued timers to make sure no deferred fetch fires either.
    jest.runOnlyPendingTimers();

    expect(mockAxios).not.toHaveBeenCalled();
  });

  it('should not pass onRequestFullscreen to the inner renderer when in dialog mode', async () => {
    const handleClose = jest.fn();

    render(
      <EmbeddedFileViewer
        {...defaultProps}
        open={true}
        onClose={handleClose}
      />
    );

    await waitFor(() => {
      expect(screen.getByTestId('react-pdf-renderer')).toBeInTheDocument();
    });

    // Dialog mode disables the inline fullscreen affordance — the
    // trigger-fullscreen stub only renders when onRequestFullscreen is set.
    expect(screen.queryByTestId('trigger-fullscreen')).not.toBeInTheDocument();
  });

  it('should call onClose when the Dialog close button is clicked', async () => {
    const handleClose = jest.fn();

    render(
      <EmbeddedFileViewer
        {...defaultProps}
        open={true}
        onClose={handleClose}
      />
    );

    fireEvent.click(screen.getByRole('button', { name: /close/i }));

    expect(handleClose).toHaveBeenCalledTimes(1);
  });
});
```

Run: `npx jest --testPathPattern="EmbeddedFileViewer.test" --no-coverage`
Expected: all six new tests fail — `open` / `onClose` props do not exist yet.

#### GREEN — Minimal implementation

In `EmbeddedFileViewer.tsx`:

1. Extend the `EmbeddedFileViewerProps` interface:
   ```ts
   /** When defined, the viewer renders inside an @repo/ui Dialog instead of inline. */
   open?: boolean;
   /** Required when `open` is provided. Fired when the user closes the Dialog. */
   onClose?: () => void;
   ```

2. Destructure `open` and `onClose` from props.

3. Gate the blob-fetch effect early-return:
   ```ts
   useEffect(() => {
     if (open === false) return;
     // ...existing fetch logic
   }, [fileUrl, open]);
   ```
   Also clear any cached `objectUrl` on `open=false` transition if the existing cleanup logic doesn't already handle it.

4. At the top of the component's return statement, branch on `open`:
   ```tsx
   const inlineContent = (
     <div className={styles.embeddedFileViewerContainer}>
       {/* existing JSX body */}
     </div>
   );

   if (open !== undefined) {
     return (
       <Dialog
         open={open}
         onClose={onClose ?? (() => {})}
         title={fileName}
         size="lg"
         maxWidth="xl"
         fixedHeight={96}
       >
         {open ? inlineContent : null}
       </Dialog>
     );
   }

   return inlineContent;
   ```

5. In the dialog-mode path, the existing inline fullscreen button + sibling fullscreen Dialog (added in Part 1) **must be conditionally skipped**. Two options:
   - **Preferred**: pass `onRequestFullscreen={undefined}` to the inner `ReactPdfRenderer` and skip rendering the sibling Dialog when `open !== undefined`. Cleanest.
   - Alternative: compute a `isDialogMode = open !== undefined` boolean and gate both the `onRequestFullscreen` prop and the sibling Dialog render on `!isDialogMode`.

6. Confirm `onClose` is required when `open` is provided. Add a runtime guard or rely on TypeScript discriminated union if necessary. The pragmatic path is to make `onClose` optional in the type and fall back to a noop if missing (`onClose ?? (() => {})`), which keeps the type simple at the call-site.

Run: `npx jest --testPathPattern="EmbeddedFileViewer.test" --no-coverage`
Expected: all tests (existing + 6 new) pass.

#### REFACTOR

- Confirm dialog-mode path does **not** render the sibling fullscreen Dialog from Part 1 (otherwise we re-introduce stacking inside the dialog mode itself).
- Confirm the blob fetch fires exactly once on `open=true` transition and never on `open=false`.
- Run `npm run lint:fix` and `npm run types:check` from `apps/web/`.

**Commit:**
```
[MLID-2201] - feat(file-viewer): support dialog mode in EmbeddedFileViewer

Adds optional open and onClose props to EmbeddedFileViewer. When open is
defined, the component renders its body inside an @repo/ui Dialog
(size="lg", maxWidth="xl", fixedHeight=96, title=fileName) and:
- Suppresses the inline fullscreen button — no popup-inside-popup.
- Skips the sibling fullscreen Dialog added in the previous commit.
- Gates the blob fetch on open !== false to avoid bogus fetches when the
  dialog is closed but the component is mounted with an empty URL.

Existing inline callers (open undefined) are unaffected. The new mode
exists so callers that today wrap EmbeddedFileViewer in the legacy
FileViewer Dialog can instead render EmbeddedFileViewer directly.

Six new tests cover: no Dialog in inline mode, Dialog renders on
open=true, no Dialog on open=false, fetch skipped on open=false, no
onRequestFullscreen forwarded in dialog mode, onClose fires on close
button click.
```

---

### Step 5 — Migrate all 4 consumers to dialog mode

**Files**:
- `apps/web/components/Modules/patient/components/patientDocumentRecords/patientDocumentRecords.tsx`
- `apps/web/components/Modules/patient/components/patientDocumentRecords/__tests__/patientDocumentRecords.test.tsx`
- `apps/web/components/Modules/patient/components/patientDocuments/patientDocuments.tsx`
- `apps/web/components/Modules/patient/components/patientDocuments/__tests__/patientDocuments.test.tsx` (new file)
- `apps/web/app/intakes/[id]/page.tsx`
- Intake-detail test file (if one exists — check)
- `apps/web/components/Modules/patient/components/patientPriorAuth/PatientPriorAuth.tsx`
- `apps/web/components/Modules/patient/components/patientPriorAuth/__tests__/PatientPriorAuth.test.tsx`

#### patientDocumentRecords.tsx

Swap the import on line 3:
```ts
// Before
import { FileViewer, LoadingSpinner, Pagination } from '@/components/UI';

// After
import { LoadingSpinner, Pagination } from '@/components/UI';
import { EmbeddedFileViewer } from '@/components/UI/FileViewer';
```

Swap the JSX at lines 548–553:
```tsx
// Before
<FileViewer
  isOpen={fileViewerOpen}
  onClose={closeFileViewer}
  fileUrl={selectedFile?.url || ''}
  fileName={selectedFile?.name || ''}
/>

// After
<EmbeddedFileViewer
  open={fileViewerOpen}
  onClose={closeFileViewer}
  fileUrl={selectedFile?.url || ''}
  fileName={selectedFile?.name || ''}
/>
```

> `pdfInitialZoom` is no longer passed explicitly — the `100` default kicks in automatically when `open` is defined.

#### patientDocuments.tsx

Swap the import on line 3:
```ts
// Before
import { BackButton, Card, FileViewer, LoadingSpinner } from '@/components/UI';

// After
import { BackButton, Card, LoadingSpinner } from '@/components/UI';
import { EmbeddedFileViewer } from '@/components/UI/FileViewer';
```

Swap the JSX at lines 161–169:
```tsx
// Before
{selectedFile && (
  <FileViewer
    isOpen={fileViewerOpen}
    onClose={closeFileViewer}
    fileUrl={selectedFile.url}
    fileName={selectedFile.name}
    fileType={selectedFile.type}
  />
)}

// After
{selectedFile && (
  <EmbeddedFileViewer
    open={fileViewerOpen}
    onClose={closeFileViewer}
    fileUrl={selectedFile.url}
    fileName={selectedFile.name}
    fileType={selectedFile.type}
  />
)}
```

#### patientDocuments.test.tsx (new file)

Create `apps/web/components/Modules/patient/components/patientDocuments/__tests__/patientDocuments.test.tsx` mirroring the `patientDocumentRecords.test.tsx` structure. Minimum coverage:

- Renders the documents table when `loadDocuments` resolves.
- Clicking the "View" button on a row opens `EmbeddedFileViewer` with the correct `fileUrl`, `fileName`, and `fileType`.
- The dialog is not rendered when no file is selected.
- The dialog closes when `onClose` fires.

Use the same `EmbeddedFileViewer` mock pattern (returns a stub `<div data-testid="embedded-file-viewer" />` only when `open=true`) and mock the patient context + `fetchIntakes` service.

#### apps/web/app/intakes/[id]/page.tsx

Swap the multi-line import to drop `FileViewer` and add `EmbeddedFileViewer`:
```ts
// Before
import {
  BackButton,
  Button,
  Card,
  Divider,
  ErrorIndicator,
  FileViewer,
  LoadingSpinner,
  PatientSelector,
} from '@/components/UI';

// After
import {
  BackButton,
  Button,
  Card,
  Divider,
  ErrorIndicator,
  LoadingSpinner,
  PatientSelector,
} from '@/components/UI';
import { EmbeddedFileViewer } from '@/components/UI/FileViewer';
```

Swap the JSX at lines 655–664:
```tsx
// Before
{fileViewer.file && (
  <FileViewer
    isOpen={fileViewer.isOpen}
    onClose={closeFileViewer}
    fileUrl={fileViewer.file.url}
    fileName={fileViewer.file.name}
    fileType={fileViewer.file.type}
  />
)}

// After
{fileViewer.file && (
  <EmbeddedFileViewer
    open={fileViewer.isOpen}
    onClose={closeFileViewer}
    fileUrl={fileViewer.file.url}
    fileName={fileViewer.file.name}
    fileType={fileViewer.file.type}
  />
)}
```

#### PatientPriorAuth.tsx

Swap the import:
```ts
// Before
import {
  Card,
  FileViewer,
  LoadingSpinner,
  Pagination,
  Select,
} from '@/components/UI';

// After
import { Card, LoadingSpinner, Pagination, Select } from '@/components/UI';
import { EmbeddedFileViewer } from '@/components/UI/FileViewer';
```

Swap the JSX at lines 612–619, **dropping `onOpen`**:
```tsx
// Before
<FileViewer
  isOpen={previewOpen}
  onClose={closePreview}
  fileUrl={selectedFile?.url || ''}
  fileName={selectedFile?.name || ''}
  pageNumber={selectedFile?.pageNumber}
  onOpen={openInNewTab}
/>

// After
<EmbeddedFileViewer
  open={previewOpen}
  onClose={closePreview}
  fileUrl={selectedFile?.url || ''}
  fileName={selectedFile?.name || ''}
  pageNumber={selectedFile?.pageNumber}
/>
```

Delete the now-dead `openInNewTab` helper and its `useRouter` usage if no other code references it:
- `const openInNewTab = () => { ... };` (lines 347–352 area)
- Any unused router import remains because other handlers use it — confirm before removing.

#### PatientPriorAuth.test.tsx mock update

The test mocks `FileViewer` at lines 31–103. Replace with an `EmbeddedFileViewer` mock following the same pattern used in `patientDocumentRecords.test.tsx`. Remove any assertion that exercises the "Open" header button — that affordance is gone.

#### patientDocumentRecords.test.tsx

Update the mock at lines 53–73 to mock `EmbeddedFileViewer` instead of `FileViewer`:

```ts
// Replace the `FileViewer` mock with an EmbeddedFileViewer mock that
// exposes `open` as a data attribute so tests can assert the dialog
// opens on row click. The existing test names and assertion structure
// stay the same; only the testid and prop names change.
jest.mock('@/components/UI/FileViewer', () => ({
  EmbeddedFileViewer: (props: { open?: boolean; fileUrl: string }) =>
    props.open ? (
      <div data-testid="embedded-file-viewer" data-file-url={props.fileUrl} />
    ) : null,
}));
```

Update assertion calls at lines 505 and 649 to query `embedded-file-viewer` instead of the legacy `FileViewer` testid.

Run the affected tests:
```bash
npx jest --testPathPattern="patientDocumentRecords.test" --no-coverage
```

**Commit:**
```
[MLID-2201] - refactor(patient-documents): use EmbeddedFileViewer dialog mode directly

Replaces the legacy FileViewer Dialog wrapper with direct
EmbeddedFileViewer usage in dialog mode on both patient-document pages.
This eliminates the Dialog-inside-Dialog stack that surfaced on
/patient/[id]/document-records after the MLID-2201 fullscreen toggle
landed: previously, clicking the toolbar's fullscreen icon would mount
the new @repo/ui Dialog on top of the legacy react-modal Dialog, leaving
two close buttons and two title bars overlapped.

Now both pages render <EmbeddedFileViewer open={...} onClose={...} />
directly. The inner fullscreen button is suppressed automatically by
the dialog-mode contract (you're already in a popup), so the stacking
case disappears entirely.

Updates the FileViewer mock in patientDocumentRecords.test.tsx to mock
EmbeddedFileViewer instead. patientDocuments.tsx has no test file.
```

---

### Step 6 — Delete legacy wrapper

After Step 5 lands, the legacy wrapper has zero consumers. Delete it and replace `index.tsx` with a minimal barrel.

**Files**:
- Delete: `apps/web/components/UI/FileViewer/index.tsx`
- Delete: `apps/web/components/UI/FileViewer/index.test.tsx`
- Create: `apps/web/components/UI/FileViewer/index.ts` (new barrel)

#### New barrel content

```ts
export { EmbeddedFileViewer } from './EmbeddedFileViewer';
export { ContentLibraryFileViewer } from './ContentLibraryFileViewer';
```

That's it — no default export, no legacy `FileViewer` symbol.

#### Verification

After deletion:
1. `grep -rn "from '@/components/UI/FileViewer'" apps/web` — confirm all imports resolve to the new barrel (or directly to `./EmbeddedFileViewer` / `./ContentLibraryFileViewer`).
2. `grep -rn "\\bFileViewer\\b" apps/web` — confirm zero remaining references to the legacy `FileViewer` symbol (excluding the directory name `FileViewer/`).
3. `apps/web/components/UI/index.ts` line 19 (`export * from './FileViewer';`) — keeps working because the new `index.ts` re-exports both named symbols.
4. Run `npm run types:check` and `npm run lint` from `apps/web/`.
5. Run the affected test suites:
   ```bash
   npx jest --testPathPattern="FileViewer|patientDocumentRecords" --no-coverage
   ```

**Commit:**
```
[MLID-2201] - chore(file-viewer): remove legacy FileViewer wrapper

Drops the legacy FileViewer wrapper (apps/web/components/UI/FileViewer/
index.tsx) and its test now that both consumers have been migrated to
EmbeddedFileViewer dialog mode. Replaces index.tsx with a small index.ts
barrel that re-exports EmbeddedFileViewer and ContentLibraryFileViewer
directly.

The wrapper carried an unused onOpen header-button prop (grep confirmed
zero callers) and depended on the legacy react-modal Dialog from
@/components/UI/Dialog. Removing it leaves the FileViewer module on a
single dialog stack (@repo/ui) and eliminates the Dialog-inside-Dialog
stacking case for good.

The legacy @/components/UI/Dialog primitive itself is still used
elsewhere (e.g. admin JobsManagement) and stays untouched — out of
scope for this story.
```

---

### Step 7 — Quality gates and visual verification

**No new commit unless fixes are required.**

#### Automated checks

Run all from `apps/web/`:

```bash
npm run test -- FileViewer
npm run test -- patientDocumentRecords
npm run types:check
npm run lint
npm run format
```

Coverage check: `EmbeddedFileViewer.tsx` must remain at or above 80% line/branch coverage after the dialog-mode branch is added.

#### Manual visual verification (browser)

Start the dev server: `npm run dev:web`

Verify:

| Route | What to verify |
|-------|----------------|
| `/patient/[patientId]/document-records` | Click a document row — opens a single `@repo/ui` Dialog (green title bar, single close X, no stacked dialog). No fullscreen button inside (correct — you're already in the popup). |
| `/patient/[patientId]/documents` | Click "View" on a document row — same single-dialog behavior. PDF and image both work. |
| `/intakes/[id]/review` | Inline viewer still works. Toolbar fullscreen button still pops out a Dialog (Part 1 behavior preserved). |
| `/patient/[patientId]/prior-auth/[intakeId]` | Same — inline mode unaffected. |
| `/patient/[patientId]/clinical-reviews/new` | Same — inline mode unaffected. |
| `/patient/[patientId]/clinical-reviews/[reviewId]` | Same — inline mode unaffected. |

**Stop here and wait for the user's browser-side visual verification before any further git operations.** Per `feedback_visual_test_before_commit`.

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/components/UI/FileViewer/EmbeddedFileViewer.tsx` | Modify | Add `open` / `onClose` props; wrap body in `@repo/ui` Dialog when `open` is defined; gate blob fetch on `open !== false`; suppress inline fullscreen button + sibling Dialog in dialog mode |
| `apps/web/components/UI/FileViewer/EmbeddedFileViewer.test.tsx` | Modify | Add `Dialog mode` describe block with six tests |
| `apps/web/components/Modules/patient/components/patientDocumentRecords/patientDocumentRecords.tsx` | Modify | Swap legacy `FileViewer` for `EmbeddedFileViewer` in dialog mode |
| `apps/web/components/Modules/patient/components/patientDocumentRecords/__tests__/patientDocumentRecords.test.tsx` | Modify | Swap legacy `FileViewer` mock for `EmbeddedFileViewer` mock; update assertion testid at lines 505 / 649 |
| `apps/web/components/Modules/patient/components/patientDocuments/patientDocuments.tsx` | Modify | Swap legacy `FileViewer` for `EmbeddedFileViewer` in dialog mode |
| `apps/web/components/Modules/patient/components/patientDocuments/__tests__/patientDocuments.test.tsx` | Create | New test file: row-click → dialog-open flow + onClose behavior |
| `apps/web/components/UI/FileViewer/index.tsx` | Delete | Legacy wrapper component (62 lines) |
| `apps/web/components/UI/FileViewer/index.test.tsx` | Delete | Legacy wrapper test |
| `apps/web/components/UI/FileViewer/index.ts` | Create | Minimal barrel: re-export `EmbeddedFileViewer` + `ContentLibraryFileViewer` |

---

## Testing Strategy

### New unit tests (6 in `EmbeddedFileViewer.test.tsx`, Dialog mode describe block)

1. No Dialog rendered when `open` is undefined (inline mode lock-in).
2. Dialog rendered when `open=true` and inline content visible inside.
3. No Dialog rendered when `open=false`.
4. Blob fetch (`axios`) does not fire when `open=false`.
5. Inner `ReactPdfRenderer` does not receive `onRequestFullscreen` in dialog mode (no `trigger-fullscreen` stub).
6. `onClose` fires when the Dialog's built-in close button is clicked.

### Updated existing tests

- `patientDocumentRecords.test.tsx` — swap mock target from `FileViewer` to `EmbeddedFileViewer`; update testid from `file-viewer` (or whatever the legacy mock used) to `embedded-file-viewer`. ~3 spots.

### Deleted tests

- `apps/web/components/UI/FileViewer/index.test.tsx` — deleted with the wrapper it tested.

### Coverage target

`EmbeddedFileViewer.tsx` must remain at or above 80% line and branch coverage after the dialog-mode branch is added. The 6 new tests should keep coverage healthy.

---

## Security Considerations

- **PHI handling**: No change. `fileName` (already user-visible in the inline viewer) becomes the Dialog title — same PHI surface as Part 1.
- **No new network calls**: The blob is still fetched once per `open=true` transition. Gating the fetch on `open !== false` actually *reduces* network surface — previously the legacy wrapper triggered an empty-URL fetch on every mount.
- **No new authorization paths**: All access control happens upstream in the consuming pages (`patientDocumentRecords`, `patientDocuments`); the viewer is pure UI.
- **HIPAA compliance**: Unchanged. Same file blob, same `fileName` exposure.

---

## Open Questions

### Resolved (user review, 2026-05-12)

1. **`pdfInitialZoom` default for dialog mode — RESOLVED: default to 100 in dialog mode.** When `open` is defined and the caller does not pass `pdfInitialZoom`, `EmbeddedFileViewer` defaults to `100`. Inline mode (open undefined) keeps the existing `50` default. Implemented as `const effectiveInitialZoom = pdfInitialZoom ?? (open !== undefined ? 100 : 50);` inside the component — destructure `pdfInitialZoom` without a default, then compute. Migrated call sites in Step 5 therefore no longer need to pass `pdfInitialZoom={100}` explicitly.

2. **Discriminated-union prop typing — RESOLVED: keep `onClose` optional.** `onClose` stays optional in the public type with a noop fallback inside the component. No discriminated-union typing.

3. **Branch / Jira framing — RESOLVED: same branch, same ticket.** All Step 4–6 commits land on `feature/MLID-2201-fullscreen-pdf-viewer` under MLID-2201.

4. **`patientDocuments.tsx` test file — RESOLVED: add a new test file.** Step 5 now includes creating `apps/web/components/Modules/patient/components/patientDocuments/__tests__/patientDocuments.test.tsx` with minimal coverage of the row-click → dialog-open flow, mirroring the structure of `patientDocumentRecords.test.tsx` so the two components are consistent in test surface.

---

## Routes to test

Start the dev server first: `npm run dev:web` (port 8080).

### Migrated routes (verify single-dialog behavior)

The bug being fixed: the toolbar fullscreen button must **not** appear inside the dialog on these routes (you're already in a popup — no popup-inside-popup).

| # | Route | What to do | What to look for |
|---|-------|------------|------------------|
| 1 | `http://localhost:8080/intakes` → click any intake row | Lands on `/intakes/<id>`. Scroll to the **Documents** section. Click **"View"** on any document. | Single `@repo/ui` Dialog opens (green title bar, single close X). PDF renders inside. Toolbar fullscreen icon should be hidden. |
| 2 | `http://localhost:8080/patient/<patientId>/document-records` | Click any **document row**. | Same single-dialog behavior. No stacked dialogs. No fullscreen icon in the toolbar (dialog mode). |
| 3 | `http://localhost:8080/patient/<patientId>/documents` | Click the **"View"** button on any document row. | Single dialog. Test with both PDF and image documents if available. |
| 4 | `http://localhost:8080/patient/<patientId>/prior-auth` | Click the small **Preview** icon button (eye icon, `aria-label="Preview"`) on any row. | Single dialog. **No "Open" header button** anymore (the dropped affordance). Page-number jump still works when the letter has `documentPage`. |

### Regression checks (inline-mode flows from Part 1)

The fullscreen pop-out feature from Part 1 must still work everywhere `EmbeddedFileViewer` is rendered inline.

| # | Route | What to verify |
|---|-------|----------------|
| 5 | `http://localhost:8080/intakes/<id>/review` | PDF renders **inline** (not in a dialog). Toolbar fullscreen icon **visible**. Click it → `@repo/ui` Dialog pops open with the full-size PDF. |
| 6 | `http://localhost:8080/patient/<patientId>/prior-auth/<intakeId>` | Click into a prior-auth letter row to land here. PDF embedded inline. Fullscreen icon visible. Pop-out works. This is the "Open in new tab" destination that replaces the dropped Open button. |
| 7 | `http://localhost:8080/patient/<patientId>/clinical-reviews/new` | Inline PDF + fullscreen pop-out. |
| 8 | `http://localhost:8080/patient/<patientId>/clinical-reviews/<reviewId>` | Inline PDF + fullscreen pop-out. |

### Quick "did I break the Dialog stack?" check

On routes 1–4, the bug we fixed is: **the toolbar fullscreen button must not appear inside the dialog**. If you see it, the dialog-mode suppression is broken. If you don't see it, the fix is working.
