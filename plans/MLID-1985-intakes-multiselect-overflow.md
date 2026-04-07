# MLID-1985 -- Intakes table: Multiple location selection UI overflow

## Task Reference

- **Jira**: [MLID-1985](https://localinfusion.atlassian.net/browse/MLID-1985)
- **Story Points**: 4
- **Branch**: `fix/MLID-1985-intakes-location-multiselect`
- **Base Branch**: `develop`
- **Status**: Done

---

## Summary

When users select 3+ locations in the Intakes page location filter, the MUI Autocomplete chips overflow and overlap inside the constrained input box (max-width 400px). The approved design (prototyped in `li-ui-playground`) replaces `MuiStandaloneMultiSelect` with the `Autocomplete` component from `@repo/ui`, suppresses chips inside the input via `renderTags={() => null}`, and renders selected locations as removable `Chip` components from `@repo/ui` in a horizontal row below the filter bar.

---

## Codebase Analysis

### Current state (li-call-processor)

- **`MuiStandaloneMultiSelect`** (`apps/web/components/UI/MUI/MuiStandaloneMultiSelect.tsx`) wraps MUI `Autocomplete` with `multiple` + `disableCloseOnSelect`. The `renderTags` prop renders `Chip` components inside the `TextField` input -- this is the overflow root cause.
- **Single consumer**: Only the Intakes page uses `MuiStandaloneMultiSelect`. Safe to replace entirely.
- **Intakes page** (`apps/web/app/intakes/page.tsx`): Builds a `controls` JSX block (lines 874-966) reused in both single-view and split-view layouts. The location filter lives at line 909 inside `<div className={styles.locationFilter}>`.
- **Data flow**: `filters.locations: string[]` holds selected location IDs; `locationOptions: { value: string; label: string }[]` provides the display mapping; `handleLocationsChange` updates filters + URL + re-fetches.

### Design reference (li-ui-playground)

The designer's prototype at `li-ui-playground/src/pages/Intakes.tsx` (lines 638-691) already implements the approved UX:

```tsx
// 1. Autocomplete with renderTags={() => null} — no chips inside input
<Autocomplete
  label="Location"
  multiple
  options={locationAutocompleteOptions}
  value={selectedLocations}
  onChange={(_e, v) => setSelectedLocations(v as string[])}
  placeholder="All locations"
  size="small"
  renderTags={() => null}
/>

// 2. Chips below filter bar — removable, horizontal row
{selectedLocations.length > 0 && (
  <Layout direction="row" gap={2} align="center" wrap="wrap">
    {selectedLocations.map((loc) => (
      <Chip
        key={loc}
        label={loc}
        variant="outlined"
        size="sm"
        onDelete={() => setSelectedLocations((prev) => prev.filter((l) => l !== loc))}
        deleteIcon={<CancelIcon style={{ fontSize: 14 }} />}
      />
    ))}
  </Layout>
)}
```

### Component availability in `@repo/ui`

Both components are already exported from `packages/ui/src/index.ts`:
- `Autocomplete` — generic MUI Autocomplete wrapper with label, loading, minInputLength, renderTags, multiple, etc.
- `Chip` — styled chip with `onDelete`, `deleteIcon`, `variant="outlined"`, `size="sm"` support

---

## Approach

**Replace** `MuiStandaloneMultiSelect` with `Autocomplete` + `Chip` from `@repo/ui`, following the exact pattern proven in the playground prototype.

This is simpler than modifying `MuiStandaloneMultiSelect` because:
1. The playground already validates the UX with these exact components
2. `@repo/ui` `Autocomplete` already supports all needed props (`renderTags`, `multiple`, `onOpen`/`onClose`, loading, disabled)
3. `@repo/ui` `Chip` already has `onDelete` + `deleteIcon` + `variant="outlined"` + `size="sm"`
4. No custom component creation needed — just wire existing design system components
5. `MuiStandaloneMultiSelect` has only 1 consumer, so no migration concerns

---

## Implementation Steps

### Step 1 — Update Intakes page tests for new behavior (RED)

- **File**: `apps/web/app/intakes/page.test.tsx` (modify)
- **What**: Add/update test cases:
  1. "should render Autocomplete for location filter" — verify the `@repo/ui` Autocomplete renders with `multiple` and placeholder "All Locations"
  2. "should render selected location chips below the filter bar" — mock `fetchLocations` to return locations, set URL params with selected locations, verify `Chip` elements appear with correct labels below the filter row
  3. "should remove a location when chip delete button is clicked" — render with selections, click a chip's delete button, verify filter updates (location removed from URL params and re-fetch triggered)
  4. "should not render chips row when no locations selected" — verify no chips appear when `filters.locations` is empty
  5. Update existing location filter tests if they reference `MuiStandaloneMultiSelect` by name

### Step 2 — Replace MuiStandaloneMultiSelect with Autocomplete + Chips (GREEN)

- **File**: `apps/web/app/intakes/page.tsx` (modify)
- **What**:
  1. **Remove** import of `MuiStandaloneMultiSelect`
  2. **Add** imports:
     ```typescript
     import { Autocomplete, Chip } from '@repo/ui';
     import CancelIcon from '@mui/icons-material/Cancel';
     ```
  3. **Replace** the `<MuiStandaloneMultiSelect>` block (lines 909-918) with:
     ```tsx
     <div className={styles.locationFilter}>
       <Autocomplete
         multiple
         options={locationOptions}
         value={locationOptions.filter((opt) =>
           filters.locations?.includes(opt.value)
         )}
         getOptionLabel={(opt) => opt.label}
         isOptionEqualToValue={(opt, val) => opt.value === val.value}
         onChange={(_e, selected) => {
           handleLocationsChange(
             (selected as { value: string; label: string }[]).map((s) => s.value)
           );
         }}
         placeholder="All Locations"
         size="small"
         disabled={locationsLoading || !!locationsError}
         loading={locationsLoading}
         renderTags={() => null}
         disableCloseOnSelect
       />
     </div>
     ```
  4. **Add** chips row after the `.filters` div, inside `.controls` (between lines 964-965):
     ```tsx
     {(filters.locations?.length ?? 0) > 0 && (
       <div className={styles.selectedLocationTags}>
         {(filters.locations || []).map((locationId) => {
           const option = locationOptions.find((opt) => opt.value === locationId);
           return (
             <Chip
               key={locationId}
               label={option?.label || locationId}
               variant="outlined"
               size="sm"
               onDelete={() => {
                 const updated = (filters.locations || []).filter((v) => v !== locationId);
                 handleLocationsChange(updated);
               }}
               deleteIcon={<CancelIcon style={{ fontSize: 14 }} />}
             />
           );
         })}
       </div>
     )}
     ```

- **File**: `apps/web/app/intakes/styles.module.css` (modify)
- **What**: Add a new class for the selected tags row:
  ```css
  .selectedLocationTags {
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
    align-items: center;
  }
  ```
  The `.controls` container is already `flex-direction: column; gap: 16px`, so this row flows naturally below `.filters`.

### Step 3 — Refactor and verify (REFACTOR)

- Clean up any unused imports from the old `MuiStandaloneMultiSelect`
- Run `npm run types:check` from `apps/web/` to catch type issues
- Run `npm run lint` from `apps/web/`
- Manually verify both single-view and split-view modes work

### Step 4 — Evaluate MuiStandaloneMultiSelect deprecation

- **File**: `apps/web/components/UI/MUI/MuiStandaloneMultiSelect.tsx`
- **What**: Since this component now has **zero consumers**, consider:
  - Option A: Delete `MuiStandaloneMultiSelect.tsx` and its test file (clean removal)
  - Option B: Keep it but add a `@deprecated` JSDoc comment pointing to `@repo/ui Autocomplete`
  - **Recommendation**: Option A (delete) — no reason to keep dead code

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/app/intakes/page.tsx` | Modify | Replace `MuiStandaloneMultiSelect` with `@repo/ui` `Autocomplete` + `Chip` pattern |
| `apps/web/app/intakes/page.test.tsx` | Modify | Add tests for chips row rendering/removal; update location filter tests |
| `apps/web/app/intakes/styles.module.css` | Modify | Add `.selectedLocationTags` class |
| `apps/web/components/UI/MUI/MuiStandaloneMultiSelect.tsx` | Delete | No remaining consumers after replacement |
| `apps/web/components/UI/MUI/MuiStandaloneMultiSelect.test.tsx` | Delete | Tests for deleted component |

---

## Testing Strategy

- **Integration tests — Intakes page** (`page.test.tsx`):
  - Autocomplete renders with correct placeholder and multiple mode
  - Chips row appears below filters when locations are selected
  - Chips row disappears when all locations are deselected
  - Clicking chip delete button removes that location from filters
  - Both split-view and single-view layouts show the chips row (both use same `controls` JSX)
  - Loading and disabled states still work on the Autocomplete

- **Manual verification**:
  1. Navigate to Intakes page
  2. Open location dropdown — verify options list appears normally
  3. Select 4+ locations — verify no overflow/overlap in the input
  4. Close the dropdown — verify chips row appears below the filter bar with correct labels
  5. Click × on a chip — verify it's removed and table re-filters
  6. Test with 0 selections — no chips row, placeholder "All Locations" shows
  7. Test with 1 selection — 1 chip below
  8. Test in dual-view (split) mode — same behavior
  9. Test loading state — Autocomplete shows loading indicator

---

## Design Reference

- **Playground prototype**: `li-ui-playground/src/pages/Intakes.tsx` (lines 638-691)
- **Live preview**: https://li-playground.vercel.app/intakes
- **Components used**: `Autocomplete` and `Chip` from `@localinfusion/ui` (= `@repo/ui` in monorepo)

---

## Security Considerations

- **Input validation**: No new API endpoints or user input. Location IDs validated by existing filter flow.
- **Authorization**: No changes — Intakes page access already gated by existing auth.
- **PHI handling**: Not applicable — location names are not PHI.

---

## Resolved Questions

- **Options/value bridging (RESOLVED)**: Use option (b) — pass `{ value, label }` objects directly to the Autocomplete's generics with `getOptionLabel` / `isOptionEqualToValue`. The existing `locationOptions` array is already in this shape. Only mapping is in `onChange`: `.map(s => s.value)` to extract IDs. No risk of label↔ID mismatch, no new data structures needed.
  ```tsx
  <Autocomplete
    multiple
    options={locationOptions}
    value={locationOptions.filter((opt) => filters.locations?.includes(opt.value))}
    getOptionLabel={(opt) => opt.label}
    isOptionEqualToValue={(opt, val) => opt.value === val.value}
    onChange={(_e, selected) => handleLocationsChange(selected.map((s) => s.value))}
    renderTags={() => null}
    disableCloseOnSelect
    placeholder="All Locations"
    size="small"
    disabled={locationsLoading || !!locationsError}
    loading={locationsLoading}
  />
  ```
