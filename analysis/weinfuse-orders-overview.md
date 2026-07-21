# WeInfuse Orders — Patient Orders Tab Overview

Analysis of the patient Orders tab at route `/patient/[id]/orders` — the LISA vs WeInfuse
side-by-side reconciliation table with the "Show Unlinked WeInfuse" / "Show Unlinked LISA"
quick filters.

Scope: **Postgres only**. Schemas live in `packages/db`; conventions are documented in
`packages/db/CLAUDE.md`.

## Where the code lives

| Layer | Path | Notes |
| --- | --- | --- |
| Page (route) | `apps/web/app/patient/[patientId]/orders/page.tsx` | Thin server component; gated by feature flag `PATIENT_ORDERS_TAB`, calls `notFound()` when the flag is off |
| UI component | `apps/web/components/Modules/patient/components/patientOrders/patientOrders.tsx` | ~640 lines — the LISA/WeInfuse split table, quick filters, order search, link/unlink modals |
| Styles | `apps/web/components/Modules/patient/components/patientOrders/styles.module.css` | CSS Module (legacy app-level styling, not `@repo/ui`) |
| Data hook | `apps/web/components/Modules/patient/components/patientOrders/hooks/usePatientOrders.ts` | Fetching, pagination and filter state for the table |
| API — list | `apps/web/app/api/patient/[patientId]/orders/route.ts` | Returns the reconciled LISA + WeInfuse order rows |
| API — link | `apps/web/app/api/patient/[patientId]/orders/[weInfuseOrderId]/link/route.ts` | Link / unlink a WeInfuse order (by order series) to a LISA order |
| Service | `apps/web/services/mongodb/patientOrders.ts` | Assembles the two halves of the table. The directory name is legacy — the reads inside are routed to Postgres |
| Ingest — endpoint | `apps/web/pages/api/webhooks/looker_orders.ts` | `POST /api/webhooks/looker_orders`; how WeInfuse orders enter the system |
| Ingest — handler | `apps/web/services/webhooks/handlers/looker_orders.ts`, `.../handlers/util/webhookUpsertOrders.ts` | Transform, auto-linking, dedup |
| Postgres schema — LISA | `packages/db/src/lisaOrders/schema/lisaOrders.ts` | `lisa_orders` table |
| Postgres schema — WeInfuse | `packages/db/src/orders/schema/orders.ts` | `orders` table |
| Postgres repositories | `packages/db/src/lisaOrders/repository.ts`, `packages/db/src/orders/repository.ts` | The `*ForRead` query methods the tab consumes |
| Migrations | `packages/db/migrations/NNNN_*.sql` | drizzle-kit output; config in `packages/db/drizzle.config.ts` |
| Tests | `patientOrders.test.tsx`, `hooks/usePatientOrders.test.ts`, `app/api/patient/[patientId]/orders/route.test.ts` | Component, hook and route coverage |
| Parity tests | `apps/web/services/mongodb/__tests__/lisaOrders.{activeIds,count,find}.parity.test.ts`, `patientOrders.e2e.parity.test.ts` | Prove each read primitive returns identical results before the cut-over |

## Ownership

**Ruslan Kryvosheiev is the primary owner.** He created the feature and owns the large
majority of the lines:

| File | Ruslan | Alberto Beltran |
| --- | --- | --- |
| `patientOrders.tsx` | 627 lines | 15 lines |
| `services/mongodb/patientOrders.ts` | 664 lines | 82 lines |

Commit counts across the component, API routes and service: Ruslan Kryvosheiev 31,
Alberto Beltran 12, Alejandro Gomez 3, Anatoliy Kolodkin 1, claude[bot] 1.

- Architecture and query questions → **Ruslan**
- WI Order ID column and Revised-order suppression → **Alberto**

## Timeline

