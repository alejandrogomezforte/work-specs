# MLID-2594 — Order Details Prior Auth Table: Clicking a Row Opens Auth Review

## Task Reference

- **Jira**: [MLID-2594](https://localinfusion.atlassian.net/browse/MLID-2594)
- **Story Points**: 4
- **Branch**: `fix/MLID-2594-prior-auth-row-clickable`
- **Base Branch**: `develop`
- **Status**: Done

---

## Summary

Clicking any cell in the Prior Auth table on the Order Details auth-review tab does not navigate to the Auth Review detail — only the Received date cell works because it wraps its content in a `<Link>`. The fix adds `onRowClick` to the `<DataGrid>`, removes the now-redundant `<Link>` from the Received cell, and adds `event.stopPropagation()` to the preview `<IconButton>` so the preview action is not accidentally combined with row navigation.

---

## Codebase Analysis

- **Buggy component**: `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/AuthReviewIndex.tsx`
  - The `<DataGrid>` at lines 374–388 has no `onRowClick` prop. The only navigation is a `<Link>` wrapping the `receivedAt` cell (lines 171–175). All other columns render plain `<Typography>` with no click handler.
  - `useRouter` is already imported from `next/navigation` (line 22), and `router` is already instantiated in the component (line 62). No new import is needed for navigation.
  - `basePath` is already available in component scope and is the correct base for `${basePath}/auth-review/${row._id}`.
  - The preview `<IconButton>` (lines 232–239) calls `setPreviewLetter(row)` with no `stopPropagation` — once `onRowClick` is added this will accidentally trigger row navigation on preview click.

- **Reference component (correct behavior)**: `apps/web/components/Modules/patient/components/patientPriorAuth/PatientPriorAuth.tsx`
  - Each `<tr>` has `onClick={() => openLetterDetails(letter._id)}`, `onKeyDown` for Enter/Space, and `tabIndex={0}`.
  - The preview-equivalent button calls `event.stopPropagation()`.

- **Shared DataGrid**: `packages/ui/src/components/DataGrid/DataGrid.tsx`
  - Already accepts and forwards `onRowClick`. No change needed to the shared package.

- **Existing tests**: `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/AuthReviewIndex.test.tsx`
  - Covers column rendering, filters, chips, clear-filters, preview panel open/close, and the link on the Received cell (line 144–148: `screen.getByRole('link')`).
  - Has no test for clicking a non-Received cell to navigate. That is the gap this fix covers.
  - The test at line 144 asserts `screen.getByRole('link')` — after removing the `<Link>`, this test must be deleted and replaced with a row-click navigation assertion.

- **Mock gap**: `apps/web/mocks/repo-ui.tsx` lines 331–361 — the mock `DataGrid` renders each row as a `<div role="row">` but does NOT accept or forward `onRowClick`. Tests using this mock cannot exercise `onRowClick` until the mock is extended to call `onRowClick(row)` when a row element is clicked.

---

## Implementation Steps

### Step 1 (RED) — Write failing test: clicking a non-Received cell navigates to the detail route

- **Files**:
  - `apps/web/mocks/repo-ui.tsx`
  - `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/AuthReviewIndex.test.tsx`
- **What**:

  **In `apps/web/mocks/repo-ui.tsx`** — extend the mock `DataGrid` to accept `onRowClick` and wire it onto each row `<div>`:

  ```tsx
  export const DataGrid = ({
    rows,
    columns,
    loading,
    onRowClick,
  }: {
    rows: Array<Record<string, unknown> & { id: string }>;
    columns: MockGridColumn[];
    loading?: boolean;
    onRowClick?: (params: { row: Record<string, unknown> }) => void;
    [key: string]: unknown;
  }) => (
    <div role="grid" data-loading={loading ? 'true' : 'false'}>
      <div role="row">
        {columns.map((col, index) => (
          <div role="columnheader" key={col.field ?? index}>
            {col.headerName}
          </div>
        ))}
      </div>
      {rows.map((row) => (
        <div
          role="row"
          key={row.id}
          onClick={() => onRowClick?.({ row })}
          data-testid={`grid-row-${row.id}`}
        >
          {columns.map((col, index) => (
            <div role="cell" key={col.field ?? index}>
              {col.renderCell
                ? col.renderCell({ row, value: row[col.field] })
                : String(row[col.field] ?? '')}
            </div>
          ))}
        </div>
      ))}
    </div>
  );
  ```

  This change is additive and backward-compatible — existing tests that do not pass `onRowClick` are unaffected because `onRowClick?.({ row })` is a no-op when the prop is absent.

  **In `AuthReviewIndex.test.tsx`** — add two new failing tests and delete the existing link-assertion test (line 144–148, "should link the Received cell to the detail route") which will break once the `<Link>` is removed:

  Test 1 — row click navigates:
  ```ts
  it('should navigate to the letter detail when any row is clicked', () => {
    render(<AuthReviewIndex orderId="ord1" />);
    fireEvent.click(screen.getByTestId('grid-row-letter-1'));
    expect(mockPush).toHaveBeenCalledWith(
      '/orders-tracker/new/ord1/auth-review/letter-1'
    );
  });
  ```

  Test 2 — preview button does NOT navigate:
  ```ts
  it('should open the preview panel and NOT navigate when the preview button is clicked', () => {
    render(<AuthReviewIndex orderId="ord1" />);
    fireEvent.click(screen.getByRole('button', { name: /preview document/i }));
    expect(screen.getByTestId('embedded-viewer')).toBeInTheDocument();
    expect(mockPush).not.toHaveBeenCalled();
  });
  ```

  Run `npm run test -- --testPathPattern="AuthReviewIndex"` and confirm:
  - Both new tests FAIL (no `onRowClick` on `<DataGrid>`, no `stopPropagation` on preview button).
  - The deleted link test no longer appears.
  - All pre-existing tests still pass (mock change is backward-compatible).

- **Commit**: `[MLID-2594] - test(auth-review): extend DataGrid mock with onRowClick; add row-click navigation tests (RED)`

---

### Step 2 (GREEN) — Wire onRowClick and fix preview stopPropagation

- **Files**:
  - `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/AuthReviewIndex.tsx`
- **What**:

  **Change 1 — Add `onRowClick` to `<DataGrid>`:**

  ```tsx
  <DataGrid
    rows={rows}
    columns={columns}
    density="compact"
    disableColumnMenu
    disableRowSelectionOnClick
    pagination
    paginationMode="server"
    rowCount={total}
    paginationModel={paginationModel}
    onPaginationModelChange={handlePaginationModelChange}
    pageSizeOptions={[25, 50, 100]}
    onRowClick={({ row }) =>
      router.push(`${basePath}/auth-review/${row._id as string}`)
    }
    sx={{ border: 'none', borderRadius: 0 }}
  />
  ```

  **Change 2 — Remove the `<Link>` from the `receivedAt` column; render plain text:**

  The `<Link>` becomes redundant and creates a nested-anchor hazard once the entire row is clickable. Replace with a plain text span that preserves the visual date rendering:

  ```tsx
  {
    field: 'receivedAt',
    headerName: 'Received',
    flex: 0.8,
    minWidth: 110,
    renderCell: ({ row }) => (
      <Typography variant="bodySmall">
        {formatUTCDate(row.receivedAt) || '—'}
      </Typography>
    ),
  },
  ```

  The `Link` import from `next/link` should be removed if this was its only usage — verify by scanning the file before deleting the import.

  **Change 3 — Add `event.stopPropagation()` to the preview `<IconButton>`:**

  ```tsx
  {
    field: 'preview',
    headerName: '',
    sortable: false,
    width: 56,
    renderCell: ({ row }) => (
      <IconButton
        size="small"
        aria-label="Preview document"
        onClick={(event) => {
          event.stopPropagation();
          setPreviewLetter(row);
        }}
      >
        <ImageSearchRoundedIcon fontSize="small" />
      </IconButton>
    ),
  },
  ```

  Run `npm run test -- --testPathPattern="AuthReviewIndex"` and confirm both new tests PASS and all pre-existing tests still pass.

- **Commit**: `[MLID-2594] - fix(auth-review): make entire Prior Auth table row clickable; remove Link from receivedAt cell; stop preview click from propagating (GREEN)`

---

### Step 3 (REFACTOR) — Clean up and run full quality checks

- **Files**: No file changes expected. Quality gate only.
- **What**:
  1. Confirm `Link` from `next/link` is no longer imported in `AuthReviewIndex.tsx` (import removed in Step 2).
  2. Run `npm run test -- --testPathPattern="AuthReviewIndex"` — all tests pass.
  3. Run the full mock file's affected tests: `npm run test -- --testPathPattern="mocks/repo-ui"` and any component that uses the mock `DataGrid` directly. Confirm no regressions.
  4. From `apps/web/`: run `npm run types:check` — no new type errors.
  5. From `apps/web/`: run `npm run lint:fix` — no lint errors.
  6. Verify the `onRowClick` callback casts `row._id as string` cleanly — if the type of `row` on the `onRowClick` callback is `GridRowParams` (from the DataGrid props), confirm `row._id` is accessible. The `rows` array is typed as `(PriorAuthLetter & { id: string })[]`, so `_id` is present on every row. If TypeScript surfaces an index-access warning, use `String(row._id)` instead of `as string`.

- **Commit**: Only create this commit if the cleanup in step 6 above produces an actual code change. Do not create a formatting-only commit.

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/mocks/repo-ui.tsx` | Modify | Add `onRowClick` prop to mock `DataGrid`; wire it onto row `<div>` elements; add `data-testid` per row |
| `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/AuthReviewIndex.tsx` | Modify | Add `onRowClick` to `<DataGrid>`; replace `<Link>` in `receivedAt` cell with plain `<Typography>`; add `event.stopPropagation()` to preview `<IconButton>`; remove unused `Link` import |
| `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/AuthReviewIndex.test.tsx` | Modify | Delete the "should link the Received cell to the detail route" test; add row-click navigation test; add preview-does-not-navigate test |

---

## Testing Strategy

### Unit tests

**Test case 1 (new) — row click navigates to detail:**
- Arrange: render `AuthReviewIndex` with `sampleLetter` in hook return; `mockPush` wired to `useRouter`.
- Act: `fireEvent.click(screen.getByTestId('grid-row-letter-1'))`.
- Assert: `mockPush` called with `/orders-tracker/new/ord1/auth-review/letter-1`.

**Test case 2 (new) — preview click does NOT navigate:**
- Arrange: render `AuthReviewIndex` with `sampleLetter`.
- Act: `fireEvent.click(screen.getByRole('button', { name: /preview document/i }))`.
- Assert: `screen.getByTestId('embedded-viewer')` is in the document; `mockPush` has NOT been called.

**Test case 3 (deleted) — "should link the Received cell to the detail route":**
- This test relied on `screen.getByRole('link')`. That element is removed in Step 2. Delete this test and replace its intent with test case 1 above.

**All pre-existing tests must remain green** after the mock change:
- Column rendering, filter controls, active filter chips, clear-filters, preview panel open/close, preview Open-button navigation, empty state, loading state.

### Manual verification (stage)

After deploying to stage:

1. Open an order that has Prior Auth letters at `/orders-tracker/new/{id}/auth-review`.
2. Click the **Drug** cell on a row — the browser must navigate to the Auth Review detail page.
3. Click the **Payor** cell on a row — same navigation.
4. Click the **Letter Type** cell on a row — same navigation.
5. Click the **Auth Valid Period** cell on a row — same navigation.
6. Click the **Review Status** cell on a row — same navigation.
7. Click the **Received** date cell on a row — same navigation (now via row click, not Link).
8. Click the **preview icon** button on a row — the in-page preview panel opens and navigation does NOT fire (URL stays the same).
9. Close the preview panel; confirm it closes normally.
10. On the Patient card → Prior Auth tab, verify the existing behavior is unchanged (separate component, not touched).

---

## Security Considerations

- **Input validation**: None required. `row._id` comes from the server-fetched `PriorAuthLetter` document already loaded into component state. No user input is involved in constructing the navigation URL.
- **Authorization**: The auth-review detail route (`/auth-review/[letterId]`) has its own server-side authorization. This fix only changes client-side navigation; it does not bypass any server check.
- **PHI handling**: No PHI is read, logged, or transmitted by this change. The `_id` used in the URL is a MongoDB ObjectId, not a PHI field.

---

## Open Questions

None. All design decisions are specified above.
