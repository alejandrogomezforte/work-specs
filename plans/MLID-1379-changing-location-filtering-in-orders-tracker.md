# MLID-1379: Changing Location Filtering in Orders Tracker

## Overview

**Ticket:** MLID-1379 - Replace location by location_name
**Route:** `/orders-tracker`
**File:** `app/orders-tracker/page.tsx`
**Date:** 2026-01-26

This document describes the analysis and implementation plan for updating the location filter in the Orders Tracker page to:
1. Use location ID for filtering (instead of string comparison)
2. Display `location_name` with fallback to `city, state`

---

## Current Implementation Analysis

### Component Tree

```
app/orders-tracker/page.tsx (Main Page)
│
├── State Management
│   ├── filters.location (string) ─── stores "Commack, NY" string
│   └── locationFilters (string[]) ── ["Commack, NY", "Bedford, TX", ...]
│
├── Data Processing
│   └── prepareOrders() function (lines 143-168)
│       ├── siteString: `${order.site.city}, ${order.site.state}`
│       └── siteStringLower: for search functionality
│
├── Filtering Logic
│   └── filterOrders() function (lines 205-269)
│       └── locationMatch = order.siteString == filters.location (line 210)
│
├── Filter Options Generation
│   └── locationFilters useMemo (lines 289-297)
│       └── Creates unique strings: `${o.site.city}, ${o.site.state}`
│
└── UI Components
    └── Location Filter Dropdown (lines 339-351)
        └── <option key={location} value={location}>{location}</option>
```

### Current Code

#### 1. Filter State (line 133-138)
```typescript
const [filters, setFilters] = useState({
  location: '',  // ← stores string like "Commack, NY"
  search: '',
  receivedStartDate: '',
  receivedEndDate: '',
});
```

#### 2. prepareOrders Function (lines 143-168)
```typescript
function prepareOrders<K extends OrderCategory>(orders: FullOrder<K>[]) {
  return orders.map((order) => ({
    ...order,
    siteString: order.site ? `${order.site.city}, ${order.site.state}` : '',
    siteStringLower: order.site
      ? `${order.site.city}, ${order.site.state}`.toLowerCase()
      : '',
    // ... other fields
  }));
}
```

#### 3. Filter Options Generation (lines 289-297)
```typescript
const locationFilters = useMemo(() => {
  return [
    ...new Set(
      (tab === 'new' ? preparedNewOrders : preparedMaintenanceOrders).map(
        (o) => (o.site ? `${o.site.city}, ${o.site.state}` : '')
      )
    ),
  ].sort();
}, [tab, preparedNewOrders, preparedMaintenanceOrders]);
```

#### 4. Filtering Logic (lines 205-211)
```typescript
const filterOrders = useCallback(
  (preparedOrders, exactOrderMatches) => {
    const result = preparedOrders.filter((order) => {
      let locationMatch = true;
      if (filters.location) {
        locationMatch = order.siteString == filters.location;  // ← string comparison
      }
      // ...
    });
  },
  [filters, defferedSearchInput]
);
```

#### 5. Dropdown UI (lines 339-351)
```typescript
<select
  id="location-filter"
  value={filters.location}
  onChange={(e) => handleFilterChange('location', e.target.value)}
  className={styles.selectInput}
>
  <option value="">All Locations</option>
  {locationFilters.map((location) => (
    <option key={location} value={location}>
      {location}
    </option>
  ))}
</select>
```

---

## Problems with Current Approach

| Issue | Description | Impact |
|-------|-------------|--------|
| **Display coupled to filter value** | The same string is used for both display and filtering | Changing display to `location_name` breaks filtering |
| **String comparison** | Filtering uses string equality check | Less efficient, prone to formatting/whitespace issues |
| **No unique identifier** | Locations identified by city+state string | Two locations with same city/state would collide (edge case but possible) |
| **Fragile architecture** | Display format change requires updating multiple places | High maintenance cost, error-prone |
| **Inconsistent with best practices** | Should filter by ID, display by label | Violates separation of concerns |

---

## Recommended Solution

### Design Principle

**Filter by ID, Display by Label**

- Use `location._id` (MongoDB ObjectId) as the filter value
- Use `location_name || city, state` as the display label
- Decouple the filter mechanism from the display format

### Data Structure Change

**Before:** `locationFilters: string[]`
```typescript
["Commack, NY", "Bedford, TX", ...]
```

**After:** `locationFilters: LocationFilterOption[]`
```typescript
[
  { id: "507f1f77bcf86cd799439011", label: "Commack Infusion Center" },
  { id: "507f1f77bcf86cd799439012", label: "Bedford, TX" },  // fallback when no location_name
  ...
]
```

---

## Implementation Plan

### Step 1: Define Type for Location Filter Option

**File:** `app/orders-tracker/page.tsx` (or separate types file)

```typescript
interface LocationFilterOption {
  id: string;
  label: string;
}
```

### Step 2: Update prepareOrders Function