| Date | Ticket | Author | Change |
| --- | --- | --- | --- |
| 2026-03-05 / 03-06 | MLID-1190 | Ruslan Kryvosheiev | Original build — "feat(patient): add Patient Orders tab with feature flag protection", plus pagination, security checks, query optimization, UX follow-ups, and the link / unlink modals |
| 2026-04-10 | (no ticket) | Ruslan Kryvosheiev | Linked-orders display improvements, drug name extraction utility for linked orders |
| 2026-04-13 | MLID-2057 | Ruslan Kryvosheiev | Replaced `isValidObjectId` with `isObjectIdString` to fix a `bson.ts` Jest transform error |
| 2026-04-23 | MLID-2164 | Alberto Beltran | Link Order modal redesign; API accepts a WeInfuse id; guard against unsafe integer overflow in the WeInfuse id fallback |
| 2026-04-01 | MLID-2049 | Ruslan Kryvosheiev | Keep filter tag text visible on hover when active |
| 2026-04-06 | MLID-2026 | Ruslan Kryvosheiev + claude[bot] | Keep the search input mounted during loading to prevent focus loss, plus a regression test |
| 2026-05-05 | MLID-2282 | Ruslan Kryvosheiev | Hide deleted LISA orders; handle mixed-form patient links; extract the unlinked-WeInfuse helper; share query mock utilities |
| 2026-05-06 / 05-12 | MLID-2346 | Alberto Beltran | Added the **WI Order ID column** with Revised-order row suppression; unique per-row `data-testid`; hide the entire Revised WI row instead of nulling the orderId |
| 2026-05-14 | MLID-1345 | Alberto Beltran | `parseWeInfuseId` utility for safely parsing IV-prefixed WeInfuse ids; route tests for missing / invalid WeInfuse id |
| 2026-05-21 | MLID-2432 | Ruslan Kryvosheiev | Link / unlink applied **by order series**; idempotent unlink; clearer 404 when a series has no matched orders |
| 2026-07-01 / 07-03 | MLID-364 | Alejandro Gomez | Next 16 + React 19 upgrade and NextAuth v4 → Auth.js v5 migration — touched the API route mechanically only (async params, auth mocks) |
| 2026-07-14 | MLID-2662 Part 2 (PR #1900) | Ruslan Kryvosheiev | Most recent functional change — routed orders, LISA and order-protocol reads to Postgres |

## The two tables

The table on screen has two halves, each backed by a different Postgres table. They are
assembled in memory by `getPatientOrders` — there is no single SQL statement that joins
them.

| Half of the table | Postgres table | Drizzle schema |
| --- | --- | --- |
| LISA (left) | `lisa_orders` | `packages/db/src/lisaOrders/schema/lisaOrders.ts` |
| WeInfuse (right) | `orders` | `packages/db/src/orders/schema/orders.ts` |
| LISA `Drug` column value | `li_drugs` | joined through `lisa_orders.li_drug_id` |

Two naming traps:

- The table called **`orders` holds WeInfuse orders**, not LISA orders.
- **`lisa_orders` is one table for all three LISA order kinds**, discriminated by the
  `category` enum (`'new' | 'maintenance' | 'lead'`). The Orders tab reads only the
  `new` and `maintenance` categories.

Reads are routed through `routeRead(...)` behind feature-flag group `PG_READ_COMPLEX_4`
(constants `LISA_ORDER_READ_GROUP` / `ORDER_READ_GROUP`). Parity of every read primitive
is proven by the parity tests listed above.

### Column-by-column source for the WeInfuse half

| UI column | Postgres column | Notes |
| --- | --- | --- |
| WI Order Series ID | `orders.order_series_id` | Falls back to the order id when null on older rows, so Series ID and Order ID can show the same value |
| WI Order ID | `orders.we_infuse_order_id` | The WeInfuse order id. Indexed but **not** unique — see below |
| Status | `orders.order_series_status`, falling back to `orders.status` | Shows the **series-level** status first, not the individual order status |
| Order Date | `orders.order_date` | `timestamptz`; rendered `MM/DD/YYYY` |
| Drug | `orders.first_listed_primary_med` | Plain text sourced from Looker — uppercased for display, **not** joined to `li_drugs` |
| Link | derived from `orders.map_to_lisa_orders` | Not a stored column — the link/unlink action |

### Column-by-column source for the LISA half

| UI column | Postgres column |
| --- | --- |
| Order ID | `lisa_orders.display_id` |
| Type | `lisa_orders.order_type` (new) or `lisa_orders.change_type` (maintenance) — both enums |
| Status | `lisa_orders.status` (text — app-managed value set that has grown over time) |
| Order Date | `lisa_orders.received` |
| Drug | `li_drugs.name` via `lisa_orders.li_drug_id`, falling back to `lisa_orders.drug_raw` |

## The relation

```
┌───────────────────────────────────┐            ┌────────────────────────────────────────────┐
│           lisa_orders             │            │                  orders                    │
│         (LISA orders)             │            │            (WeInfuse orders)               │
├───────────────────────────────────┤            ├────────────────────────────────────────────┤
│ id                bigserial  PK   │◄───────────┤ lisa_order_id        bigint    FK  NULL     │
│                                   │  REAL FK   │   REFERENCES lisa_orders(id)               │
│                                   │            │   ON DELETE SET NULL                       │
│                                   │            │   idx: orders_lisa_order_id_idx            │
│                                   │            │                                            │
│ mongo_id          text  NOT NULL  │◄╌╌╌╌╌╌╌╌╌╌╌┤ map_to_lisa_orders   text      NULL         │
│   uq: lisa_orders_mongo_id_idx    │  NO FK     │   idx: orders_map_to_lisa_orders_idx        │
│                                   │ (by value) │                                            │
│ category  enum new|maintenance|   │            │ order_series_id      integer                │
│                lead               │            │ we_infuse_order_id   integer  (not unique)  │
│ display_id, status, received, …   │            │ order_series_status, order_date, …          │
│                                   │            │                                            │
│ we_infuse_order_series_id integer ├╌╌╌╌╌╌╌╌╌╌╌►│ order_series_id                             │
│   NO FK — target is not unique    │  advisory  │                                             │
└───────────────────────────────────┘            └────────────────────────────────────────────┘

                 1  ───────────────────────────────<  N
        one LISA order  ←  many WeInfuse orders (a whole order series)
```

The link is stored **entirely on `orders`**. `lisa_orders` has no column pointing back at
`orders`; the relation is one-directional.

### Are the link columns foreign keys?

| Column | FK? | Detail |
| --- | --- | --- |
| `orders.lisa_order_id` | **Yes** | `bigint` → `REFERENCES lisa_orders(id) ON DELETE SET NULL`. Enforced. |
| `orders.map_to_lisa_orders` | **No** | Plain `text`, indexed but unconstrained. Joins to `lisa_orders.mongo_id` by value. |
| `lisa_orders.we_infuse_order_series_id` | **No** | Plain `integer`, indexed. Cannot be an FK — its target `orders.order_series_id` is deliberately non-unique. |

`map_to_lisa_orders` and `lisa_order_id` are an **expand/contract pair**, not duplicates:

- `map_to_lisa_orders` is currently the **source of truth for reads** — the read path
  matches on the text column so results stay parity-exact during the migration.
- `lisa_order_id` is **derived**: the text value is resolved against `lisa_orders.mongo_id`
  and the resulting `lisa_orders.id` is stored. It is re-resolved on every write that
  touches the text column, so the two cannot drift.

Per `packages/db/CLAUDE.md`, foreign keys always reference the parent's `bigserial id`,
and `mongo_id` is transitional and will be dropped when the migration completes. The
cut-over for this relation is therefore: move the reads from `map_to_lisa_orders` to
`lisa_order_id`, then drop the text column.

**Caveat to plan around:** `lisa_order_id` can be `NULL` while `map_to_lisa_orders` is
populated — that happens when the referenced LISA order has no row in `lisa_orders`. Those
rows read as linked today but would silently become unlinked if the reads switched to the
FK without a backfill. Worth an audit query before the cut-over.

### Cardinality, and why `we_infuse_order_id` is not unique

One LISA order to many WeInfuse orders. Two reasons for the N:

1. A WeInfuse order **series** is linked as a unit — every row sharing
   `orders.order_series_id` receives the same `map_to_lisa_orders` value.
2. The `orders` table carries **both a modern row and a legacy-ETL row for the same
   WeInfuse order** in thousands of cases. Row identity is therefore `orders.mongo_id`
   (unique index `orders_mongo_id_idx`), and `we_infuse_order_id` carries only a plain
   index.

`LisaOrderRow.weInfuseOrders` is an array in the API response, and the UI renders multiple
WeInfuse rows beside a single LISA row.

### `lisa_orders.we_infuse_order_series_id` is not the link

It looks like a reverse link but is not one. It is **intake-captured provenance**: the
order-creation routes (`/api/orders/new`, `/api/orders/maintenance`) accept
`weInfuseOrderSeriesId` in the request body and persist it on the new LISA order.

It serves two purposes:

1. **Seed for the real link.** `linkCreatedOrdersToWiSeries` reads it from freshly created
   LISA orders and triggers the series-wide write that populates
   `orders.map_to_lisa_orders` (and the derived `orders.lisa_order_id`). Fire-and-forget:
   failures are logged and never fail order creation. Base orders claim a series first; an
   SQ protocol partner claims its own series only when no base took it.
2. **Display value.** The order-details route merges it with the series ids reached
   through the real link:
   `weInfuseOrderSeriesIds = unique(seriesFromLinked ∪ seriesFromTracker)`.

**It is never kept in sync.** Linking or unlinking through the dialog writes only
`orders.map_to_lisa_orders` / `orders.lisa_order_id`; this column keeps whatever intake
recorded. After a manual re-link the two can disagree. For "which WeInfuse orders belong
to this LISA order", join `orders.lisa_order_id → lisa_orders.id` and ignore this column.

### Patient scoping

Both tables reach `patients` independently, through different keys — which is why the tab
needs two identifiers:

| Table | FK column | Column actually matched on at read time |
| --- | --- | --- |
| `lisa_orders` | `patient_id → patients.id` | `patient_raw` |
| `orders` | `patient_id → patients.id` | `we_infuse_patient_id` (numeric WeInfuse id) |

The reads match on the raw/source columns rather than the resolved FKs to stay
parity-exact during the migration. Dedicated indexes exist for exactly this
(`orders_we_infuse_patient_id_idx`, `lisa_orders_patient_raw_category_deleted_idx`) —
without them these reads sequential-scan once `PG_READ_COMPLEX_4` is on.

The link route enforces that the two identifiers agree — it rejects with 403 when the
patient's WeInfuse id does not match the order's.

## How WeInfuse orders get into Postgres — the `looker_orders` webhook

Rows in `orders` are not created by the application. They arrive from WeInfuse by way of a
Looker export, delivered to a webhook endpoint. The Orders tab only ever reads them and
flips their link columns.

### The chain

| Step | File |
| --- | --- |
| HTTP endpoint | `apps/web/pages/api/webhooks/looker_orders.ts` |
| Auth | `validateLookerWebhook` — `apps/web/services/webhooks/middleware/lookerAuth.ts` |
| Handler | `handleLookerOrders` — `apps/web/services/webhooks/handlers/looker_orders.ts` |
| Transform + link inference | `webhookUpsertOrders` — `apps/web/services/webhooks/handlers/util/webhookUpsertOrders.ts` |
| Persistence entry point | `bulkUpsertOrders` — `apps/web/services/mongodb/orders.ts` |
| Postgres write | `OrderRepository` — `packages/db/src/orders/repository.ts` |

`POST /api/webhooks/looker_orders`, body limit 10 MB. Authentication is a shared secret in
the `x-looker-webhook-token` header, compared against `LOOKER_WEBHOOK_TOKEN`. A generic
`pages/api/webhooks/[type].ts` dispatcher also exists, but this webhook has its own
dedicated endpoint file. Responses: 200 on success, 422 when the handler reports failure,
500 on an unhandled throw.

### Payload shape

Looker sends flat rows with dotted keys (`WebhookOrderEntry`) — a join of `orders` ×
`order_series` × `formulary_entries`:

```
orders.id                                orders.order_series_id
orders.patient_id                        orders.status
order_series.status                      orders.order_date
orders.expiry_date                       orders.indication_code
orders.first_listed_primary_med
formulary_entries.first_listed_ndc_code  formulary_entries.med_name_id
```

### Payload field → `orders` column

`transformToOrderSchema` performs the mapping.

| Looker key | `orders` column | Notes |
| --- | --- | --- |
| `orders.id` | `we_infuse_order_id` | |
| `orders.order_series_id` | `order_series_id` | |
| `orders.patient_id` | `we_infuse_patient_id` | `patient_id` FK is resolved from this at write time |
| `orders.status` | `status` | |
| `order_series.status` | `order_series_status` | |
| `orders.order_date` | `order_date` | |
| `orders.expiry_date` | `expiry_date` | Only mapped when the `ORDER_DETAILS` flag is on |
| `orders.indication_code` | `indication_code` | Only mapped when the `ORDER_DETAILS` flag is on |
| `orders.first_listed_primary_med` | `first_listed_primary_med` | Skipped when null, so a stored value is preserved |
| `formulary_entries.first_listed_ndc_code` | `first_listed_ndc_code` | Skipped when null |
| `formulary_entries.med_name_id` | `med_name_id` | Skipped when null |
| *(derived)* | `active` | `true` only when `orders.status` is `"Active"` (case-insensitive) |
| *(derived)* | `map_to_lisa_orders`, `lisa_order_id` | Auto-linking — see below |

### Auto-linking on ingest

This is the ingest-side counterpart to the Link dialog: the webhook can populate the link
columns without any user action. Two mechanisms, applied in order:

1. **Series inheritance.** If any existing order in a series is already linked to a LISA
   order, every incoming order in that same series inherits the link.
2. **Intake back-fill.** For series still unlinked after step 1, it looks for a LISA order
   carrying that series in **`lisa_orders.we_infuse_order_series_id`** and links to it.
   This is the column documented above as advisory provenance — consumed here from the
   opposite direction. Failures are non-fatal: the lookup error is logged and the upsert
   proceeds unlinked.

So `orders.map_to_lisa_orders` has three possible origins: this webhook's series
inheritance, this webhook's intake back-fill, and a manual link through the dialog.

### Delta mode, not snapshot

An empty payload is a **no-op**. This is deliberate and load-bearing: under the previous
snapshot behavior, absence from the payload implied inactive, and a single empty delivery
on 2026-07-01 mass-deactivated 6,484 orders. Since MLID-2748, `active` is derived per row
from `orders.status`, and absence never deactivates anything.

### Duplicate rows from the Looker join

The `formulary_entries` join fans out: one med matching more than one formulary entry
produces several rows with the same `orders.id`. `dedupeOrdersById` keeps the first
occurrence and logs the duplicate ids, because a single Postgres upsert statement cannot
touch the same row twice. Which row wins depends on Looker's row order, so the formulary
fields kept for a duplicated order may differ between deliveries — acceptable, since the
source data is ambiguous by definition and every delivery re-upserts.

### What reaches Postgres

The upsert must **merge over the existing row, not replace it**. A Looker delivery is a
partial payload: `transformToOrderSchema` omits fields that arrive null, and omits
`expiry_date` / `indication_code` entirely while `ORDER_DETAILS` is off. A plain
overwrite would null those columns on every delivery. The persistence layer therefore
reads the current rows first and merges the payload over them before writing, and writes
in chunks rather than one statement.

**Replicating this webhook: this is the behavior to preserve.** Either merge before
writing, or use `ON CONFLICT DO UPDATE` with a `SET` list restricted to the columns the
payload actually carries. Never `SET` every column from a partial payload.

The repository resolves both foreign keys from the raw source values before writing:

- `we_infuse_patient_id` → `patients.id` → stored in `patient_id`
- `map_to_lisa_orders` → `lisa_orders.mongo_id` → `lisa_orders.id` → stored in
  `lisa_order_id`

Unresolvable references are stored as `NULL` in the FK column while the raw/source column
keeps its value, so resolution stays re-runnable.

### Error handling

A persistence failure is re-thrown rather than swallowed, so the webhook answers non-2xx.
The reasoning is recorded in the code: a swallowed failure reads as a healthy sync while
nothing is actually being written. Error `cause` chains are collected into a separate log
property because App Insights truncates `errorMessage` / `errorStack` at 8192 characters,
which would otherwise hide the root cause of long Drizzle "Failed query" messages.

## The Link dialog — end-to-end flow

### Short answer: what a link click affects

**One table is written: `orders`. Three columns.**

| Table | Written? | Involvement |
| --- | --- | --- |
| `orders` | **Yes** | `map_to_lisa_orders`, `lisa_order_id`, `updated_at` |
| `lisa_orders` | No — **read only** | Read to resolve `mongo_id` → `id` for the FK, and to populate the picker list |
| `patients` | No — **read only** | Read to verify the order belongs to the patient |

Sequence:

1. The user picks a LISA order in the dialog. The client sends
   `PUT /api/patient/{patientId}/orders/{wiOrderId}/link` with the chosen LISA order's
   identifier.
2. The route runs its guards (session, role, feature flag, id validation, and a check that
   the patient's WeInfuse id matches the order's).
3. It reads the clicked WeInfuse row to get its **`order_series_id`**, then discards the
   row. The write is series-scoped.
4. It resolves the selected LISA order against `lisa_orders` to get its `lisa_orders.id`.
5. It updates **every row in `orders` sharing that `order_series_id`**:

| Column | New value |
| --- | --- |
| `map_to_lisa_orders` | the selected LISA order's `mongo_id` value |
| `lisa_order_id` | the resolved `lisa_orders.id` (FK) |
| `updated_at` | `now()` |

On unlink, the same rows get `map_to_lisa_orders = NULL` and `lisa_order_id = NULL`.

The two things that are easy to get wrong:

- **It is not a single-row update.** One click updates the whole order series — every
  `orders` row with the same `order_series_id`.
- **The LISA order is never modified.** `lisa_orders` is only read. Nothing on the LISA
  side records that a link happened, and no audit row is written anywhere.

### Full flow

The link icon appears only on **unlinked WeInfuse rows** (rows rendered from
`unlinkedWeInfuseOrders`, showing "No linked LISA source" on the left). Linked rows get an
unlink icon instead, opening `UnlinkOrderModal`.

1. **Open.** The button stores `{ wiOrderId, orderSeriesId }`. `wiOrderId` is the WeInfuse
   row's `orders.mongo_id` value — not `we_infuse_order_id` and not the series id. The
   header renders the *series* id, because the write is series-wide.

2. **Populate the picker.** The modal calls the same `usePatientOrders` hook as the page,
   issuing `GET /api/patient/[patientId]/orders?page=1&pageSize=25&search=…`, and renders
   `data.lisaOrders` as `display_id · drug · type · status`. Search is debounced 300 ms.

3. **Select.** Clicking an item sets local state only; no request is issued. "Save Link"
   stays disabled until something is selected.

4. **Save.**

   ```
   PUT /api/patient/{patientId}/orders/{wiOrderId}/link
   { "lisaOrderId": "<lisa_orders.mongo_id value>" }
   ```

   On a non-2xx response the modal stays open showing "Failed to link order. Please try
   again." — it does not surface the server's error message.

5. **Server guards**, in order: session present (401) → role in
   `['admin','auditor','user']` (403) → `PATIENT_ORDERS_TAB` enabled (404) → body parses
   (400) → all three ids are well-formed (400) → order exists (404) → patient exists (404)
   → **patient's WeInfuse id matches the order's** (403) → the order has a numeric
   `order_series_id` (404).

6. **Write.** Series-scoped, not row-scoped: the clicked row is used only to look up its
   `order_series_id`, which is then discarded.

7. **Response and refresh.** `{ success: true }`. When linking and nothing matched, the
   route returns 404 instead; unlink is idempotent and accepts a zero match count. The
   modal then triggers a single refetch and closes. The row moves from the unlinked block
   into the linked LISA row's `weInfuseOrders` array on the next render.

### How the write is targeted

The update targets an **exact set of `orders.mongo_id` values** — the rows matched by the
series lookup — never a translated filter. See "Short answer" above for the columns
written. Two details not repeated there: `lisa_order_id` is also `NULL` when the selected
LISA order has no row in `lisa_orders` yet, and the FK is re-resolved on every write so it
cannot drift from the text column.

### Behavioral consequences worth knowing

- **Linking is a series-level operation.** Selecting one WeInfuse row links every row
  sharing its `order_series_id` (MLID-2432). The UI already collapses a series to one
  representative row, so the clicked row stands for the whole series.

- **Legacy-ETL rows are excluded from the update.** The series lookup matches the modern
  row shape only, so a legacy twin keeps its previous `map_to_lisa_orders` value. This is
  deliberate and documented as an "identity-exact mirror" decision.

- **"Unlinked" is not simply `map_to_lisa_orders IS NULL`.** The service re-checks that
  the referenced LISA order still exists, is not soft-deleted (`lisa_orders.deleted =
  false`), and belongs to this patient. A WeInfuse row is unlinked when the column is null
  **or** points at a deleted / other-patient LISA order. In the default paginated view, a
  WeInfuse row linked to a LISA order on a *different page* counts as linked, not
  unlinked.

- **Series deduplication.** Each `order_series_id` group is collapsed to one
  representative row before rendering. If exactly one row in the group is not `Revised`,
  the `Revised` siblings are dropped; with zero or two-plus non-`Revised` rows the whole
  group is kept. The suppression is deliberately **global**, not per page, so a `Revised`
  row stays hidden even when its sibling lands on another page.

- **Legacy-ETL rows can render a blank WI Order ID.** Those rows store the WeInfuse order
  id under a legacy field name that the row formatter does not read, so the WI Order ID
  cell is empty while WI Order Series ID still resolves. Possible small bug — not yet
  investigated.

- **The picker searches `display_id` only.** The placeholder reads "Search by order ID or
  drug…", but the filter is `ILIKE` on `lisa_orders.display_id`. Typing a drug name
  returns nothing.

- **The picker shows only the first 25 LISA orders.** `usePatientOrders` defaults to
  `page=1, pageSize=25` and the modal renders no pagination, so anything past the first 25
  is reachable only by searching an order id.

- **No audit trail.** The action writes only the link columns — nothing lands in
  `lisa_orders.audit` or `audit_logs`, so who linked what and when is recoverable only
  from the application log.

- **The selected LISA order is not validated.** The route verifies the WeInfuse order
  belongs to the patient but does not check the same for the chosen LISA order, nor that
  the drugs match. A cross-patient link would simply render as unlinked afterwards,
  because the read path scopes by patient.
