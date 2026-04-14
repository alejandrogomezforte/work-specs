# MLID-2025 — Order Tracker: Update Custom Filter Appearance & Multi-select Overlap Fix

## Task Reference

- **Jira**: [MLID-2025](https://localinfusion.atlassian.net/browse/MLID-2025)
- **Story Points**: 4
- **Branch**: `feature/MLID-2025-order-tracker-filter-ui`
- **Base Branch**: `develop`
- **Status**: Done

---

## Summary

UI improvements to the Order Tracker page header and custom filter experience, plus a bug fix for multi-select filter values overlapping the input field. The task has three distinct goals:

1. **Restructure the page header layout** — move search bar to the left, quick filters to the right of search, and change "Clear Filter" from a chip/button to plain text
2. **Improve custom filter visibility** — highlight the filter icon when a custom filter is active, show "Clear filter" as red text, and display chips for each active custom filter below the filter button
3. **Fix multi-select overlap bug** — when using "is any of" operators with multiple values, the selected values overlap inside the input field

Related issue: [MLID-1985](https://localinfusion.atlassian.net/browse/MLID-1985) (same root cause for the overlap bug, already fixed for Intakes page)

---

## Codebase Analysis

### Current Page Header Layout

**File**: `apps/web/app/orders-tracker/[category]/page.tsx` (lines 524–585)

The current layout is a single horizontal row inside a `<Layout direction="row">`:

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ "Quick Filters:" label │ [Clear Filters chip] │ [QF chips...] │ [Search...────] │ 🔲 │ 🔽 │
└──────────────────────────────────────────────────────────────────────────────┘
```

- **Left side**: "Quick Filters:" label + "Clear Filters" chip + quick filter chips
- **Middle** (grows): Search `TextField` with `startIconName="Search"`
- **Right side**: Column visibility `IconButton` + Filter panel `IconButton`

**Desired layout** (from Jira):

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ [Search...────────────────] │ [QF chips...] │ Clear Filters │ 🔲 │ 🔽 │
├──────────────────────────────────────────────────────────────────────────────┤
│ (when custom filter active) [Filter chip 1] [Filter chip 2] [Filter chip 3] │
└──────────────────────────────────────────────────────────────────────────────┘
```

- Search bar moves to the **left**
- Quick filter chips move to the **right of search**
- "Clear Filters" becomes **plain text** (not a chip)
- "Quick Filters:" label removed
- New row below: chips for each active custom filter (only shown when custom filter is active)

### Clear Filters (current)

**File**: `apps/web/app/orders-tracker/[category]/page.tsx` (lines 533–537)

```tsx
<Chip
  label="Clear Filters"
  variant="outlined"
  onClick={clearFilters}
/>
```

Currently a `Chip` from `@repo/ui`. Needs to become a `Typography` or plain text element, styled as red when custom filter is active.

### Custom Filter Panel

The filter panel is MUI DataGrid Pro's built-in panel, triggered by:

```tsx
<IconButton size="small" onClick={() => activeApiRef.current?.showFilterPanel()}>
  <FilterListIcon fontSize="small" />
</IconButton>
```

There is currently **no visual indicator** that a custom filter is active — the icon always looks the same.

### Filter State Management

**File**: `apps/web/app/orders-tracker/hooks/useGridUrlModels.ts`

Filter state is stored in URL search parameters via the `useGridUrlModels` hook:
- `filterModel`: `{ items: GridFilterItem[] }` — contains active custom filters
- `sortModel`: sort directives
- `activeQuickFilter`: string ID or null
- `search`: quick search value

The `filterModel.items` array is the source of truth for active custom filters. Each item has `field`, `operator`, and `value`.

### Multi-Select Overlap Bug

**Root cause**: MUI DataGrid Pro's built-in filter panel uses an Autocomplete for "is any of" operators on `singleSelect` columns. When multiple values are selected, the chips render inside the constrained-width input and overflow/overlap.

**Affected columns** (all `type: 'singleSelect'` with `valueOptions`):

| Column | File | Values |
|--------|------|--------|
| Order Type | `NewOrders.tsx:232` | `NewOrderType` enum |
| Coverage Status | `NewOrders.tsx:315` | `CoverageStatus` enum |
| Order Status | `NewOrders.tsx:328` | `OrderStatus` enum |
| LI Pharm Status | `NewOrders.tsx:341` | `LIPharmacyStatus` enum |
| Audit | `NewOrders.tsx:419` | `Audit` enum |
| Order Status (Maint) | `MaintenanceOrders.tsx:296` | `OrderStatus` enum |
| Coverage Status (Maint) | `MaintenanceOrders.tsx:309` | `CoverageStatus` enum |
| Status (Lead) | `LeadOrder.tsx:163` | `LeadOrderStatus` enum |

**MLID-1985 precedent** (Intakes page — Done, commit `274f933`):

The same overflow bug was fixed in the Intakes location filter by:
1. Replacing `MuiStandaloneMultiSelect` with `Autocomplete` from `@repo/ui`
2. Suppressing chips inside the input with `renderTags={() => null}`
3. Rendering selected values as removable `Chip` components from `@repo/ui` in a flex-wrap row below the filter controls bar
4. Each chip has a `CancelIcon` delete button (fontSize 14) that removes the value from filters

**Key difference for Order Tracker**: The Intakes fix replaced a standalone multi-select component. Here, the multi-select lives inside MUI DataGrid's **built-in filter panel** — the panel renders its own Autocomplete for `singleSelect` columns when the operator is "is any of". We cannot simply replace a component; we need to either:
- (a) Style the DataGrid filter panel's Autocomplete via CSS to handle overflow (flex-wrap on the tags container)
- (b) Provide a custom filter input component via DataGrid column `renderFilterInput` or `filterOperators`

**However**, the Jira task description says the fix is about the **custom filter chips displayed outside the panel** (in the page header row), not inside the filter panel itself. The approach aligns with MLID-1985: show active filter values as `Chip` components in a row below the filter bar, making the overlap inside the panel less important since users see their selections externally.

### Key Files

| File | Role |
|------|------|
| `apps/web/app/orders-tracker/[category]/page.tsx` | Main page — filter bar, quick filters, search, icon buttons (lines 524–585) |
| `apps/web/app/orders-tracker/hooks/useGridUrlModels.ts` | Filter/sort/search state management via URL params |
| `apps/web/app/orders-tracker/Orders/Orders.tsx` | DataGrid wrapper — receives `filterModel`, `onFilterModelChange` |
| `apps/web/app/orders-tracker/Orders/NewOrders.tsx` | Column definitions for New Orders tab |
| `apps/web/app/orders-tracker/Orders/MaintenanceOrders.tsx` | Column definitions for Maintenance Orders tab |
| `packages/ui/src/components/DataGrid/DataGrid.styles.ts` | DataGrid styles including filter panel (lines 223–250) |
| `packages/ui/src/components/Chip/Chip.tsx` | Chip component used for quick filters |
| `packages/ui/src/components/Autocomplete/Autocomplete.tsx` | Autocomplete component (multi-select support) |

### Components Available in `@repo/ui`

- **Layout**: Flexbox with `direction`, `gap`, `align`, `justify`, `padding`, `grow`, `wrap` props
- **Chip**: `variant` (filled/outlined/soft), `size` (sm/md), `onDelete`, `deleteIcon`, `tokenBg`, `tokenColor`
- **Typography**: `variant`, `weight`, `color` (supports "secondary", "error", etc.)
- **TextField**: `startIconName`, `size` (xs/sm/md)
- **IconButton**: `size`, `color`
- **DataGrid**: MUI DataGrid Pro wrapper with custom styles

### Design Tokens

- `status.activeLight` — green highlight (used for active quick filter chips)
- `status.error` / `text.error` — red (for "Clear filter" text when custom filter active)
- `text.secondary` — gray secondary text

### MLID-1985 Fix Pattern (Reference)

Commit `274f9335` changed the Intakes page (`apps/web/app/intakes/page.tsx`):

**Imports added:**
```tsx
import CancelIcon from '@mui/icons-material/Cancel';
import { Autocomplete, Chip as RepoChip } from '@repo/ui';
```

**Autocomplete pattern** (replaces multi-select input):
```tsx
<Autocomplete
  multiple
  options={locationOptions}                           // { value, label }[]
  value={locationOptions.filter((opt) =>
    filters.locations?.includes(opt.value)
  )}
  getOptionLabel={(opt) => opt.label}
  isOptionEqualToValue={(opt, val) => opt.value === val.value}
  onChange={(_e, selected) => {
    handleLocationsChange(selected.map((s) => s.value));
  }}
  placeholder="All Locations"
  size="small"
  disabled={locationsLoading || !!locationsError}
  loading={locationsLoading}
  renderTags={() => null}                              // suppress chips inside input
  disableCloseOnSelect
/>
```

**Chips row pattern** (below filter bar):
```tsx
{(filters.locations?.length ?? 0) > 0 && (
  <div className={styles.selectedLocationTags}>
    {(filters.locations || []).map((locationId) => {
      const option = locationOptions.find((opt) => opt.value === locationId);
      return (
        <RepoChip
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

**CSS** (`styles.module.css`):
```css
.selectedLocationTags {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  align-items: center;
}
```

---

## Applying the Pattern to Order Tracker

### How the Order Tracker differs

| Aspect | Intakes (MLID-1985) | Order Tracker (MLID-2025) |
|--------|---------------------|---------------------------|
| Filter component | Standalone `MuiStandaloneMultiSelect` in page JSX | MUI DataGrid built-in filter panel (opened via `showFilterPanel()`) |
| Filter state | `filters.locations: string[]` local state | `filterModel: { items: GridFilterItem[] }` via URL params |
| Columns with multi-select | 1 (location) | 5–8 `singleSelect` columns across tabs |
| Filter visibility | Always visible in the page header | Hidden inside a popover panel (user must open it) |

### Approach

The MLID-1985 pattern can be adapted for the Order Tracker by focusing on the **external chip representation** of active custom filters — not on replacing the DataGrid's internal filter panel UI:

1. **Read `filterModel.items`** from the existing URL-based state to know which custom filters are active
2. **Render chips below the filter bar** for each active filter item, using the same `Chip` + `CancelIcon` pattern from MLID-1985
3. **Each chip's `onDelete`** removes that item from `filterModel.items` and updates the URL state
4. **Chip label format**: `"{columnName}: {value}"` for single-value filters, or `"{columnName}: {value1}, {value2}"` for "is any of" multi-value filters
5. **The DataGrid filter panel itself stays as-is** — we add a CSS fix (`flex-wrap: wrap`) on the filter panel's Autocomplete tags container to prevent overflow inside the panel too

This way the user sees their active filters both:
- Inside the panel (with the CSS overflow fix)
- Outside the panel as chips (can remove filters without opening the panel)

---

## Resolved Decisions

1. **Chip label format**: Grouped per column. One chip per `filterModel.items` entry with label format `{columnName} {operator} "{value}"`. For multi-value ("is any of"), values are comma-joined. When more than 2 values exist, truncate to first 2 + `"+N more"`.

2. **"Clear Filter" behavior scope**: Unchanged — clears everything (quick search + quick filters + custom filters via the existing `clearModels()`). The text just becomes red and reads "Clear all filters" when custom filters are active.

3. **Filter icon highlight style**: Change icon color to teal (`#00695C`) when any custom filter is active. The color matches the overall palette of active states.

4. **Date formatting in chip labels**: Format ISO date strings (matching `YYYY-MM-DD...`) as `M/D/YYYY`. Non-date and malformed strings pass through unchanged.

5. **Multi-select overlap fix**: CSS-only fix on `packages/ui/src/components/DataGrid/DataGrid.styles.ts` targeting `.MuiDataGrid-filterForm .MuiAutocomplete-inputRoot` — `flexWrap: wrap` + `height: auto` + `minHeight: 40`. No `maxHeight` so the input grows freely. No component replacement needed.

6. **Chip styling**: Use design system props (`variant="soft"` + `color="success"`) instead of hardcoded styles. The target colors (`#edf4f1` bg, `#3a888c` border, `#2e696d` text) are already mapped in `Chip.styles.ts:36`. Active quick filter chips and active custom filter chips both use this combo.

7. **Delete icon size**: Fixed at the design system level in `Chip.styles.ts` — added `& .MuiChip-deleteIcon { fontSize: iconPx }` to match the left-side icon size (16px for `sm`, 20px for `md`). Previously MUI's default 22px made `sm` chips look unbalanced. The `li-ui-playground` uses a hardcoded 14px `CancelIcon` — we opted for `iconPx` consistency across our design system instead.

---

## Implementation Steps

### Commit 1 — `1fa9c8cd` — Restructure filter header layout

**File**: `apps/web/app/orders-tracker/[category]/page.tsx`

- Move `TextField` (search) to the left as first element with `Layout.Item grow="1"`
- Add "Quick Filters:" label (`Typography bodySmall`, color `#6B6B74`) between search and quick filter chips
- Move quick filter chips into their own `Layout direction="row" gap={3}`
- Replace "Clear Filters" `Chip` with `Typography bodySmall` wrapped in a clickable `span` with `role="button"`, `tabIndex={0}`, and `onKeyDown` for Enter/Space

**Tests added** (`apps/web/app/orders-tracker/[category]/page.test.tsx`, 5 tests):
- Search field renders
- "Quick Filters:" label renders
- "Clear Filters" is a `span role="button"`, not a `Chip`
- Quick filter chips render
- Clicking "Clear Filters" calls `clearModels`

### Commit 2 — `80fddc2d` — Custom filter visibility

**File**: `apps/web/app/orders-tracker/[category]/page.tsx`

Added:
- `hasActiveCustomFilter = filterModel.items.length > 0`
- `activeColumns` — resolves the current tab's columns (`newOrdersColumns` / `maintenanceOrdersColumns` / `leadOrdersColumns`)
- `removeFilterItem(index)` — removes an item from `filterModel.items` and clears `activeQuickFilter`
- `formatFilterChipLabel(item)` — builds a human-readable label from `field`, `operator`, `value`

JSX changes:
- Filter `IconButton` gets `data-testid="filter-icon-button"` and `style={{ color: '#00695C' }}` when active
- "Clear Filters" → "Clear all filters" with `color: '#D32F2F'` when active
- New `Layout direction="row" justify="end" wrap="wrap" data-testid="active-filter-chips"` row below the header, rendered only when `hasActiveCustomFilter`. Renders one `Chip` per filter item with `onDelete` wired to `removeFilterItem`.

**Tests added** (7 tests):
- Filter icon highlighted when active
- Filter icon not highlighted when inactive
- "Clear all filters" red text when active
- "Clear Filters" default color when inactive
- Chips render for each filter item
- No chips row when no filters
- Chip delete removes the correct filter item from `filterModel.items`

### Commit 3 — `c4f3db12` — DataGrid multi-select chips vertical expansion

**File**: `packages/ui/src/components/DataGrid/DataGrid.styles.ts`

Add CSS override inside `.MuiDataGrid-filterForm`:

```ts
'& .MuiInputBase-root.MuiAutocomplete-inputRoot': {
  flexWrap: 'wrap',
  height: 'auto',
  minHeight: 40,
},
```

This lets the "is any of" multi-select Autocomplete input grow vertically so all selected chips are visible, instead of piling up in a fixed-height container.

**No unit tests** — purely CSS on a shared component. Visual verification only.

### Commit 4 — `10f1b167` — Design system Chip variants for filter colors

**File**: `apps/web/app/orders-tracker/[category]/page.tsx`

Replace hardcoded `style={{ borderColor, color, backgroundColor }}` with design system props:

- **Quick filter chips**: `variant={active ? 'soft' : 'outlined'}` + `color={active ? 'success' : undefined}`
- **Active custom filter chips**: `variant="soft"` + `color="success"`

The colors come from the `success` + `soft` entry in `Chip.styles.ts:36` (`#edf4f1` / `#3a888c` / `#2e696d`).

**No new tests** in this commit — the visual style is exercised by the rendering tests from commits 1 and 2. Styling assertions added later in commit 7.

### Commit 5 — `e6476a54` — Date formatting, operator mapping, delete icon sizing

**Files**:
- `apps/web/app/orders-tracker/[category]/page.tsx`
- `packages/ui/src/components/Chip/Chip.styles.ts`

`formatFilterChipLabel` updates:

- Operator label map: `isAnyOf → "is any of"`, `is → "is"`, `not → "is not"`, `onOrAfter → "after"`, `onOrBefore → "before"`, `after → "after"`, `before → "before"`.
- New helper `formatDateIfISO(value)` — detects strings starting with `YYYY-MM-DD`, parses via `new Date()`, formats as `M/D/YYYY`. Safe fallback on `isNaN(d.getTime())` returns the raw value.
- Apply `formatDateIfISO` to the final value string before composing the label.

`Chip.styles.ts`:
```ts
// Note: li-ui-playground uses a hardcoded deleteIcon at 14px via
// <CancelIcon style={{ fontSize: 14 }} />. We keep iconPx (16/20)
// for consistency with the left-side icon in our design system.
'& .MuiChip-deleteIcon': {
  fontSize: iconPx,
},
```

**No tests** in this commit — coverage added in commit 7.

### Commit 6 — `6b75319a` — Truncate multi-value filter chip labels

**File**: `apps/web/app/orders-tracker/[category]/page.tsx`

Update `formatFilterChipLabel`:

- When `item.value` is an array:
  - Filter out empty strings and `null` (regression fix: `activeOrdersByDueDate` quick filter seeds the array with a leading `undefined`)
  - If more than 2 non-empty values remain → show first 2 joined + `+N more` (e.g., `"Not Started, Working this Week +4 more"`)
  - Otherwise → join all remaining values

Constant: `MAX_VISIBLE_VALUES = 2`

**No tests** in this commit — coverage added in commit 7.

### Commit 7 — `592ef860` — Tests for label formatting and styling

**File**: `apps/web/app/orders-tracker/[category]/page.test.tsx`

Added 12 tests:

**Filter chip label formatting** (10 tests):
- Date formatting (3): ISO date → `M/D/YYYY`; non-date string unchanged; malformed date fallback
- Operator labels (3): `onOrAfter → "after"`, `onOrBefore → "before"`, `not → "is not"`
- Multi-value truncation (4): ≤2 values joined; 3+ values truncated with `+N more`; empty/null values filtered before truncating; filtered values without truncation path

**Quick filter chip styling** (2 tests):
- Active chip renders with `backgroundColor: rgb(237, 244, 241)` (soft success)
- Inactive chip does not have that background

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/app/orders-tracker/[category]/page.tsx` | Modify | Restructure header, add `hasActiveCustomFilter`/`activeColumns`/`removeFilterItem`/`formatFilterChipLabel`/`formatDateIfISO`, render filter chips row, switch to design system Chip variants |
| `apps/web/app/orders-tracker/[category]/page.test.tsx` | Modify | 24 new tests (5 header layout + 7 visibility + 10 label formatting + 2 chip styling) |
| `packages/ui/src/components/DataGrid/DataGrid.styles.ts` | Modify | CSS override for multi-select filter input vertical expansion |
| `packages/ui/src/components/Chip/Chip.styles.ts` | Modify | Add `.MuiChip-deleteIcon fontSize: iconPx` for consistent icon sizing |

---

## Testing Strategy

- **Unit tests** (47 total, 35 existing + 12 new + 12 prior in commits 1–2):
  - Filter header layout: search, label, "Clear Filters" text, quick filter chips rendering, clear action
  - Custom filter visibility: icon highlight, red "Clear all filters", chip rendering, delete behavior
  - Label formatting: dates, operators, multi-value truncation, empty filtering
  - Chip styling: soft success background for active chips

- **Manual verification**:
  1. Open `/orders-tracker/new` — verify header layout (search left, "Quick Filters:" label, chips, "Clear Filters" text, icons right)
  2. Click a quick filter — verify soft/success styling applied, chip appears below header
  3. Open custom filter panel (filter icon) — select "Order Status is any of" with 6 values — verify multi-select chips wrap vertically and all are visible
  4. Close panel — verify external chip shows `Order Status is any of "Not Started, Working this Week +4 more"` with correct soft/success styling and 16px delete icon
  5. Apply the "Orders Updated Last 7 Days" quick filter — verify chip shows `lastUpdated after "M/D/YYYY"` (formatted date, "after" operator)
  6. Click a chip's delete icon — verify that filter is removed but other filters remain
  7. Click "Clear all filters" — verify all filters clear

---

## Security Considerations

- **Input validation**: No new API endpoints or user input paths. Filter values are already validated by MUI DataGrid.
- **Authorization**: No changes — Order Tracker page access already gated by existing auth.
- **PHI handling**: Not applicable — filter columns are not PHI fields.