Add `siteId` field for filtering, update `siteString` for display with `location_name`:

```typescript
function prepareOrders<K extends OrderCategory>(orders: FullOrder<K>[]) {
  return orders.map((order) => ({
    ...order,
    siteId: order.site?._id || '',
    siteString: order.site
      ? (order.site.location_name || `${order.site.city}, ${order.site.state}`)
      : '',
    siteStringLower: order.site
      ? (order.site.location_name || `${order.site.city}, ${order.site.state}`).toLowerCase()
      : '',
    // ... other fields unchanged
  }));
}
```

### Step 3: Update locationFilters Computation

Change from string array to object array with id and label:

```typescript
const locationFilters = useMemo((): LocationFilterOption[] => {
  const uniqueLocations = new Map<string, LocationFilterOption>();

  (tab === 'new' ? preparedNewOrders : preparedMaintenanceOrders).forEach((o) => {
    if (o.site && o.site._id && !uniqueLocations.has(o.site._id)) {
      uniqueLocations.set(o.site._id, {
        id: o.site._id,
        label: o.site.location_name || `${o.site.city}, ${o.site.state}`,
      });
    }
  });

  return [...uniqueLocations.values()].sort((a, b) =>
    a.label.localeCompare(b.label)
  );
}, [tab, preparedNewOrders, preparedMaintenanceOrders]);
```

### Step 4: Update filterOrders Function

Change from string comparison to ID comparison:

```typescript
const filterOrders = useCallback(
  (preparedOrders, exactOrderMatches) => {
    const result = preparedOrders.filter((order) => {
      let locationMatch = true;
      if (filters.location) {
        locationMatch = order.siteId === filters.location;  // ← ID comparison
      }
      // ... rest unchanged
    });
    // ...
  },
  [filters, defferedSearchInput]
);
```

### Step 5: Update Dropdown UI

Use `id` for value and `label` for display:

```typescript
<select
  id="location-filter"
  value={filters.location}
  onChange={(e) => handleFilterChange('location', e.target.value)}
  className={styles.selectInput}
>
  <option value="">All Locations</option>
  {locationFilters.map((location) => (
    <option key={location.id} value={location.id}>
      {location.label}
    </option>
  ))}
</select>
```

---

## Test Cases (TDD)

### Unit Tests for prepareOrders

1. **Should include siteId from order.site._id**
2. **Should set siteString to location_name when available**
3. **Should fallback siteString to "city, state" when location_name is empty**
4. **Should fallback siteString to "city, state" when location_name is undefined**
5. **Should handle orders without site (siteId and siteString empty)**

### Unit Tests for locationFilters

1. **Should return array of LocationFilterOption objects**
2. **Should use location_name as label when available**
3. **Should fallback to "city, state" as label when location_name is empty**
4. **Should not include duplicate locations (by ID)**
5. **Should sort options alphabetically by label**
6. **Should handle orders without site (exclude from options)**

### Unit Tests for filterOrders

1. **Should filter orders by location ID**
2. **Should return all orders when location filter is empty**
3. **Should correctly match orders with the selected location ID**
4. **Should exclude orders that don't match the selected location ID**

### Integration Tests

1. **Dropdown displays location_name when available**
2. **Dropdown displays "city, state" fallback when location_name is empty**
3. **Selecting a location filters the table correctly**
4. **"All Locations" option shows all orders**
5. **Search still works with location_name in siteStringLower**

---

## Files to Modify

| File | Changes |
|------|---------|
| `app/orders-tracker/page.tsx` | Main implementation changes (5 areas) |
| `app/orders-tracker/page.test.tsx` | New test file (TDD) |

---

## Risk Assessment

| Risk | Level | Mitigation |
|------|-------|------------|
| Breaking existing filter functionality | Medium | Comprehensive test coverage before changes |
| Type errors with new LocationFilterOption | Low | TypeScript will catch issues at compile time |
| Performance impact from Map operations | Low | Map operations are O(1), minimal impact |
| Edge case: orders without site | Low | Already handled with conditional checks |

---

## Acceptance Criteria

- [ ] Location filter dropdown displays `location_name` when available
- [ ] Location filter dropdown falls back to "city, state" when `location_name` is empty/undefined
- [ ] Filtering by location works correctly using location ID
- [ ] Search functionality still includes location in search terms
- [ ] Both "New Orders" and "Maintenance Orders" tabs work correctly
- [ ] All existing functionality remains intact
- [ ] Tests pass with >80% coverage for changed code

---

## Related Changes in Branch

This change is part of the broader MLID-1379 effort. Related commits:

```
d04e190b feat(MLID-1379): display location_name in MuiLocationSelect dropdown
24fb015b feat(MLID-1379): display location_name in intake review page
23aebac7 feat(MLID-1379): display location_name in messages list
1d891f15 feat(MLID-1379): display location_name in intakes list
```

---

*Document created: 2026-01-26*
*Author: Development Team*
