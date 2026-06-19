# MLID-2628 — D5-T1: Link-Orders Category Pass-Through (remove collection guessing)

## Task Reference

- **Jira**: [MLID-2628](https://localinfusion.atlassian.net/browse/MLID-2628) — standalone deliverable D5-T1 under epic MLID-2417.
- **History**: Originally scoped as D1-T2 / "Stage 2" of MLID-2447 (the maintenance table badge). Split out 2026-06-18 into its own ticket and its own feature branch — it is **not** attached to MLID-2447 anymore and does **not** ship in the D1-T1 PR.
- **Why it survives the "merge collections" direction**: even if `neworders` and `maintenanceorders` are merged into one collection later, `handleOrderLink` still benefits from receiving the category to resolve the order in a single `getOrderById` call instead of guessing — this removes a redundant aggregation today and is forward-compatible.
- **Branch**: `feature/MLID-2628-link-orders-category-passthrough` (new branch, off the epic).
- **Base Branch**: `epic/MLID-2417-document-notifications-maintenance`.
- **Epic Plan**: `docs/agomez/plans/MLID-2417-plan-progress.md`.
- **Status**: To Do.

---

## Problem

When a document is linked to one or more orders, `handleOrderLink` (in `apps/web/services/notifications/documentNotification.ts`) resolves each order with a sequential cross-collection guess:

```ts
const order =
  (await getOrderById(orderId, 'new')) ??
  (await getOrderById(orderId, 'maintenance'));
```

For a **maintenance** order this runs **two** full aggregation queries (`getOrderById` runs the enriched `getOrdersAggregate` pipeline, not a cheap `findOne`): the first `neworders` lookup returns `null`, then the `maintenanceorders` lookup runs. For a **new** order it is one query (the right side short-circuits on `??`).

This guess exists only because the order **category is discarded before it reaches the server**. The information is known at every UI source and is thrown away at the request boundary. This task threads the category end-to-end so `handleOrderLink` calls `getOrderById(id, category)` exactly once, and the guess is deleted.

This is a write-path correctness/cleanliness improvement, not a hot path — but the data is already available, so guessing is unjustified.

---

## Why the category is recoverable (it is already known and dropped)

**Sender 1 — patient Documents tab** (`apps/web/components/Modules/patient/components/patientDocumentRecords/LinkOrdersModal.tsx`):
- Order list comes from `usePatientOrders(patientId)` → `lisaOrders` (`LisaOrderRow`), each carrying `category`. The modal already renders it per row (`order.category === 'new' ? 'New Order' : 'Maintenance'`).
- On save it sends `linkedOrderIds: Array.from(selectedOrderIds)` — a bare id array. The category is dropped here.

**Sender 2 — document-intake submission** (`apps/web/components/Modules/document-intake/components/useSubmissionProcessor.ts`, the `LINK_DOCUMENT_TO_LISA_ORDER` callback):
- Calls `DocumentOrderLinkApiService.linkDocumentToOrders(documentId, lisaOrderIds)` in `apps/web/components/Modules/document-intake/components/submission-api.ts`, which sends `linkedOrderIds: orderIds`.
- `lisaOrderIds` is either:
  - `createdLisaOrderIds` — orders just created during this intake; their category equals the intake's `selectedType` (`'new' | 'maintenance'`, from `order.orderTypeReview.selectedType`), or
  - `[selectedExistingOrderId]` — an existing order picked in the "update" flow from `existingOrders` (`ExistingOrderOption`). That option list is built by `useLisaOrderOptions` → `getLisaOrderOptions` in `apps/web/services/mongodb/lisaOrders.ts`, which **knows** the category (it builds the `"(${category})"` label) but returns only `{ id, label }`.

**Server source already has it too**: `getLisaOrderOptions(patientId, category)` receives `category` as a parameter and uses it to pick the collection and build the label, then omits it from the returned `LisaOrderOption`.

So no new database reads are needed anywhere to obtain the category — only stop discarding it.

---

## Target contract

`PUT /api/documents/[documentId]/link-orders` request body changes from:

```ts
{ linkedOrderIds: string[] }
```

to:

```ts
{ linkedOrders: Array<{ id: string; category: 'new' | 'maintenance' }> }
```

- The route still persists bare ids via `linkDocumentToOrders(documentId, ids)` — `document.linkedOrderIds` storage stays a `string[]`, unchanged.
- The route passes the `{ id, category }` pairs (only for the newly-added delta) to `handleOrderLink`.
- `handleOrderLink` calls `getOrderById(id, category)` once. The `'new' ?? 'maintenance'` fallback is deleted. If the order is not found for the supplied category, it logs the existing "order not found" warning and skips (no cross-collection retry).

**Out of scope (confirmed):** the unlink path (`handleOrderUnlink`) needs no category — it deletes `notifications`/`notificationreads` by the `(orderId, documentId)` tuple and never resolves the order. The `/api/intakes/[id]/link-orders` route does not call `handleOrderLink`, so it is untouched.

---

## Stage map

All stages land on the same branch / same PR (`feature/MLID-2628-link-orders-category-passthrough`). Each stage is independently testable.

| Stage | Title | Depends on | End-of-stage testable outcome |
|------|-------|-----------|-------------------------------|
| 1 | Server accepts category + helper uses it (backward compatible) | — | Route accepts the new `linkedOrders` shape **and** the legacy `linkedOrderIds` shape; `handleOrderLink` uses a supplied category and only falls back to the guess when category is absent. All existing tests pass. |
| 2 | Senders + UI types send category | Stage 1 | Both senders (patient modal, intake submission) send `linkedOrders: {id, category}[]`. End-to-end linking works with category supplied; verify in browser that linking a maintenance order produces a single `getOrderById` call. |
| 3 | Remove legacy shape + the guess | Stage 2 | Route accepts only `linkedOrders`; `handleOrderLink` requires category; the `?? 'maintenance'` fallback and any legacy `linkedOrderIds` handling are deleted. |

Rationale for staging: Stage 1 is additive so nothing breaks while senders still send the old shape; Stage 2 migrates the senders; Stage 3 tightens the contract and removes the guess only after every caller is migrated.

---

## Stage 1 — Server accepts category; helper uses it (backward compatible)

### Step 1.1 (RED) — `handleOrderLink` uses a supplied category

**Files:** `apps/web/services/notifications/documentNotification.test.ts`

Add tests for `handleOrderLink` (and adjust `HandleOrderLinkParams`):
- When called with an order whose category is `'maintenance'`, `getOrderById` is called once with `'maintenance'` and **not** with `'new'`.
- When called with category `'new'`, `getOrderById` is called once with `'new'`.
- When a category is supplied but the order is not found for that category, it logs the "order not found" warning and does **not** retry the other collection.
- (Back-compat) when an entry has no category, it still resolves via the legacy `'new' ?? 'maintenance'` path (kept until Stage 3).

These fail because the helper currently ignores category and always probes.

### Step 1.2 (GREEN) — Implement category-aware resolution in `handleOrderLink`

**Files:** `apps/web/services/notifications/documentNotification.ts`

- Change `HandleOrderLinkParams.newOrderIds: string[]` to a new field carrying pairs, e.g. `newOrders: Array<{ id: string; category?: 'new' | 'maintenance' }>` (category optional during Stage 1 for back-compat).
- For each entry: if `category` is present, `const order = await getOrderById(id, category)`; otherwise fall back to the existing `(await getOrderById(id,'new')) ?? (await getOrderById(id,'maintenance'))`.
- Keep the not-found warning and the `propagateDocumentNotificationToOrder` call unchanged.

### Step 1.3 (RED) — Route accepts both shapes

**Files:** `apps/web/app/api/documents/[documentId]/link-orders/__tests__/route.test.ts`

- New: request with `linkedOrders: [{ id, category }]` links the ids (persists via `linkDocumentToOrders`) and calls `handleOrderLink` with the category-bearing delta.
- Existing: request with legacy `linkedOrderIds: string[]` still works and calls `handleOrderLink` (without category → helper falls back). Keep these passing.
- Validation: each `linkedOrders[i].id` is a 24-char hex; each `category` is `'new'` or `'maintenance'`; malformed entries return 400.

### Step 1.4 (GREEN) — Implement dual-shape parsing in the route

**Files:** `apps/web/app/api/documents/[documentId]/link-orders/route.ts`

- Parse `linkedOrders` when present; otherwise read legacy `linkedOrderIds` and treat each as `{ id, category: undefined }`.
- Derive `ids = entries.map(e => e.id)` for storage (`linkDocumentToOrders(documentId, ids)`) and for the previous/added/removed delta computation (unchanged logic, but over `ids`).
- Build the added-delta as `{ id, category }[]` and pass it to `handleOrderLink({ ..., newOrders })`.
- `handleOrderUnlink` call unchanged (still id-only).

**Checkpoint:** full existing suite green; new shape accepted alongside old.

---

## Stage 2 — Senders + UI types send category

### Step 2.1 (GREEN+RED) — Carry category on the option/row types

**Files:**
- `apps/web/services/mongodb/lisaOrders.ts` — add `category: 'new' | 'maintenance'` to `LisaOrderOption` and include it in the object returned by `getLisaOrderOptions` (the value is already in scope).
- `apps/web/components/Modules/document-intake/forms/OrderTypeReview/types.ts` — add `category: 'new' | 'maintenance'` to `ExistingOrderOption`.
- Update `getLisaOrderOptions` unit tests and `useLisaOrderOptions` tests to assert `category` is present on each option.

### Step 2.2 (RED) — `DocumentOrderLinkApiService.linkDocumentToOrders` sends category

**Files:** `apps/web/components/Modules/document-intake/components/submission-api.test.ts`

- Assert the request body is `{ linkedOrders: [{ id, category }] }` (not `{ linkedOrderIds }`).

### Step 2.3 (GREEN) — Update the API wrapper + intake submission sender

**Files:**
- `apps/web/components/Modules/document-intake/components/submission-api.ts` — change `linkDocumentToOrders(documentId, orders: Array<{ id; category }>, options?)`; body becomes `{ linkedOrders: orders }`.
- `apps/web/components/Modules/document-intake/components/useSubmissionProcessor.ts` — build the `{ id, category }[]`:
  - For `createdLisaOrderIds`: category = the intake `selectedType` (`order.orderTypeReview.selectedType`), restricted to `'new' | 'maintenance'`.
  - For `[selectedExistingOrderId]`: look up the matching `existingOrders` entry (now category-bearing) and use its `category`.
- Update `useSubmissionProcessor` tests accordingly.

> **Open item to confirm during implementation:** verify that every id in `createdLisaOrderIds` corresponds to the intake's `selectedType` (i.e., an intake never creates a mix of new + maintenance in one submission). If it can, source each created id's category from the creation result rather than from `selectedType`. This is the only place where the category mapping is not already one-to-one obvious.

### Step 2.4 (RED) — Patient modal sends category

**Files:** `apps/web/components/Modules/patient/components/patientDocumentRecords/__tests__/patientDocumentRecords.test.tsx` (or a focused `LinkOrdersModal` test)

- Assert that saving sends `{ linkedOrders: [{ id, category }] }`, with category taken from the selected order's `lisaOrders` row.

### Step 2.5 (GREEN) — Update `LinkOrdersModal`

**Files:** `apps/web/components/Modules/patient/components/patientDocumentRecords/LinkOrdersModal.tsx`

- At save time, map each selected id to `{ id, category }` using the `lisaOrders` list already in scope (build an `id → category` map). Send `{ linkedOrders }`.
- `selectedOrderIds` can remain a `Set<string>` internally; resolve category at save time from `lisaOrders`.

**Checkpoint (browser):** Link a maintenance order to a document from the patient Documents tab and from intake submission. Confirm the link succeeds and the maintenance-order resolution issues a single `getOrderById('…','maintenance')` call (verify via logs / a temporary count).

---

## Stage 3 — Remove legacy shape and the guess

### Step 3.1 (RED) — Tighten contract + helper

**Files:** route test + `documentNotification.test.ts`
- Route: a request with legacy `linkedOrderIds` (no `linkedOrders`) now returns 400.
- `handleOrderLink`: category is required on every entry; an entry without a valid category is skipped with a logged warning (no cross-collection retry); update `HandleOrderLinkParams` so `category` is non-optional.

### Step 3.2 (GREEN) — Delete the fallback

**Files:**
- `apps/web/app/api/documents/[documentId]/link-orders/route.ts` — remove legacy `linkedOrderIds` parsing.
- `apps/web/services/notifications/documentNotification.ts` — make `category` required in `HandleOrderLinkParams`; delete the `(await getOrderById(id,'new')) ?? (await getOrderById(id,'maintenance'))` fallback; update the docstring that currently explains the guess.

**Checkpoint:** full suite green; grep confirms no remaining `?? (await getOrderById(` guess and no remaining `linkedOrderIds` request field on the documents link endpoint.

---

## Files Affected (summary)

| File | Action | Description |
|------|--------|-------------|
| `apps/web/services/notifications/documentNotification.ts` | Modify | `handleOrderLink` takes `{ id, category }[]`; uses `getOrderById(id, category)`; remove the guess (Stage 3) |
| `apps/web/app/api/documents/[documentId]/link-orders/route.ts` | Modify | Accept `linkedOrders: {id,category}[]`; pass category-bearing delta to `handleOrderLink`; persist bare ids unchanged |
| `apps/web/services/mongodb/lisaOrders.ts` | Modify | Add `category` to `LisaOrderOption` + return it from `getLisaOrderOptions` |
| `apps/web/components/Modules/document-intake/forms/OrderTypeReview/types.ts` | Modify | Add `category` to `ExistingOrderOption` |
| `apps/web/components/Modules/document-intake/components/submission-api.ts` | Modify | `linkDocumentToOrders` sends `{ linkedOrders }` |
| `apps/web/components/Modules/document-intake/components/useSubmissionProcessor.ts` | Modify | Build `{ id, category }[]` (created → `selectedType`; existing → option category) |
| `apps/web/components/Modules/patient/components/patientDocumentRecords/LinkOrdersModal.tsx` | Modify | Send `{ linkedOrders }`, category from `lisaOrders` |
| Tests for each of the above | Modify/Create | TDD per step |

`document.ts#linkDocumentToOrders` and `handleOrderUnlink` are **unchanged** (storage stays bare ids; unlink is id-only).

---

## Testing Strategy

- **Unit — `handleOrderLink`**: one `getOrderById` call with the correct category; no second-collection call; not-found-for-category path; (Stage 1 only) back-compat fallback.
- **Integration — link-orders route**: new shape parsed/validated; correct delta + category passed to `handleOrderLink`; bare ids persisted; legacy shape accepted in Stage 1 then rejected in Stage 3.
- **Unit — `getLisaOrderOptions` / `useLisaOrderOptions`**: `category` present on options.
- **Component — `LinkOrdersModal`, intake submission**: request body carries `{ id, category }`.
- **Manual (browser)**: link new and maintenance orders from both entry points; confirm single maintenance lookup.
- **Regression gate**: existing new-orders link/propagation behavior unchanged.

---

## Security Considerations

- `category` is a constrained enum (`'new' | 'maintenance'`) validated server-side; ids keep the existing 24-char hex validation. No new PHI flows through the link payload.
- Authorization on the route (`hasAllowedRole`) and the `ORDER_DOCUMENT_NOTIFICATIONS` gate inside `handleOrderLink` are unchanged.

---

## Open Questions

1. Intake `createdLisaOrderIds` → category mapping: confirm a single intake submission cannot create a mix of new and maintenance orders (see the open item in Step 2.3). If it can, source category from the creation result per id instead of from `selectedType`.
