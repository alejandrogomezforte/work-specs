# [MLID-2417] feat(notifications): document notifications for maintenance orders (epic D1–D5)

## Summary

Ports the document-notification + SignalR machinery built in MLID-2011 from new orders to **maintenance orders** (`maintenanceorders`), and threads order category through the document-link path. This is the **second consumer** of the MLID-2011 foundation — no net-new notification infrastructure is introduced; the work wires the existing badge/chip UI, propagation orchestrator, and API routes into the maintenance surfaces and widens a few shared helpers to span both collections.

Most sub-tasks turned out to be **verification + test coverage only**, because the order-detail surfaces already live under the shared `[category]` route and the propagation helpers are collection-agnostic. The real production changes are concentrated in three places (D1 table badge, D4 upload fan-out + reassignment cascade, D5 link-category pass-through).

> **Scope note:** D0 (the new-orders retrofit hotfixes — MLID-2391, MLID-2413, MLID-2439) is **not** part of this PR. D0 shipped independently off `develop` earlier in the sprint. This PR covers **D1–D5 only**.

## Deliverables

| D | Ticket | Area | Production change? | Key change |
|---|--------|------|:--:|------------|
| **D1** | MLID-2447 | UI | Yes | Unread-count badge per row on the `/orders/maintenance` table |
| **D2** | MLID-2448 | UI | No (verify) | Documents-tab badge, "New" chip, mark-as-read, mark-all-as-read on maintenance order detail |
| **D3** | MLID-2449 | UI | No (verify) | Document log entries on the maintenance order History tab |
| **D4** | MLID-2450…2454 | BE | Partial | Maintenance fan-out on upload / e-order-fax / link / unlink / reassignment |
| **D5** | MLID-2628 | FS | Yes | Thread order category through document linking; remove cross-collection guess |

## Key Changes

**D1 — Maintenance table badge (MLID-2447).** Extracted the badge cell into a shared `OrderNotificationBadge` component used by both the new- and maintenance-orders column factories. `page.tsx` now includes maintenance order ids in `visibleOrderIds` (was new-only), so the existing `useOrderNotificationCounts` / `POST /api/notifications/counts` path serves maintenance rows. Badge links to `/orders-tracker/maintenance/${id}/documents`.

**D2 + D3 — Documents & History tabs (MLID-2448, MLID-2449).** No production code change: the order-detail layout, documents page, and history page are all under the shared `[category]` route with no `category === 'new'` gate, and the audit-log "Change" formatter is category-agnostic. Delivered as maintenance-specific test coverage plus manual verification once D4 produces maintenance notification data.

**D4 — Maintenance triggers (MLID-2450…2454).** The one upload-path change: `getOrdersByPatient`'s lean (`expandRefs: false`) branch gained an optional `collections` option (default `['new']` — zero change for existing callers) and now fans out across the requested collections; `notifyDocumentCreated` passes `['new', 'maintenance']`. This single change covers intakes upload **and** e-order/fax (same chain). Link (T3) and unlink (T4) needed no change — `handleOrderLink`/`handleOrderUnlink` already key on the `(order, document)` tuple regardless of collection. The reassignment cascade (T5) was added to `PUT /api/orders/maintenance`, mirroring the D0-T1 wiring on `/api/orders/new` and reusing the unmodified `handleOrderReassignment` helper.

**D5 — Link-orders category pass-through (MLID-2628).** `handleOrderLink` previously resolved each order with a two-query cross-collection guess (`getOrderById(id, 'new') ?? getOrderById(id, 'maintenance')`) — two full aggregations for a maintenance order. The category is known at every UI sender and was being dropped at the request boundary. Now the request body is `{ linkedOrders: [{ id, category }] }`, both senders (`LinkOrdersModal`, intake submission) supply the category, and each order resolves in a single `getOrderById(id, category)` call. Document `linkedOrderIds` storage stays a bare `string[]`.

## Feature flag

All surfaces stay gated by the existing `FeatureFlag.ORDER_DOCUMENT_NOTIFICATIONS` (reused — no parallel flag), default `false`.

## Changes Overview

- **31 files changed**, +2292 / −243.
- Production source touched: `OrderNotificationBadge.tsx` (new), `MaintenanceOrders.tsx`, `NewOrders.tsx`, `[category]/page.tsx`, `orders/maintenance/route.ts`, `ordersTracker.ts`, `documentNotification.ts`, `link-orders/route.ts`, `lisaOrders.ts`, and the D5 document-intake sender chain (`useSubmissionProcessor.ts`, `submission-api.ts`, `UpdateOrderForm.tsx`, `LinkOrdersModal.tsx`, `types.ts`). The remainder is test coverage.

## Test Plan

- [x] Automated: new/updated suites pass for every touched surface (maintenance table column, documents/layout/history pages, maintenance reassignment route, `getOrdersByPatient` multi-collection lookup, `documentNotification` link contract, weinfuse uploader fan-out, link-orders route, lisaOrders). Type-check + ESLint clean on changed files.
- [x] Existing new-orders behavior verified unchanged at every shared surface (badge, tabs, propagation, link/unlink, reassignment).
- [x] Manual end-to-end on maintenance orders `0yn` and `010w` across two accounts: table badge, Documents-tab badge + "New" chip + mark-as-read/mark-all (live via SignalR), History log entries, upload/link/unlink fan-out, reassignment cascade, and feature-flag-OFF hiding everything. Tester ≠ assignee where the MLID-2434 self-upload suppression applies. D5 link payload confirmed carrying `category: "maintenance"` from both senders.

## Jira

- Epic: [MLID-2417](https://localinfusion.atlassian.net/browse/MLID-2417)
- Sub-tasks: D1 [MLID-2447] · D2 [MLID-2448] · D3 [MLID-2449] · D4 [MLID-2450] [MLID-2451] [MLID-2452] [MLID-2453] [MLID-2454] · D5 [MLID-2628]
- Not in this PR (shipped separately off `develop`): D0 [MLID-2391] [MLID-2413] [MLID-2439]
