# How the New Orders Query Works

> Analysis of the route `/orders-tracker/[category]` for new orders

---

## 1. Entry Point: UI → API

The page at `/orders-tracker/[category]` uses an `OrdersTrackerProvider` context that calls:

```
GET /api/orders/new?page=1&pageSize=100&sort=[...]&filter={...}&fieldTypes={...}&search=...
```

The API route (`app/api/orders/new/route.ts:31-57`) parses these params and passes them to `getOrders()`.

---

## 2. Parameter Conversion (gridToMongoQuery.ts)

Before reaching MongoDB, MUI DataGrid params are converted:

- **Sort**: `gridSortToMongoQuery()` — converts `[{ field: "received", sort: "desc" }]` → `[{ field: "received", order: -1 }]`
- **Filter**: `gridFilterToMongoQuery()` — converts each DataGrid filter item into a `(field: string) => Document` builder function. Supports operators like `contains`, `equals`, `isEmpty`, `isAnyOf`, date comparisons, etc. The result is `{ logic: '$and' | '$or', queryBuilders: [...] }`.

---

## 3. Query Orchestration (ordersTracker.ts:815-920 — `getOrders()`)

`getOrders()` builds the final aggregation pipeline in this order:

```javascript
db.collection('neworders').aggregate([
  ...userFilter,          // Step A: role-based access
  ...searchFilter,        // Step B: text search (optional)
  ...additionalFilters,   // Step C: extra filters (rarely used)
  ...getOrdersAggregate({ category: 'new', page, pageSize, sort, filter }),
])
```

**Step A — `getViewOrdersFilter(user)`** (`types/orders.ts:224-250`):
Role-based pre-filter applied FIRST, before anything else.

| User Role | Filter |
|-----------|--------|
| No positions | `{ _id: { $in: [] } }` — matches nothing |
| Admin/Auditor (viewAll) | No filter — sees everything |
| Location-based roles | `{ site: { $in: [user.locations] } }` |
| AssignedTo-based roles | `{ assignedTo: user._id }` |
| Location + AssignedTo | `$or` of both |

**Step B — Search filter** (ordersTracker.ts:839-896):
If `searchQuery` is provided, runs 3 parallel pre-queries:
1. `patients` — match by `full_name` (regex) or `we_infuse_id_IV` (exact number)
2. `lidrugs` — match by `name` (regex)
3. `libiosimilars` — match by `name` (regex), extracts `drugs[].li_drug_id`

Then builds a `$match: { $or: [...] }` that checks `displayId`, `patient`, or `drug` against the found IDs.

---

## 4. Pipeline Construction (ordersTracker.ts:330-769 — `getOrdersAggregate()`)

This is the core. For paginated requests it builds:

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. $match: deleted filter                                       │
│    Excludes orders where deleted == true                        │
├─────────────────────────────────────────────────────────────────┤
│ 2. providerOfficeAlias                                          │
│    $addFields: { providerOffice: normalizedFax }                │
│    (creates a sortable/filterable fax alias on raw documents)   │
├─────────────────────────────────────────────────────────────────┤
│ 3. lookupStage (DYNAMIC — from getFilterAndSortAggregateQuery)  │
│    Only added when user filters/sorts on related fields         │
│    e.g., filtering by "patient.full_name" triggers a $lookup    │
│    on the patients collection BEFORE facet                      │
├─────────────────────────────────────────────────────────────────┤
│ 4. filterStage (DYNAMIC)                                        │
│    $match with the converted grid filters                       │
│    Uses $and or $or based on filter.logic                       │
├─────────────────────────────────────────────────────────────────┤
│ 5. $facet                                                       │
│    ┌─────────────────────────────────────────────────────────┐  │
│    │ data:                                                   │  │
│    │   setSortByDisplayIdPairStage (order protocol pairing)  │  │
│    │   sortStage ({ received: -1 } by default)               │  │
│    │   $skip: (page - 1) * pageSize                          │  │
│    │   $limit: pageSize                                      │  │
│    │   expandIdsAggregate (ALL relationship lookups)          │  │
│    ├─────────────────────────────────────────────────────────┤  │
│    │ totalCount:                                             │  │
│    │   $count: 'count'                                       │  │
│    └─────────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│ 6. unsetStage — removes temp fields (sortFilterObj_*, etc.)     │
├─────────────────────────────────────────────────────────────────┤
│ 7. $project: { data: 1, total: $arrayElemAt totalCount }        │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. Dynamic Lookup System (ordersTracker.ts:28-327 — `getFilterAndSortAggregateQuery()`)

