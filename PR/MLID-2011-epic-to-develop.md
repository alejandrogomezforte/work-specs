# [MLID-2011] feat(notifications): real-time order document notifications (epic → develop)

## Summary

This PR ships **the full MLID-2011 epic — Orders: Document Notifications — from the epic branch to `develop`**. It delivers the end-to-end feature: when a new document lands on an active order (e-order, fax, or manual upload), the user assigned to that order sees a real-time notification badge, can acknowledge it inline ("Mark as Read") or in bulk ("Mark all as Read"), and the change is recorded permanently in Order History.

The whole feature is gated behind the **`ORDER_DOCUMENT_NOTIFICATIONS`** feature flag, defaulted to `false`. Nothing changes for end users until the flag is toggled in the admin panel.

The epic also closes a long-standing gap in the Orders product: it lays down a **reusable Azure SignalR broadcast layer** that any future feature can use to push real-time events to connected clients (no Socket.IO migration — the two coexist).

Finally, on top of the notification work, this PR includes two recent UX/architecture refactors landed in the closing days of the epic:

- **Shared "Order Details Summary" header** rendered on all three order-detail tabs (Dashboard, Order History, Documents) instead of just the dashboard. Patient/Provider & Insurance/Fulfillment columns now live in the layout's `Card`, so all three routes get the same header strip — no more duplicated 3-column block on Order History.
- **Documents tab migrated from a hand-rolled `<table>` to the `@repo/ui` DataGrid component**, matching the design at `li-ui-playground` and the pattern used on the orders tracker list page. Sorting, pagination, and empty state are now handled by DataGrid out of the box.

## Why

Today, Infusion Guides find out about new order documents through Spruce — the patient-messaging app — which means document arrival messages compete for attention with patient messages that have a ~2-hour SLA. That noise hurts the team's ability to push orders to the ~3-day "Ready to Schedule" target.

This epic moves document arrival into an in-app real-time channel and a permanent audit trail, so:

- Documents are visible **on the order itself**, not in a chat app
- The assigned guide sees the badge appear live, not on next refresh
- Anyone on the team can see (but not clear) the backlog of unread documents per order — useful as a dashboard signal
- Document additions show up in Order History with source attribution (e-order / upload), giving a permanent trail without overloading any other field

## Architecture (one-paragraph version)

