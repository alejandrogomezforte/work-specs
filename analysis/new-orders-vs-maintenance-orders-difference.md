# New Orders vs Maintenance Orders — Collection Difference

## Purpose

Reference analysis comparing the `neworders` and `maintenanceorders` MongoDB
collections, captured while rethinking the MLID-2417 epic (Document
Notifications V2). The central question was whether the two order types are
structurally different enough to warrant different notification handling. The
short answer: **no — the difference is business workflow, not structure**, and
every field the notification system touches is shared identically by both.

Data captured 2026-06-18 from `local-infusion-db` (dev): 50-document schema
sample of each collection plus the latest live record from each.

- `neworders` — 4037 documents
- `maintenanceorders` — 3754 documents

---

## Code-layer findings

- There is **no dedicated maintenance Mongoose model**. The only model is
  `NewOrder` (`apps/web/models/NewOrder.ts`), and it is deliberately thin
  (`strict: false`, declaring only `patient`, `drug`, `deleted`, timestamps)
  because the full order shape is large and is otherwise accessed via the raw
  driver in `apps/web/services/mongodb/ordersTracker.ts`.
- Both collections are reached through one switch
  (`getOrdersCollection` in `ordersTracker.ts`) and run through the **same
  aggregation pipeline** (`getOrdersAggregate`) with only a `category`
  parameter:
  - `'new'` → `neworders`
  - `'maintenance'` → `maintenanceorders`
  - `'lead'` → `leadorders`
- Because every collection uses `strict: false`, neither enforces a fixed
  schema. Any field can be sparsely present, so the table below reflects what
  the sampled documents actually contain, not an enforced contract.

---

## Field comparison

Legend: ✅ = present, ✅* = present and carries a meaningful value as the
discriminator, — = not observed.

| Field | neworders | maintenanceorders |
|---|---|---|
| `_id` | ✅ ObjectId | ✅ ObjectId |
| `rowNumber` | ✅ Number | ✅ Number |
| `displayId` | ✅ String | ✅ String |
| `createdAt` | ✅ Date | ✅ Date |
| `lastUpdated` | ✅ Date | ✅ Date |
| `lastAssigned` | ✅ Date | ✅ Date |
| `createdBy` | ✅ String | ✅ String |
| `updatedBy` | ✅ String | ✅ String |
| `assignedBy` | ✅ String | ✅ String |
| `assignedTo` | ✅ String | ✅ String |
| `received` | ✅ Date | ✅ Date |
| `site` | ✅ String | ✅ String |
| `patient` | ✅ String | ✅ String \| Null |
| `drug` | ✅ String | ✅ String |
| `provider` | ✅ String | ✅ String |
| `providerPracticeFax` | ✅ String | ✅ String |
| `orderProtocol` | ✅ String | ✅ String |
| `drugSwitchContext` | ✅ Array | ✅ Array |
| `deleted` | ✅ Boolean | ✅ Boolean |
| `status` | ✅ String | ✅ String |
| `rtsOrNoGoDate` | ✅ Date | ✅ Date |
| `orderType` | ✅* String (e.g. `"New Start"`) | — |
| `changeType` | ✅ String (empty `""`) | ✅* String (e.g. `"Maintenance"`) |
| `dispenseAsWritten` | ✅ Boolean | — |
| `audit` | ✅ String | — |
| `weInfuseOrderSeriesId` | ✅ Number | — |
| `dueDate` | — | ✅ Date \| Null |
| `noGoContext` | — | ✅ Array |
| `notes` | — | ✅ String |
| `coverageStatus` | — | ✅ String |
| `authSubmissionDate` | — | ✅ Date |

---

## Main differences explained

### Shared core (rows 1–21)

The first 21 fields are identical in both collections. These are the order's
core identity and assignment fields, and they include **every field the
notification system reads** — namely `_id` and `assignedTo`. This is why the
notification orchestrator (`propagateDocumentNotificationToOrder` in
`apps/web/services/notifications/documentNotification.ts`) can treat both order
types the same: its `order` parameter is only `{ _id, assignedTo }`, and the
notification entity tuple is hardcoded `{ entityType: 'order', entityId }` with
no new-vs-maintenance distinction.

### The discriminator pair: `orderType` / `changeType`

The cleanest way to tell the two apart at the document level:

- **new orders** carry a meaningful `orderType` (e.g. `"New Start"`); their
  `changeType` is empty (`""`).
- **maintenance orders** carry a meaningful `changeType` (e.g. `"Maintenance"`)
  and have no `orderType`.

### Maintenance-only workflow fields

`dueDate`, `noGoContext`, `notes`, `coverageStatus`, `authSubmissionDate` exist
only on maintenance orders. They represent the re-verification / renewal
workflow for a therapy already in progress: coverage re-checks, prior-auth
resubmission dates, due dates, and a rich `noGoContext` array (insurance name,
provider NPI, WeInfuse order ids, no-go event details).

### New-orders-leaning fields

`dispenseAsWritten`, `audit`, `weInfuseOrderSeriesId` appeared on the live
`neworders` record but were not in the 50-document schema sample for either
collection — they are likely present on a subset of new orders. Treat them as
new-orders-leaning, not guaranteed-universal.

### `patient` nullability

`patient` is `String` on new orders but `String | Null` on maintenance orders —
maintenance orders can exist with a null patient reference.

---

## What this means semantically

- **neworders** = a brand-new therapy order — the patient's first start on a
  drug (`orderType: "New Start"`). Carries new-start specifics like
  `weInfuseOrderSeriesId`, `audit`, `dispenseAsWritten`.
- **maintenanceorders** = continuation / recurring order for therapy already in
  progress (`changeType: "Maintenance"`). Carries the renewal/re-verification
  workflow fields (`coverageStatus`, `authSubmissionDate`, `dueDate`,
  `noGoContext`).

The difference is **business workflow, not structure**. The two share every
field that matters to notifications.

---

## Relevance to MLID-2417 (notifications rethink)

- The type-agnostic `entityType: 'order'` notification design holds up: there is
  no notification-relevant difference between the two collections, only a
  workflow difference that lives elsewhere in the app.
- The one practical consequence of having two separate collections (rather than
  one collection with a discriminator field) is that, given only an order `_id`
  from a notification, the code cannot tell which collection the order lives in
  without either a lookup-guess (`getOrderById(id, 'new') ?? getOrderById(id,
  'maintenance')`) or carrying the category alongside the id. This is exactly
  the friction the link-orders category pass-through work removes on the write
  path.
