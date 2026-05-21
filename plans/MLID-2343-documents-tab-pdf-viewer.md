# MLID-2343 — Order Documents Tab: PDF Viewer with Navigation Footer

## Task Reference

- **Jira**: [MLID-2343](https://localinfusion.atlassian.net/browse/MLID-2343)
- **Predecessor**: [MLID-2011](https://localinfusion.atlassian.net/browse/MLID-2011) — single task that added the `EmbeddedFileViewer` dialog mode; already merged to develop
- **Story Points**: 5
- **Branch**: `feature/MLID-2343-documents-tab-pdf-viewer`
- **Base Branch**: `develop`
- **Status**: Done — pending PR creation. PR doc: `docs/agomez/PR/MLID-2343.md`

---

## Summary

Replaces the "Document Name" external link in the Orders Documents tab with an in-page PDF viewer. Clicking a document name opens the existing `EmbeddedFileViewer` as a dialog; a consumer-owned footer (rendered via the viewer's new `dialogActions` prop) shows prev/next navigation, the document position count, the category chip, and the received date with time. The viewer component stays domain-agnostic — all navigation state and domain data live in the page.

---

## Goals and Non-Goals

**Goals**

- Document Name column renders as a green underlined button (not an anchor) that opens the viewer dialog.
- `EmbeddedFileViewer` gains one optional prop, `dialogActions?: React.ReactNode`, forwarded to the dialog-mode `Dialog`'s `actions` slot only.
- New `DocumentViewerFooter` component, co-located with the page, provides the footer UI: prev arrow, "Document N of total" text, category chip, received date with time, next arrow.
- Navigation steps through `data.documents` in array order; `fileUrl`/`fileName` swap on the viewer triggers the existing re-fetch inside the viewer.
- No `target="_blank"` behavior remains on any document row.

**Non-Goals**

- Auto-mark-as-read when the viewer opens — PO may add later.
- Sharing zoom/rotation state across documents during navigation.
- Keyboard navigation (prev/next via arrow keys).
- Lifting `DocumentViewerFooter` into `@repo/ui`.
- Migrating other routes off the legacy `FileViewer` wrapper.

---

## Architecture

`EmbeddedFileViewer` already renders as an `@repo/ui` `Dialog` when `open`/`onClose` are passed (MLID-2201). This task adds one optional prop, `dialogActions`, that is forwarded as the `actions` slot of that dialog-mode `Dialog`. The inline-mode internal fullscreen `Dialog` (lines 346–369) is not touched — it serves generic consumers and must not receive domain-specific footer content.

The page owns all navigation state via a single `openIndex: number | null` piece of state. Swapping `fileUrl` and `fileName` on the viewer triggers its existing `useEffect([fileUrl, ...])` re-fetch — no new logic inside the viewer. The new `DocumentViewerFooter` is a pure presentational component that receives `currentIndex`, `total`, `category`, `receivedDate`, `onPrev`, and `onNext` as props — no internal state, no domain data imports.

The `formatReceived` function is extracted from `page.tsx` into a sibling file so the footer can reuse it without circular imports. It stays co-located with the documents directory rather than being promoted to a global utility, since its local-timezone formatting semantics are specific to this surface.

---

## Codebase Analysis

**`EmbeddedFileViewer.tsx`** (`apps/web/components/UI/FileViewer/EmbeddedFileViewer.tsx`, 459 lines)

The dialog-mode branch is lines 440–453. It wraps the body in `@repo/ui` `Dialog` with `title={fileName}`, `size="lg"`, `maxWidth="xl"`, `fixedHeight={96}`. No `actions` prop is currently forwarded. The `@repo/ui` `Dialog` already accepts `actions?: React.ReactNode` and renders it inside `DialogActions`. The inline-mode internal fullscreen `Dialog` is at lines 346–369; it must remain unchanged. Re-exported from `apps/web/components/UI/FileViewer/index.ts` and `apps/web/components/UI/index.ts`. Import: `import { EmbeddedFileViewer } from '@/components/UI/FileViewer'`.

**`EmbeddedFileViewer.test.tsx`** (`apps/web/components/UI/FileViewer/__tests__/EmbeddedFileViewer.test.tsx`)

Wraps renders in `<LiThemeProvider>`. Mocks `next/dynamic` (returns a stable `MockPdfRenderer`), `axios` (resolves to a Blob), and `URL.createObjectURL`. The new tests follow this existing setup exactly — no new mock infrastructure needed.

**`page.tsx`** (`apps/web/app/orders-tracker/[category]/[orderId]/documents/page.tsx`)

`'use client'` component. Renders `@repo/ui` `DataGrid` backed by `useOrderDetails().data.documents`. Local `formatReceived(isoValue: string): string` formats to `"M/D/YYYY H:MM AM/PM"` in local timezone; currently used only by the `receivedDate` column's `valueFormatter` and `renderCell`. The `name` column's `renderCell` is currently `<a href={params.row.viewUrl} target="_blank" rel="noreferrer">{params.value}</a>` — this is replaced entirely. Already imports `useState` from React and `@repo/ui` `IconButton`, `Layout`, `Chip`, `Typography`, `Button`, `Card`, `DataGrid`. Test file: `page.test.tsx` with stubs for `useOrderDetails`, `@repo/ui`, `useDocumentsTabNotifications`, `FeatureFlagContext`, `next-auth/react`.

**`OrderDetailsDocumentItem`** (`apps/web/types/orders.ts`, lines 223–229)

```ts
interface OrderDetailsDocumentItem {
  id: string;
  name: string;
  receivedDate: string;   // ISO timestamp string
  category: string;
  viewUrl: string;
}
```

**`@repo/ui` `Dialog` `actions` slot** — confirmed present in `packages/ui/src/components/Dialog/Dialog.tsx`.

**`@repo/ui` `Link` limitation** — does not support custom colors (only `'primary' | 'secondary' | 'inherit'`). The document-name cell must use `@mui/material/Link` with `sx` and `component="button"` — consistent with the `feedback_use_repo_ui.md` fallback rule.

**Green color token** — `colors.status.success` (`#1d8474`) from `packages/ui/src/tokens/colors.ts`. Import: `import { colors } from '@repo/ui/tokens'`.

**`@repo/ui` `Chip`** — `size="sm" variant="outlined"` with default color for the category chip. The "New" badge in the same page already uses `color="success" variant="soft"` — the category chip intentionally uses a different style to avoid visual confusion.

**`@repo/ui` `IconButton`** — available. Chevron icons: `ChevronLeftIcon` and `ChevronRightIcon` from `@mui/icons-material` — consistent with `MoreVertIcon` already used in `page.tsx`.

**`@repo/ui` `Layout`** — already used in `page.tsx` for the section header and in the `receivedDate` cell.

The canonical spec is the implementation guidance comment on Jira ticket MLID-2343. All architecture decisions (Option B consumer-owned navigation, array-order navigation, no auto-mark-as-read) are confirmed and documented there.

---

## Implementation Steps

### Step 0 — Create the feature branch (no commit)

```
git checkout develop
git pull origin develop
git checkout -b feature/MLID-2343-documents-tab-pdf-viewer
```

---

### Step 1 — Forward `dialogActions` to the dialog-mode Dialog in `EmbeddedFileViewer`

**Commit**: `[MLID-2343] - feat(file-viewer): forward dialogActions to Dialog in dialog mode`

**Files**:
- `apps/web/components/UI/FileViewer/__tests__/EmbeddedFileViewer.test.tsx` (modify — RED tests first)
- `apps/web/components/UI/FileViewer/EmbeddedFileViewer.tsx` (modify — GREEN)

**RED** — Add two tests to the existing describe block:

1. `should render dialogActions content inside the Dialog when in dialog mode` — render with `open={true}`, `onClose={jest.fn()}`, and `dialogActions={<div data-testid="viewer-footer">footer</div>}`. Assert `screen.getByTestId('viewer-footer')` is in the document. This relies on `@repo/ui` `Dialog` rendering `actions` in the DOM; since the test uses `LiThemeProvider`, the real `Dialog` renders.

2. `should NOT render dialogActions in the inline mode's internal fullscreen Dialog` — render WITHOUT the `open` prop (inline mode), pass `dialogActions={<div data-testid="viewer-footer">footer</div>}`. Assert `screen.queryByTestId('viewer-footer')` is `null`. This is the regression guard ensuring the generic-popup `Dialog` at lines 346–369 never receives the order-documents footer.

Run `npx jest EmbeddedFileViewer.test.tsx` — expect both tests to fail because `dialogActions` is not yet accepted.

**GREEN** — In `EmbeddedFileViewerProps`:
- Add `dialogActions?: React.ReactNode` with a JSDoc comment: `/** Forwarded to Dialog actions slot. Only used in dialog mode. */`.

In the component destructuring, add `dialogActions`.

In the `isDialogMode` branch (lines 440–453), add `actions={dialogActions}` to the `Dialog` element. The inline-mode fullscreen `Dialog` at lines 346–369 is not modified.

Run `npx jest EmbeddedFileViewer.test.tsx` — both new tests pass, no existing tests regress.

**REFACTOR** — Confirm no `any` types introduced. Confirm the JSDoc matches the Jira spec wording exactly. Verify `EmbeddedFileViewer.test.tsx` still covers the pre-existing dialog-mode and inline-mode test cases.

---

### Step 2 — Viewer wiring, navigation footer, and page changes

**Commit**: `[MLID-2343] - feat(orders/documents): open PDF viewer with paginated footer on document-name click`

This commit covers three sub-changes in source order (RED for all, then GREEN for all, then REFACTOR):

**Files**:
- `apps/web/app/orders-tracker/[category]/[orderId]/documents/formatReceived.ts` (create)
- `apps/web/app/orders-tracker/[category]/[orderId]/documents/formatReceived.test.ts` (create)
- `apps/web/app/orders-tracker/[category]/[orderId]/documents/DocumentViewerFooter.tsx` (create)
- `apps/web/app/orders-tracker/[category]/[orderId]/documents/DocumentViewerFooter.test.tsx` (create)
- `apps/web/app/orders-tracker/[category]/[orderId]/documents/page.tsx` (modify)
- `apps/web/app/orders-tracker/[category]/[orderId]/documents/page.test.tsx` (modify)

**RED — `formatReceived.test.ts`**

Four test cases (all will fail because the file does not exist yet):
- ISO timestamp representing "May 14 2026, 3:05 PM" → `"5/14/2026 3:05 PM"`
- Midnight (12:00 AM): ISO at hour 0 → `"M/D/YYYY 12:00 AM"`
- Noon (12:00 PM): ISO at hour 12 → `"M/D/YYYY 12:00 PM"`
- Invalid/empty string does not throw — returns `"N/A"`

**RED — `DocumentViewerFooter.test.tsx`**

Props type: `{ currentIndex: number; total: number; category: string; receivedDate: string; onPrev: () => void; onNext: () => void }`.

Test cases (all fail because the component does not exist yet):
- Renders `"Document 1 of 4"` when `currentIndex=0`, `total=4`
- Renders `"Document 3 of 4"` when `currentIndex=2`, `total=4`
- Renders a chip with the `category` label
- Renders the formatted date — the expected string is the output of `formatReceived(receivedDate)` (import and call it in the test's arrange block)
- Prev `<button>` is disabled when `currentIndex === 0`
- Prev `<button>` is enabled when `currentIndex === 1`
- Next `<button>` is disabled when `currentIndex === total - 1`
- Next `<button>` is enabled when `currentIndex < total - 1`
- Clicking prev fires `onPrev`
- Clicking next fires `onNext`
- Both buttons are disabled when `total === 1` and `currentIndex === 0`

The test queries buttons by their accessible role; since `@repo/ui` `IconButton` renders a `<button>`, `getAllByRole('button')` should find them. If the component also has other buttons from imported primitives, query by `aria-label` on the IconButtons (set `aria-label="Previous document"` and `aria-label="Next document"` on the IconButtons).

**RED — `page.test.tsx` additions**

Mock `EmbeddedFileViewer` at the module boundary so tests can inspect the props it was called with, without rendering the real viewer:
```
jest.mock('@/components/UI/FileViewer', () => ({
  EmbeddedFileViewer: jest.fn(() => null),
}))
```

Also mock `@mui/material/Link` minimally:
```
jest.mock('@mui/material/Link', () => ({
  __esModule: true,
  default: ({ children, onClick, component, sx }: ...) => (
    <button type="button" onClick={onClick} data-testid="doc-link">{children}</button>
  ),
}))
```

Test cases (all fail because page changes don't exist yet):
- The name column renders an element with `data-testid="doc-link"`, not an `<a>` with `target="_blank"`
- No element with `target="_blank"` exists anywhere in the rendered output (regression guard)
- Clicking the link in row 0 causes `EmbeddedFileViewer` to be called with `open={true}`, `fileUrl={documents[0].viewUrl}`, `fileName={documents[0].name}`
- Clicking the link in row 1 causes `EmbeddedFileViewer` to be called with `fileUrl={documents[1].viewUrl}`, `fileName={documents[1].name}`
- `EmbeddedFileViewer` receives a non-null `dialogActions` prop
- The `dialogActions` prop contains a rendered `DocumentViewerFooter` — verify by rendering `dialogActions` and asserting `"Document 1 of"` text appears
- Firing `onClose` (captured from the most recent `EmbeddedFileViewer` call's props) causes the next render to pass `open={false}` to the viewer (or for the viewer not to render at all — either is acceptable; check for `open={false}` on the mock)
- Clicking next via the `DocumentViewerFooter`'s `onNext` (captured from `dialogActions` props) increments the index — subsequent render shows `documents[1].viewUrl`
- Clicking prev via `onPrev` when `currentIndex > 0` decrements the index
- `onPrev` when `currentIndex === 0` does not decrement below 0 (clamped)
- `onNext` when `currentIndex === documents.length - 1` does not exceed bounds (clamped)

**GREEN — `formatReceived.ts`**

Extract the existing `formatReceived` function body from `page.tsx` verbatim. Export as named export `formatReceived`. No behavior change.

**GREEN — `DocumentViewerFooter.tsx`**

Pure presentational component, `'use client'` directive because it uses `@repo/ui` components that may require it. No state. Props interface:

```ts
interface DocumentViewerFooterProps {
  currentIndex: number;
  total: number;
  category: string;
  receivedDate: string;
  onPrev: () => void;
  onNext: () => void;
}
```

Layout (left to right) via `@repo/ui` `Layout` with `direction="row"` and `align="center"`:
- `<IconButton aria-label="Previous document" onClick={onPrev} disabled={currentIndex === 0}><ChevronLeftIcon /></IconButton>`
- `<Typography>Document {currentIndex + 1} of {total}</Typography>`
- `<Chip label={category} size="sm" variant="outlined" />`
- Date text: `<Typography>{formatReceived(receivedDate)}</Typography>` (imports from `./formatReceived`)
- `<IconButton aria-label="Next document" onClick={onNext} disabled={currentIndex === total - 1}><ChevronRightIcon /></IconButton>`

Import `ChevronLeftIcon` and `ChevronRightIcon` from `@mui/icons-material`. Import `IconButton`, `Layout`, `Typography`, `Chip` from `@repo/ui`. Named export `DocumentViewerFooter`.

**GREEN — `page.tsx` changes**

Three changes:

1. Replace the local `formatReceived` function with `import { formatReceived } from './formatReceived'`.

2. Add state: `const [openIndex, setOpenIndex] = useState<number | null>(null)`. Derive helpers:
   ```ts
   const isViewerOpen = openIndex !== null;
   const currentDoc = openIndex !== null ? data.documents[openIndex] : null;
   ```

3. Replace the `name` column `renderCell`:
   ```tsx
   renderCell: (params) => {
     const docIndex = data.documents.findIndex((d) => d.id === params.row.id);
     return (
       <Link
         component="button"
         onClick={() => setOpenIndex(docIndex)}
         sx={{ color: colors.status.success, textDecoration: 'underline' }}
       >
         {params.value as string}
       </Link>
     );
   },
   ```
   Add `import Link from '@mui/material/Link'` and `import { colors } from '@repo/ui/tokens'` at the top of the file.

4. Below the closing `</Card>` tag (before the closing `</Layout>`), add the conditional viewer render:
   ```tsx
   {currentDoc && (
     <EmbeddedFileViewer
       fileUrl={currentDoc.viewUrl}
       fileName={currentDoc.name}
       fileType="pdf"
       open={isViewerOpen}
       onClose={() => setOpenIndex(null)}
       dialogActions={
         <DocumentViewerFooter
           currentIndex={openIndex!}
           total={data.documents.length}
           category={currentDoc.category}
           receivedDate={currentDoc.receivedDate}
           onPrev={() => setOpenIndex((i) => Math.max(0, (i ?? 0) - 1))}
           onNext={() =>
             setOpenIndex((i) =>
               Math.min(data.documents.length - 1, (i ?? 0) + 1)
             )
           }
         />
       }
     />
   )}
   ```
   Import `EmbeddedFileViewer` from `@/components/UI/FileViewer` and `DocumentViewerFooter` from `./DocumentViewerFooter`.

Run `npx jest formatReceived.test.ts DocumentViewerFooter.test.tsx page.test.tsx EmbeddedFileViewer.test.tsx` — all tests pass.

**REFACTOR**

- Confirm the `openIndex!` non-null assertion inside the `dialogActions` JSX is safe (it is, because `currentDoc && (...)` already guards that `openIndex !== null` at that render point). Add an inline comment if not obvious.
- Confirm no `any` types in any of the four new/modified files.
- Confirm `DocumentViewerFooter` does not import anything from `page.tsx` (dependency direction is one-way: page imports footer, footer imports `formatReceived`).
- Remove the now-duplicate `formatReceived` function body from `page.tsx` (it should be replaced by the import in the GREEN step, so this is a verification step only).

---

### Step 3 — Visual verification (no commit)

Per `feedback_visual_test_before_commit.md`, stop here and verify in the browser before committing either step.

Run `npm run dev:web` and navigate to an order with multiple documents in `/orders-tracker/[category]/[orderId]/documents`.

Checklist:
- Document Name cells render as green underlined text that looks like a link
- Clicking a name opens the dialog (not a new tab); the PDF loads
- Dialog title shows the document's name
- Footer is visible at the bottom of the dialog: left chevron, "Document N of total", category chip, received date with time, right chevron
- Left chevron is disabled on the first document; right chevron is disabled on the last document
- Both chevrons disabled when the order has exactly one document
- Clicking next swaps the PDF (brief loading state expected — viewer re-fetches)
- Clicking prev steps back correctly; index is clamped (no wrap-around)
- Closing via the X button clears the viewer (no stale state on reopen of the same row)
- No browser tab opens when clicking a document name
- Category chip text matches the document's category
- Received date in the footer includes both date and time (e.g., `"5/14/2026 3:05 PM"`)
- Compare against the playground reference at `https://li-playground.vercel.app/orders/new/319309/documents`
- Confirm the green shade (`#1d8474`) is acceptable to the designer; flag `colors.status.successDark` (`#065f46`) as an alternative if the designer wants higher contrast
- Confirm the category chip `variant="outlined"` style is acceptable

Do not commit until visual verification passes.

---

### Step 4 — Quality gates and push

From `apps/web/`:
```
npx jest formatReceived.test.ts DocumentViewerFooter.test.tsx page.test.tsx EmbeddedFileViewer.test.tsx
npm run types:check
npm run lint:fix
```

From repo root:
```
npm run format
```

Push:
```
git push -u origin feature/MLID-2343-documents-tab-pdf-viewer
```

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/components/UI/FileViewer/EmbeddedFileViewer.tsx` | Modify | Add `dialogActions?: React.ReactNode` prop; forward it to the dialog-mode `Dialog` only |
| `apps/web/components/UI/FileViewer/__tests__/EmbeddedFileViewer.test.tsx` | Modify | Add two tests: `dialogActions` renders in dialog mode; does not render in inline-mode fullscreen Dialog |
| `apps/web/app/orders-tracker/[category]/[orderId]/documents/formatReceived.ts` | Create | Named export `formatReceived` extracted from `page.tsx`; shared by page and footer |
| `apps/web/app/orders-tracker/[category]/[orderId]/documents/formatReceived.test.ts` | Create | Four test cases: typical timestamp, midnight, noon, invalid input |
| `apps/web/app/orders-tracker/[category]/[orderId]/documents/DocumentViewerFooter.tsx` | Create | Pure presentational footer: prev/next `IconButton`, count text, `Chip`, formatted date |
| `apps/web/app/orders-tracker/[category]/[orderId]/documents/DocumentViewerFooter.test.tsx` | Create | 11 test cases covering disabled states, click handlers, count text, chip, date |
| `apps/web/app/orders-tracker/[category]/[orderId]/documents/page.tsx` | Modify | Replace `formatReceived` with import; add `openIndex` state; replace name-column `renderCell`; render `EmbeddedFileViewer` with `dialogActions` |
| `apps/web/app/orders-tracker/[category]/[orderId]/documents/page.test.tsx` | Modify | Add mock for `EmbeddedFileViewer` and `@mui/material/Link`; add 11 wiring and regression tests |

---

## Testing Strategy

**`EmbeddedFileViewer.test.tsx`** — tests the prop-forwarding contract in isolation. Uses the real `@repo/ui` `Dialog` (via `LiThemeProvider`) so the `actions` slot renders into the DOM. The inline-mode fullscreen `Dialog` does not receive `dialogActions` — validated by asserting the `data-testid="viewer-footer"` marker is absent in inline mode.

**`formatReceived.test.ts`** — pure function unit tests, no mocks needed. Validates the time formatting edge cases (midnight, noon) that the existing code handles but does not test.

**`DocumentViewerFooter.test.tsx`** — renders the component with test props. Mocks `@repo/ui` minimally (or uses real components if they render cleanly in jsdom). Queries buttons by `aria-label` to avoid ambiguity with other buttons. Uses `formatReceived` in the test arrange block to compute the expected date string — the test does not hard-code the formatted output (avoids timezone brittleness in CI).

**`page.test.tsx`** — stubs `EmbeddedFileViewer` via `jest.mock` so it renders `null` but its captured props are inspectable via `(EmbeddedFileViewer as jest.Mock).mock.calls`. Stubs `@mui/material/Link` to render a `<button data-testid="doc-link">` so RTL can `fireEvent.click` it. The `dialogActions` wiring test renders the captured `dialogActions` ReactNode via `rtlRender` in a sub-assertion to check for "Document 1 of" text.

**Mock boundaries**:
- `@/components/UI/FileViewer` → `jest.fn(() => null)` in `page.test.tsx`
- `@mui/material/Link` → minimal button stub in `page.test.tsx`
- All existing mocks in `page.test.tsx` (`useOrderDetails`, `@repo/ui`, `useDocumentsTabNotifications`, `FeatureFlagContext`, `next-auth/react`) are preserved unchanged
- `next/dynamic`, `axios`, `URL.createObjectURL` → already mocked in `EmbeddedFileViewer.test.tsx`

---

## Security Considerations

- **No new API routes** — this task is UI-only. Authentication and authorization requirements are unchanged.
- **No PHI exposure** — `viewUrl` is an internal API route already protected by session auth. The viewer fetches the PDF server-side; no signed blob URLs or pre-signed storage URLs are exposed to the client.
- **No feature flag required for this task** — the PDF viewer is a direct replacement for the existing "open in new tab" link. There is no configuration to gate.
- **No server-side changes** — the `dialogActions` prop is never serialized or transmitted; it is a purely client-side ReactNode.

---

## Acceptance Criteria

- [ ] Clicking a Document Name opens the PDF viewer dialog — no new browser tab opens
- [ ] Document Name cells render as green (`#1d8474`) underlined text (button, not anchor element)
- [ ] Dialog title shows the document's file name
- [ ] PDF renders inside the dialog using the existing `EmbeddedFileViewer` content
- [ ] Footer shows: left chevron, "Document N of total" (1-based), category chip, received date with time, right chevron
- [ ] Left chevron is disabled when the first document is open (`currentIndex === 0`)
- [ ] Right chevron is disabled when the last document is open (`currentIndex === total - 1`)
- [ ] Both chevrons are disabled when the order has exactly one document
- [ ] Clicking next increments the open document; PDF re-fetches and the title updates
- [ ] Clicking prev decrements the open document; PDF re-fetches and the title updates
- [ ] Navigation is clamped: prev does not go below index 0; next does not exceed `documents.length - 1`
- [ ] Closing the dialog via the X button sets `openIndex` to `null`; reopening any row starts fresh
- [ ] No `target="_blank"` attribute remains on any document name element

---

## Risks and Open Questions

**Loading state between documents** — each prev/next click triggers a fresh `axios` fetch inside `EmbeddedFileViewer` (the viewer's `useEffect([fileUrl, ...])` fires on `fileUrl` change). A brief loading overlay is expected between documents. This is not a bug — it is documented in the "Out of Scope" section as an accepted behavior. Testers should be informed so they do not file a bug report.

**`ReactPdfRenderer` toolbar** — the viewer always renders its own zoom/rotation toolbar. This toolbar appears inside the dialog alongside the new footer. If the designer finds the dual-toolbar layout unacceptable, a separate task would be needed to hide the viewer's internal toolbar when in dialog mode. No action required for this task.

**Chip color and variant for category** — using `variant="outlined"` with default color, based on the neutral semantic of a category label. The "New" badge in the same page uses `color="success" variant="soft"`, so the distinction is intentional. Confirm with designer during visual review (Step 3).

**Green token shade** — `colors.status.success` (`#1d8474`) is chosen as the link color because it is the standard success green in the design system and has WCAG contrast verified. The playground may use a different hex. Confirm during visual review; `colors.status.successDark` (`#065f46`) is the next-darker alternative.

**Navigation order is array order, not grid sort order** — "Document 1 of 4" counts documents in the order returned by the API, not the order shown in the DataGrid (which the user may have sorted by received date or category). This was confirmed as acceptable by the PO (documented in the Jira spec). If the PO later wants sort-order navigation, that is a separate task.

**`@mui/material/Link` fallback** — `@repo/ui` `Link` does not support custom colors; `@mui/material/Link` is used as documented in `feedback_use_repo_ui.md`. This is not a gap — it is the correct fallback per the project preference file.

---

## Out of Scope

- Auto-mark-as-read on viewer open (separate concern; PO may add later)
- Sharing zoom/rotation state across documents during navigation (each viewer mount has independent state; expected)
- Keyboard navigation (arrow keys for prev/next) — nice-to-have, deferred
- Migrating other routes off the legacy `FileViewer` wrapper — separate task
- Lifting `DocumentViewerFooter` into `@repo/ui` — keep local until other consumers want it

---

## Branch and Commit Guidance

Single feature branch off `develop`. Two commits:

**Commit 1**: `[MLID-2343] - feat(file-viewer): forward dialogActions to Dialog in dialog mode`

Body: Adds one optional prop `dialogActions?: React.ReactNode` to `EmbeddedFileViewerProps`. The prop is forwarded exclusively to the `@repo/ui` `Dialog`'s `actions` slot in the dialog-mode branch (lines 440–453). The inline-mode internal fullscreen `Dialog` at lines 346–369 is intentionally unchanged — it serves generic consumers across nine routes and must remain domain-agnostic. Two regression tests added: one confirming the actions slot renders in dialog mode, one confirming it is absent in inline mode.

**Commit 2**: `[MLID-2343] - feat(orders/documents): open PDF viewer with paginated footer on document-name click`

Body: Replaces the `target="_blank"` anchor in the Document Name column with an MUI `Link` component="button" that opens the `EmbeddedFileViewer` dialog at the clicked document's index. Navigation state is managed by a single `openIndex: number | null` state value in the page component. Swapping `fileUrl`/`fileName` on the viewer triggers its existing `useEffect` re-fetch — no new logic inside the viewer.

The new `DocumentViewerFooter` is a pure presentational component co-located with the page. It receives `currentIndex`, `total`, `category`, `receivedDate`, `onPrev`, and `onNext` as props and renders: prev `IconButton` (disabled at index 0), "Document N of total" count, category `Chip`, received date with time via extracted `formatReceived`, and next `IconButton` (disabled at last index). Navigation is clamped at both ends.

The `formatReceived` helper is extracted from `page.tsx` into a sibling `formatReceived.ts` module so the footer can import it without circular dependencies. No behavior change to the formatting logic. Navigation order follows the `data.documents` array — not the DataGrid's current sort order — as confirmed by the PO.

No `Co-Authored-By` lines in either commit.

---

## PR Delivery

Do not run `gh pr create`. Write the PR draft to `docs/agomez/PR/MLID-2343.md` and let the user open the PR (against `develop`) manually.