**SignalR is a 3rd-party service the webapp integrates with — like Spruce or WeInfuse.** The server (Next.js API routes) holds a connection-string-derived JWT and `POST`s to the Azure SignalR REST API to broadcast events. Each browser separately calls `POST /api/signalr/negotiate` to receive a per-user token and connects via `@microsoft/signalr` with `withAutomaticReconnect()`. Broadcasts are **fire-and-forget** — they tell clients "something changed, re-fetch from REST". Notification state lives in two MongoDB collections (`notifications` + `notificationreads`); SignalR carries no persistent state. Document targeting uses `targetUserId` (the order's assigned user) for ownership, but **entity-scoped badge counts are globally visible** so every team member sees which orders have outstanding work.

Full architecture reference: `docs/signalr/signalr-integration.md` and `docs/signalr/order-documents-signalr-notifications.md` (both added in this PR).

## What's in this PR (deliverable map)

The work is split into five deliverables (D0–D4). All except D4 (post-merge verification) are merged to the epic and shipping in this PR.

### D0 — Azure SignalR Infrastructure (Anatoliy / PE)

Provisioned in dev, staging, prod. Connection string in Key Vault as `signalr-primary-connection-string`. CORS configured per environment. Hub: `lihub`. Mode: Serverless. **Done before this PR; no app code changes here.**

### D1 — SignalR Integration (foundation)

| Sub-task | Jira | What it adds |
|---|---|---|
| D1-T1 | [MLID-2080](https://localinfusion.atlassian.net/browse/MLID-2080) | `ORDER_DOCUMENT_NOTIFICATIONS` feature flag + `broadcastToAll(eventName, payload)` server-side function calling `POST {endpoint}/api/hubs/{hub}/:send?api-version=2022-06-01`. SignalR constants (`HUB_NAME`, `TOKEN_EXPIRY`, `REST_API_VERSION`) initially externalized to env vars then later inlined as plain constants (`368ffe87`) since runtime override turned out to be unnecessary. Default hub name corrected to `lihub` (valid C# identifier — Azure SignalR rejects names with hyphens). Server-token audience corrected to the broadcast path on the REST API (`b4c38586`). Every successful broadcast logs at `info` level (`386e137b`) for observability. |
| D1-T2 | [MLID-2081](https://localinfusion.atlassian.net/browse/MLID-2081) | `Notification` + `NotificationRead` Mongoose models. Service layer with full CRUD (`createNotification`, `getNotificationsForUser`, `getUnreadCountForUser`, `getCountForEntity`, `markTaskAsRead` / `markUserNotificationAsRead` / `bulkMarkUserNotificationsAsRead`). Two notification types: `task` (any team member can ack — inline `isRead`/`readBy`/`readAt`) and `user` (per-user read receipts via `notificationreads` collection). Targeting via `targetUserId` / `targetLocationId` / `targetRole` with OR-logic match. Polymorphic entity linking via `entities[]: { entityType, entityId }`. Expiry-aware queries (sparse index, not TTL). Delivered by Anatoliy under PR #1049. |

After D1, the broadcast primitive exists and the data model is ready, but there are no callers yet.

### D2 — Document Notification Wiring

| Sub-task | Jira | What it adds |
|---|---|---|
| D2-T1 | [MLID-2082](https://localinfusion.atlassian.net/browse/MLID-2082) | Five new authenticated REST API routes: `GET /api/notifications` (user's inbox), `GET /api/notifications/counts` (header bell), `GET /api/notifications/by-order/[orderId]` (per-order detail), `PATCH /api/notifications/[id]/read` (single-mark via `markUserNotificationAsRead`), `PATCH /api/notifications/bulk-read` (bulk-mark via `bulkMarkUserNotificationsAsRead`). All gated by `ORDER_DOCUMENT_NOTIFICATIONS`. Mark-as-Read routes additionally enforce that the caller is the notification's `targetUserId` — non-targets get a 403 even though they can see the badge count. |
| D2-T2 | [MLID-2083](https://localinfusion.atlassian.net/browse/MLID-2083) | Two document-creation entry points wired to the notification system: `POST /api/documents/split-pdf` (manual uploads / split-pdf flow) and the JotForm submission handler in `services/jobs/definitions/jotFormSubmission/weinfuseUploader.ts` (e-order ingestion, including Spruce-source detection via `intake.source === 'spruce'`). Both call `notifyDocumentCreated()` which: (1) looks up the patient by `patientId`, (2) finds all active `neworders` for that patient (`patient.we_infuse_id_IV` match, `deleted !== true`), (3) creates a `user`-type notification per matched order with `targetUserId = order.assignedTo`, `entities = [{order}, {document}]`, (4) broadcasts a `documentAdded` SignalR event. Fire-and-forget — broadcast failures are logged but do not block the document write. |

### D3 — Notification UI

| Sub-task | Jira | What it adds |
|---|---|---|
| D3-T1 | [MLID-2084](https://localinfusion.atlassian.net/browse/MLID-2084) | **Order Tracker (list view):** unread-count badge next to the Order ID column for every order with outstanding documents. Powered by `useOrderNotificationCounts(orderIds[])` — single batched fetch against `/api/notifications/counts?orderIds=...`. SignalR `documentAdded` and `documentRead` events trigger a re-fetch so badges move live. Per the **2026-04-14 decision**, badges are **globally visible** to every authenticated viewer (not gated on assignment) so the tracker stays useful as a team dashboard. The "Action Required" quick filter introduced earlier in D3-T1 was removed (`5e9ea7d6`) after stakeholders decided the simpler badge-only view was the better signal. |
| D3-T2 | [MLID-2085](https://localinfusion.atlassian.net/browse/MLID-2085) | **Documents tab:** unread badge on the tab header (driven by `useDocumentsTabNotifications(orderId)`), "New" chip on each unread document row for the assigned user, "Mark as Read" button per row (only for the assignee), "Mark all as Read" button at the section header. Bulk action disables per-row buttons while the bulk request is in flight (`886b4080`) to prevent double-acks. Badge color uses the design system's teal primary (`ba602a01`). |
| D3-T3 | [MLID-2086](https://localinfusion.atlassian.net/browse/MLID-2086) | **Order History tab:** every document addition writes one `auditLogs` row per matched active order via the new `recordDocumentAddedToOrders()` service. The audit row uses `field: 'documents'`, `from: null`, `to: { _id, category, source }` — no schema extensions. The renderer learned a new `renderChange` hook on `AuditFieldConfig` so the "Change" cell formats as `New Document received (Document type = <category>)` instead of the default "Original / New value" lines. Same fire-and-forget semantics as the notification trigger; same gating under `ORDER_DOCUMENT_NOTIFICATIONS`. |

### Closing-day refactors (epic polish)

These two refactors landed at the end of the epic on top of the notification work. They are visible to users (UI restructure + table swap) but not feature-flagged — they apply to every order regardless of `ORDER_DOCUMENT_NOTIFICATIONS`.

#### Refactor 1 — Shared Order Details Summary header on all three order-detail routes (`dc594afe`)

**Goal:** the same "Order Details Summary" card (medication name, LISA Order #, Order Type, status chip, plus the 3-column Patient / Provider & Insurance / Fulfillment block) renders for all three tabs of the order detail (Dashboard `/`, Order History `/order-history`, Documents `/documents`).

**Before:** Dashboard had the full summary; Order History rendered its own duplicated 3-column block as its first card; Documents rendered just the title strip with no body. Three different visual shapes.

**Change:**

- Extracted the dashboard's 3-column JSX from `page.tsx` into a new reusable `components/OrderDetailsSummary.tsx`. The component owns its own `useOrderDetails()` consumption and `ORDER_DETAILS` feature-flag gating.
- The layout's header `<Card>` now renders that component unconditionally + the dashboard-only Edit Order / Chevron / Expand-panel pieces (still gated on `isDashboardTab && isNonLeadCategory`).
- All three routes now render their `{children}` **outside** the Card as siblings, gated by their tab predicates. Documents and Order History no longer try to share the Card body with the dashboard's columns.
- The dashboard `page.tsx` is now `return <OrderDetailsSummary />` — visually a no-op (the summary is rendered by the layout for every route now), but kept as a working slot for upcoming dashboard-only content per the backlog.
- Order History's `page.tsx` lost its duplicated 3-column block (and the now-unused `useOrderDetails` import + the early-return guard that fed it). Eight unused CSS classes were dropped from `OrderHistory.module.css`. Tests pruned the seven assertions that exercised the deleted JSX.

#### Refactor 2 — Documents tab migrated to `@repo/ui` DataGrid (`5a38e735`)

**Goal:** the Documents tab matches the playground design (`li-ui-playground/src/pages/OrderDetail.tsx`) and uses the same DataGrid pattern as the orders tracker list page.

**Change:**

- Replaced the hand-rolled `<table>` with `<DataGrid>` from `@repo/ui` wrapped in `<Card padding="none">`.
- Columns: `name` (rendered as `<a target="_blank">` to `viewUrl` — the design-system `Link` doesn't yet accept `target`/`rel`, so plain `<a>` is the pragmatic choice for now), `receivedDate` (formatted "M/D/YYYY h:mm AM/PM" with the "New" chip rendered alongside via `renderCell`, sortable), `category`, and a conditional `actions` column (Mark as Read button when unread + a placeholder `MoreVertIcon` 3-dot button — empty menu for now per the spec, room for future per-row actions).
- DataGrid configured with `rowHeight={78}`, `pageSizeOptions={[25, 50, 100]}`, default sort by `receivedDate desc`, `getRowId={row => row.id}`. Empty state delegated to DataGrid's built-in "No rows".
- Actions column gating preserved — when `ORDER_DOCUMENT_NOTIFICATIONS` is off, the column is not appended at all (3 columns vs. 4).
- Cleaned up a pre-existing latent bug: the previous table referenced classes (`documentsTable`, `viewLink`, `emptyState`) that **never existed** in `OrderDetailsLayout.module.css` — so the table had been silently rendering unstyled. The dead `styles` import is gone.
- Tests rewritten with a smarter `@repo/ui` mock: a `DataGrid` stub that renders `<table>` with `<thead>` and `<tbody>` rows by invoking each column's `renderCell` (or `valueFormatter`). All per-row assertions (Mark-as-Read, "New" chip, link href) still meaningful, without paying the MUI Pro grid render cost in jsdom.

## D4 — what's still ahead (NOT in this PR)

Four follow-on items were scoped during the epic. They do not need to ship before merging this PR — they are post-merge verification work. Tracked in the plan but no Jira sub-tasks yet:

1. **D4-T1** — Execute the 6 manual SignalR-integration test scenarios in staging (cross-tab badge updates, `notification:read` propagation, reconnect catch-up, flag-OFF cleanliness, cross-tab noise boundary).
2. **D4-T2** — Write and execute the backend-trigger test strategy (PHI masking, idempotency, partial bulk failures, legacy email-shaped `assignedTo`, etc.).
3. **D4-T3** — Close the SignalR architectural gaps (no per-user targeting yet, no replay/catch-up beyond mount refetch, payloads not consumed by listeners, no UI for connection health, no automated E2E coverage). Umbrella — to be split into Jira sub-tasks once prioritized.
4. **Azure infra env vars** for SignalR were initially planned as overrides but inlined back into constants in `368ffe87` after we decided the runtime override added complexity without a clear use case (per the `feedback_no_env_vars_without_infra_knowledge` rule — keep it inline; PE adds flexibility when needed). Nothing required from infra side for this PR.

## Changes Overview

- **Branch**: `epic/MLID-2011-order-document-notifications` → `develop`
- **Files changed**: 61
- **Lines added**: +7,673
- **Lines removed**: -568
- **New tests**: ~120 across notification model/service, API routes, document triggers, audit, UI hooks, components, and the migrated DataGrid table

## Files Changed (high-level groupings)

### Backend — SignalR layer
- `apps/web/services/signalr/index.ts` + `index.test.ts` — `broadcastToAll()`, broadcast-path token audience, info-level success logs, hub name correction, inlined constants.

### Backend — notification data model + service (PE under PR #1049)
- `apps/web/models/Notification.ts` + `Notification.test.ts`
- `apps/web/models/NotificationRead.ts`
- `apps/web/services/mongodb/notification.ts` + `notification.test.ts`

### Backend — notification API routes (D2-T1)
- `apps/web/app/api/notifications/route.ts` + test (GET inbox)
- `apps/web/app/api/notifications/counts/route.ts` + test (GET counts, batched per-order)
- `apps/web/app/api/notifications/by-order/[orderId]/route.ts` + test
- `apps/web/app/api/notifications/[id]/read/route.ts` + test (PATCH, target-only)
- `apps/web/app/api/notifications/bulk-read/route.ts` + test (PATCH, target-only)

### Backend — document triggers + audit (D2-T2 / D3-T3)
- `apps/web/services/notifications/documentNotification.ts` + test
- `apps/web/services/audit/documentAudit.ts` + test
- `apps/web/services/mongodb/auditLog.ts` + `auditLog.config.ts` + tests — `renderChange` hook, `documents` field config
- `apps/web/types/auditLog.ts` — `renderChange` hook on `AuditFieldConfig`
- `apps/web/types/orders.ts` — minor field additions consumed by triggers
- `apps/web/app/api/documents/split-pdf/route.ts` + tests — wires upload entry point to `notifyDocumentCreated` + `recordDocumentAddedToOrders`
- `apps/web/services/jobs/definitions/jotFormSubmission/weinfuseUploader.ts` + tests — wires e-order entry point with `intake.source === 'spruce'` detection

### Backend — feature flag
- `apps/web/utils/featureFlags/featureFlags.ts` + test — `ORDER_DOCUMENT_NOTIFICATIONS` enum + default `false`
- `docs/FEATURE_FLAGS.md` — flag row added

### Frontend — Order Tracker (D3-T1)
- `apps/web/app/orders-tracker/[category]/page.tsx` + test — quick-filter cleanup, badge wiring
- `apps/web/app/orders-tracker/Orders/NewOrders.tsx` + test — Order ID column renders the badge using the count map
- `apps/web/app/orders-tracker/hooks/useOrderNotificationCounts.ts` + test — batched count hook with SignalR re-fetch on `documentAdded` / `documentRead`
- `apps/web/components/UI/Tab/Tab.tsx` + test — badge on tab header (used by Documents tab)

### Frontend — Order Details Layout (closing refactor)
- `apps/web/app/orders-tracker/[category]/[orderId]/layout.tsx` + test — header Card now renders for all 3 routes; children are sibling slots below
- `apps/web/app/orders-tracker/[category]/[orderId]/components/OrderDetailsSummary.tsx` (new) — extracted 3-column summary
- `apps/web/app/orders-tracker/[category]/[orderId]/page.tsx` — delegates to `<OrderDetailsSummary />`
- `apps/web/app/orders-tracker/[category]/[orderId]/order-history/page.tsx` + test + `OrderHistory.module.css` — duplicated 3-column block + 8 unused CSS classes deleted

### Frontend — Documents tab (D3-T2 + DataGrid refactor)
- `apps/web/app/orders-tracker/[category]/[orderId]/documents/page.tsx` + test — DataGrid migration; New chip; Mark as Read; Mark all as Read; bulk-disables-per-row
- `apps/web/app/orders-tracker/[category]/[orderId]/documents/useDocumentsTabNotifications.ts` + test — hook backing the tab badge + per-document unread map

### Frontend — Order History (D3-T3)
- `apps/web/app/orders-tracker/[category]/[orderId]/order-history/components/AuditLogTable.tsx` + test — uses `renderChange` hook for the `documents` field

### Documentation
- `docs/notifications-data-model.md`
- `docs/signalr/signalr-integration.md`
- `docs/signalr/order-documents-signalr-notifications.md`

## Test Plan

```bash
# From apps/web/

# Notification data model + service
npx jest --testPathPattern="models/Notification" --no-coverage
npx jest --testPathPattern="services/mongodb/notification" --no-coverage

# SignalR
npx jest --testPathPattern="services/signalr" --no-coverage

# Document triggers
npx jest --testPathPattern="services/notifications/documentNotification" --no-coverage
npx jest --testPathPattern="services/audit/documentAudit" --no-coverage
npx jest --testPathPattern="api/documents/split-pdf" --no-coverage
npx jest --testPathPattern="weinfuseUploader" --no-coverage

# API routes
npx jest --testPathPattern="api/notifications" --no-coverage

# Audit log
npx jest --testPathPattern="services/mongodb/__tests__/auditLog" --no-coverage

# Order Tracker / hooks
npx jest --testPathPattern="orders-tracker/Orders/NewOrders" --no-coverage
npx jest --testPathPattern="orders-tracker/hooks/useOrderNotificationCounts" --no-coverage

# Order detail subtree (layout + dashboard + documents + order-history)
npx jest --testPathPattern="orders-tracker/\[category\]/\[orderId\]" --no-coverage

# Tab badge component
npx jest --testPathPattern="components/UI/Tab" --no-coverage

# Type check
npx tsc --noEmit
```

### Manual smoke test (browser)

With `ORDER_DOCUMENT_NOTIFICATIONS` ON:

1. Visit `/orders-tracker/new`. Orders with outstanding unread documents show a teal badge next to the Order ID. Reload and confirm the count matches `/api/notifications/counts`.
2. Visit an order with unread documents → Documents tab. Confirm the tab header badge, the per-row "New" chip on rows owned by the current user, the "Mark as Read" button (only on rows owned by the current user), the "Mark all as Read" button at the top right.
3. Click a row's "Mark as Read" — chip disappears, badge counts decrement, button disabled while in flight.
4. Click "Mark all as Read" with multiple unread rows — all chips disappear, all per-row buttons remain disabled until the bulk request settles.
5. Visit Order History tab → confirm document additions appear with `Field: New Document`, `Category: Documents`, and the "New Document received (Document type = X)" change text.
6. Open the same order in two browser tabs (or two different users with the same flag-on session) and have one mark a document as read. The other tab's badge counts should update via SignalR within ~1s without a manual reload.
7. Toggle the flag OFF in `/admin/feature-flags` → all badges, chips, "Mark as Read" buttons, and audit-trail document rows disappear immediately on next page load. API routes return 404.

### DataGrid + shared header smoke test

8. Visit Order Dashboard → confirm the 3-column summary (Patient / Provider & Insurance / Fulfillment) renders inside the header card with Edit Order + Chevron buttons.
9. Switch to Order History → confirm the **same** 3-column summary renders in the header card. Below the card, the audit log table renders.
10. Switch to Documents → confirm the **same** 3-column summary renders in the header card. Below the card, the DataGrid renders with sortable Received column, "New" chips, pagination footer (default 25 rows, options 25/50/100), and DataGrid's built-in "No rows" message when there are no documents.

## Known non-blocking items / follow-ups

- **D4 verification work** (T1/T2/T3 above) is post-merge. Each will produce a test report and, if needed, a bugfix branch. T3 is an architectural-gaps umbrella that needs to be split into Jira sub-tasks before estimation.
- **`@repo/ui` `Link` component does not accept `target`/`rel`.** The Documents DataGrid uses a plain `<a target="_blank">` for the document name. If the design system gains those props later, this is a one-line follow-up.
- **DataGrid 3-dot menu is empty** — placeholder for future per-row actions (open in new tab, download, etc.) per spec. Renders as a `MoreVertIcon` button with no menu attached. Click does nothing today; this is intentional.
- **SignalR token refresh** — per-user tokens expire after 1h. The client context doesn't re-negotiate on close, so long-running sessions silently lose real-time updates after an hour (REST APIs still serve correct state on next page load). Captured in D4-T3 gap inventory; will be addressed once we see it bite in staging.

## Reference

- **Architecture**: `docs/signalr/signalr-integration.md`, `docs/signalr/order-documents-signalr-notifications.md`
- **Data model**: `docs/notifications-data-model.md`

## Jira

- **Epic**: [MLID-2011](https://localinfusion.atlassian.net/browse/MLID-2011) — Orders: Document Notifications
- D0-T1: [MLID-2079](https://localinfusion.atlassian.net/browse/MLID-2079) — SignalR infra (Anatoliy)
- D1-T1: [MLID-2080](https://localinfusion.atlassian.net/browse/MLID-2080) — Feature flag + broadcast primitive
- D1-T2: [MLID-2081](https://localinfusion.atlassian.net/browse/MLID-2081) — Notification data model
- D2-T1: [MLID-2082](https://localinfusion.atlassian.net/browse/MLID-2082) — Notification API routes
- D2-T2: [MLID-2083](https://localinfusion.atlassian.net/browse/MLID-2083) — Document creation triggers
- D3-T1: [MLID-2084](https://localinfusion.atlassian.net/browse/MLID-2084) — Order Tracker badges
- D3-T2: [MLID-2085](https://localinfusion.atlassian.net/browse/MLID-2085) — Documents tab UI
- D3-T3: [MLID-2086](https://localinfusion.atlassian.net/browse/MLID-2086) — Order History audit trail
