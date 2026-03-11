# MLID-1780 — Intakes table: search input loses focus after each keystroke

## Task Reference

- **Jira**: [MLID-1780](https://localinfusion.atlassian.net/browse/MLID-1780)
- **Story Points**: 2
- **Branch**: `fix/MLID-1780-intakes-search-focus`
- **Base Branch**: `develop`
- **Status**: In Progress (QA follow-ups)

---

## Summary

The search input on the Intakes page ("Search by patient name or ID...") loses focus after typing a single character, forcing the user to click the field again for each keystroke. This is a React re-rendering bug.

---

## Codebase Analysis

### Root Cause

In `apps/web/app/intakes/page.tsx`, two inner "components" are defined as **arrow functions inside the page component**:

```typescript
// Line 750 — redefined on every render
const Controls = () => (
  <div className={styles.controls}>
    <StyledInput value={filters.search} onChange={handleSearchChange} ... />
    ...
  </div>
);

// Line 845 — same pattern
const TableSection = () => ( ... );
```

These are then used as JSX components:
```tsx
<Controls />    // Line 955 (split view)
<Controls />    // Line 1025 (single view)
<TableSection /> // Line 966, 1036
```

**Why this causes focus loss:**

1. User types a character → `handleSearchChange` fires → `updateFilters()` calls `setFilters()` + `updateUrl()` + `loadIntakes()`
2. State update → page component re-renders → `Controls` function is **redefined** (new reference)
3. React sees `<Controls />` pointing to a **different function** → treats it as a new component type → **unmounts the old DOM and mounts new DOM**
4. New `<input>` element doesn't have focus → focus is lost

The `StyledInput` component itself is fine — it's a simple `forwardRef` wrapper around `<input>`. The problem is the container being recreated.

### Other findings

- `TableSection` has the same pattern but doesn't cause a user-visible bug (no focusable inputs)
- Both are used in two places (split view vs single view) — that's why they were extracted
- Existing test file at `apps/web/app/intakes/page.test.tsx` has comprehensive tests

---

## Implementation Steps

### Step 1 — Add regression tests (RED)

- **Files**: `apps/web/app/intakes/page.test.tsx`
- **What**: Add two failing tests:
  1. **Focus retention**: Type multiple characters into the search input and assert focus is maintained throughout
  2. **Debounced API call**: Type quickly, assert `fetchIntakes` is NOT called on every keystroke but only after a 500ms pause

### Step 2 — Convert `Controls` and `TableSection` from component functions to JSX variables

- **Files**: `apps/web/app/intakes/page.tsx`
- **What**: Change the pattern from function-as-component to plain JSX stored in a variable:

**Before (broken):**
```tsx
const Controls = () => (
  <div className={styles.controls}>...</div>
);
// used as: <Controls />
```

**After (fixed):**
```tsx
const controls = (
  <div className={styles.controls}>...</div>
);
// used as: {controls}
```

This eliminates the component identity change on re-render. React will reconcile the JSX tree normally and preserve the DOM elements (including focus).

Apply the same change to `TableSection` → `tableSection` for consistency since it has the same anti-pattern.

### Step 3 — Add debounced search (500ms)

- **Files**: `apps/web/app/intakes/page.tsx`
- **What**: Follow the established precedent from `hospital-system-agreements/page.tsx`:
  1. Add a separate `searchInput` local state for immediate UI updates (keeps input responsive)
  2. Add a `searchTimerRef` (`useRef<NodeJS.Timeout | null>`)
  3. Rewrite `handleSearchChange`: update `searchInput` immediately, debounce `updateFilters()` by 500ms
  4. Bind the search input's `value` to `searchInput` instead of `filters.search`
  5. Sync `searchInput` from URL params on initialization
  6. Clean up timer on unmount

This ensures:
- Input stays responsive (no lag on typing)
- API calls + URL updates only fire once the user pauses typing for 500ms
- Follows the same pattern already used in the codebase

### Step 4 — Verify the fix passes tests (GREEN + REFACTOR)

- Run the new regression tests + all existing tests to confirm everything passes

### Step 5 — Enable search by WeInfuse patient ID ✅

- **Commit**: `193ec2bf`
- **Files**: `apps/web/app/api/intakes/route.ts`, `apps/web/app/api/intakes/route.test.ts`
- **What**: When the search input is numeric-only, add an exact match on `patient.patientId` alongside the name regex conditions. Non-numeric input is rejected for patient ID search to prevent injection.
- **Logic**:
  ```typescript
  const isNumeric = /^\d+$/.test(search);
  query.$or = isNumeric
    ? [...nameConditions, { 'patient.patientId': search }]
    : nameConditions;
  ```
- **Tests added**: 4 tests — numeric includes patientId, non-numeric excludes it, injection rejected, exact match (not regex)

### Step 6 — Full name search (concat firstName + lastName) ✅

- **Files**: `apps/web/app/api/intakes/route.ts`, `apps/web/app/api/intakes/route.test.ts`
- **Problem**: Searching "Kristen Johnson" returned 0 results because firstName and lastName are separate fields. The regex `"Kristen Johnson"` matches neither `"Kristen"` nor `"Johnson"` individually.
- **Fix**: When search contains multiple words, use `$expr` with `$concat` to build a virtual `"firstName lastName"` field and regex-match the search string against it:
  ```typescript
  query.$expr = {
    $regexMatch: {
      input: { $concat: ['$patient.firstName', ' ', '$patient.lastName'] },
      regex: search,
      options: 'i',
    },
  };
  ```
- **Tests added**: 2 tests — full name concat match, name order respected

#### Search behavior summary

| Search input | Matches | Strategy |
|---|---|---|
| `"Kristen"` | Any patient with "Kristen" in firstName or lastName | Single word → `$or` regex on each field |
| `"Kristen Johnson"` | Only "Kristen Johnson" | Multi-word → `$expr` concat `firstName + " " + lastName` |
| `"Johnson Kristen"` | No match for "Kristen Johnson" | Order is respected |
| `"12345"` | Name regex + exact patientId match | Numeric → `$or` includes `patient.patientId` |

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/app/intakes/page.test.tsx` | Modify | Add regression tests for search focus retention and debounce |
| `apps/web/app/intakes/page.tsx` | Modify | Convert `Controls`/`TableSection` to JSX variables, add debounced search |
| `apps/web/app/api/intakes/route.test.ts` | Modify | Add tests for patient ID search and full name search |
| `apps/web/app/api/intakes/route.ts` | Modify | Add patient ID exact match for numeric search, concat full name search |

---

## Testing Strategy

- **Focus regression test**: Type multiple characters via `userEvent.type()` and assert the input retains focus and full value
- **Debounce test**: Type quickly, assert `fetchIntakes` is not called during typing, then advance timers by 300ms and assert it's called once with the final value
- **Patient ID search tests**: Numeric input includes `patient.patientId` exact match, non-numeric excluded, injection rejected
- **Full name search tests**: Multi-word input uses `$expr` concat match, name order respected
- **Existing tests**: All existing intakes page tests must continue to pass
- **Manual verification**: Open Intakes page → click search → type a full name → field keeps focus, API call fires only after 300ms pause, results match

---

## Security Considerations

- **Input validation**: No change — search input already goes through existing filter logic
- **Authorization**: No change — page-level auth unchanged
- **PHI handling**: No change — search queries are handled by the existing service layer

---

## Design Decision: JSX Variables vs. Extracted Components

Other pages in this codebase (e.g., `/intakes/[id]/review`) follow the pattern of extracting reusable UI blocks into standalone React components in a `components/` folder. We evaluated doing the same here but determined that **JSX variables are the more cost-effective fix** for this bug.

### Why JSX variables win here

- The original bug was caused by defining `Controls` and `TableSection` as arrow functions inside the page component — making them new component identities on every render. Converting them to JSX variables eliminates the identity problem with zero prop-passing overhead.
- Extracting them into proper components would fix the bug equally well (stable function reference in a separate module), but requires defining large props interfaces and threading state + callbacks through — effort that doesn't add value for this fix.

### What a component extraction would require

#### `IntakesControls` — 13 props

| Prop | Type | Source in page |
|------|------|----------------|
| `searchInput` | `string` | state |
| `onSearchChange` | `ChangeEventHandler<HTMLInputElement>` | handler |
| `filters` | `IntakesListParams` (status, type, locations, category, hasErrors, excludeSkipped) | state |
| `onStatusChange` | `(option: { value: string; label: string } \| null) => void` | handler |
| `onTypeChange` | `(option: { value: string; label: string } \| null) => void` | handler |
| `onLocationsChange` | `(values: string[]) => void` | handler |
| `onCategoryChange` | `(option: { value: string; label: string } \| null) => void` | handler |
| `onErrorsToggle` | `ChangeEventHandler<HTMLInputElement>` | handler |
| `onExcludeSkippedToggle` | `ChangeEventHandler<HTMLInputElement>` | handler |
| `locationOptions` | `{ value: string; label: string }[]` | derived (useMemo) |
| `locationsLoading` | `boolean` | state |
| `locationsError` | `string \| null` | state |
| `isSplitView` + `onSplitViewToggle` | `boolean` + `(checked: boolean) => void` | state |

#### `IntakesTable` — ~8 props

| Prop | Type | Source in page |
|------|------|----------------|
| `intakes` | `IIntakeWithLocation[]` | state |
| `selectedIntakeId` | `string \| undefined` | viewerState |
| `onRowClick` | `(id: string) => void` | handler |
| `tableWrapperRef` | `RefObject<HTMLDivElement>` | ref |
| `pagination` | `{ page, pageSize, totalCount, totalPages }` | state |
| `onPageChange` | `(page: number) => void` | handler |
| `onPageSizeChange` | `(size: number) => void` | handler |
| `maskPatientName` / `maskDateOfBirth` | PHI masking functions | usePHIMasking hook |

Helper functions (`formatDate`, `formatPatientName`, `getTypeLabel`) and sub-components (`AttachmentIndicator`, `CategoryTags`, `StatusLabel`, `ErrorIndicator`) would be imported directly by the new component.

### Conclusion

A full component extraction is a **low-risk but medium-effort refactor** (~30-45 min including test updates). It would align this page with patterns used elsewhere in the codebase but is out of scope for this bug fix. It could be tackled as a follow-up refactor if desired.

---

## Open Questions

None — the fix is straightforward.