This is the clever part. It builds lookups **only when needed** for filtering/sorting on related fields.

**Collection mapping** (line 66-85):
```
assignedTo     → users (by _id)
site           → patientLocations (by _id)
patient        → patients (by _id)
drug           → lidrugs (by _id)
provider       → cmsnpiproviders (by npi)
providerOffice → provideroffices (by faxNumbers)
updatedBy      → users (by _id)
```

When the user filters on e.g. `patient.full_name`:
1. The field is split into `parentField = "patient"`, `subField = "full_name"`
2. A `$lookup` is generated to join `patients` by converting the string ID to ObjectId
3. The looked-up value is stored in a temp field `sortFilterKey_N_M`
4. The filter's `$match` references this temp field
5. After `$facet`, the temp field is cleaned up via `$unset`

**Deduplication**: If both filter and sort reference the same collection (e.g., `patient.full_name` filter + `patient.full_name` sort), only one `$lookup` is generated.

**Concat handling**: When filtering on multi-part fields like `provider.first_name,provider.last_name`, it concatenates the looked-up values into a single string for matching.

---

## 6. expandIdsAggregate (ordersTracker.ts:348-654)

This runs **inside `$facet.data` after `$skip/$limit`**, so it only processes the 25-100 orders on the current page. It expands all string IDs into full documents:

| # | Lookup | From Collection | Join Field | Result |
|---|--------|----------------|------------|--------|
| 1 | updatedByUser | `users` | `updatedBy` → `_id` | User document |
| 2 | assignedToUser | `users` | `assignedTo` → `_id` | User document |
| 3 | site | `patientLocations` | `site` → `_id` | Location document |
| 4 | patient | `patients` | `patient` → `_id` | Patient document (excludes heavy arrays) |
| 5 | drug | `lidrugs` + `libiosimilars` | `drug` → `_id` | Drug + biosimilar document |
| 6 | orderProtocol | `orderprotocols` | `orderProtocol` → `_id` | Protocol document |
| 7 | provider | `cmsnpiproviders` | `provider` → `npi` | Provider document (falls back to raw NPI string) |
| 8 | providerOffice | `provideroffices` | `providerPracticeFax` → normalized `faxNumbers` | Office name |

Each lookup follows the pattern: `$addFields` (convert string to ObjectId) → `$lookup` → `$unwind` (preserveNullAndEmptyArrays).

---

## 7. Final Output

```typescript
{
  data: FullOrder<'new'>[],  // page of orders with all relationships expanded
  total: number              // total count for pagination
}
```

---

## Key Takeaway for Eligibility

The **eligibility stages** would be appended at the end of `expandIdsAggregate` (step 6), after all existing lookups. At that point `$patient`, `$drug`, `$site`, and `$provider` are already resolved documents, so the eligibility `$lookup` stages (hospitalSystems, patient_insurances, insurance_plans) can reference their fields directly. And since this runs after `$skip/$limit`, only the page of orders gets the extra lookups.

---

## Key Files

| File | Lines | Purpose |
|------|-------|---------|
| `app/api/orders/new/route.ts` | 31-57 | API route — parses params, calls `getOrders()` |
| `utils/api/orders/gridToMongoQuery.ts` | 288-334 | Converts MUI DataGrid filter/sort to MongoDB format |
| `utils/api/orders/gridToMongoQuery.ts` | 68-286 | `itemToMongoPredicate()` — individual filter operators |
| `services/mongodb/ordersTracker.ts` | 28-327 | `getFilterAndSortAggregateQuery()` — dynamic lookup building |
| `services/mongodb/ordersTracker.ts` | 330-769 | `getOrdersAggregate()` — main pipeline factory |
| `services/mongodb/ordersTracker.ts` | 815-920 | `getOrders()` — orchestrates pipeline + user filters + search |
| `types/orders.ts` | 224-250 | `getViewOrdersFilter()` — role-based access filter |
| `utils/constants/collections.ts` | 1-36 | Collection name constants |
