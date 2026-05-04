# How SignalR is implemented in this app

> Technical reference: the architecture of real-time notification delivery used by MLID-2011 (Order Tracker document notifications). If you're looking for **how to test** it, see `MLID-2011-testing-signalr-integration.md`.

---

## 1. Architecture Overview

### Transport

- **Library**: `@microsoft/signalr` on the client, custom `SignalRService` wrapper on the server.
- **Backend service**: Azure SignalR Service. Server broadcasts via the REST API; clients receive over WebSocket.
- **Broadcast model**: all events are **broadcast-to-all-connected-clients**. There is no server-side per-user or per-group routing. Every client receives every event and filters locally.
- **Connection auth**: the negotiate endpoint requires a NextAuth session. Once connected, events are not re-filtered per user — broadcast reach is determined by who is currently connected.
- **Feature flag**: `ORDER_DOCUMENT_NOTIFICATIONS` gates both dispatch and listener-reaction behavior.

### Connection lifecycle (client)

- **Provider**: `SignalRProvider` in `apps/web/utils/context/signalr-context.tsx`.
- **Mount point**: `apps/web/app/providers.tsx` (wraps the whole app; always attempts to connect on page load, regardless of feature flag).
- **Hook**: `useSignalR()` exposes `{ connection, isConnected }`.
- **Lifecycle**:
  1. On provider mount → `POST /api/signalr/negotiate`.
  2. Receives `{ url, accessToken }`.
  3. Builds a `HubConnection` with auto-reconnect backoff `[0, 2000, 5000, 10000, 30000]` ms and `LogLevel.Warning`.
  4. Calls `connection.start()`.
  5. On `onclose` / `onreconnecting` → `isConnected = false`. On `onreconnected` → `isConnected = true`.
  6. On unmount → `connection.stop()`.

### Negotiate endpoint (server)

- **File**: `apps/web/app/api/signalr/negotiate/route.ts`.
- **Auth**: requires NextAuth session; otherwise 401.
- **Returns**: `{ url, accessToken }` suitable for `HubConnectionBuilder.withUrl`.

### Configuration

Env vars consumed by the SignalR service wrapper:

| Variable | Default | Purpose |
|---|---|---|
| `SIGNALR_HUB_NAME` | `li-hub` | Target hub for REST `:send` calls |
| `SIGNALR_TOKEN_EXPIRY` | `1h` | JWT expiry for client connection |
| `SIGNALR_REST_API_VERSION` | `2022-06-01` | Azure SignalR REST API version |
| `SIGNALR_KV_SECRET_NAME` | `signalr-primary-connection-string` | Key Vault secret name for the connection string |

Key Vault naming in Terraform-managed environments: `li-<env>-signalr-primary-connection-string`.

---

## 2. Actors

### Dispatchers (server → clients)

| Event | File | Function | Broadcast call | Triggered by |
|---|---|---|---|---|
| `documentAdded` | `apps/web/services/notifications/documentNotification.ts` | `notifyDocumentCreated()` (line ~68) | `signalr.broadcastToAll('documentAdded', payload)` (line 122) | `POST /api/documents/split-pdf` (both upload branches) and the JotForm `weinfuseUploader` job |
| `notification:read` | `apps/web/app/api/notifications/[id]/read/route.ts` | `PATCH` handler | `signalr.broadcastToAll('notification:read', …)` | User clicks "Mark as Read" on a single notification |
| `notification:read` | `apps/web/app/api/notifications/bulk-read/route.ts` | `PATCH` handler | `signalr.broadcastToAll('notification:read', …)` | User clicks "Mark all as read" / bulk action |

**Payload shapes:**
- `documentAdded` → `{ documentId, orderId, patientId, source, category }`
- `notification:read` (single) → `{ notificationId, userId }`
- `notification:read` (bulk) → `{ notificationIds: string[], userId }`

**Broadcast utility**: `apps/web/services/signalr/index.ts` — wraps the Azure SignalR REST `:send` endpoint. Reads the connection string from Key Vault (`SIGNALR_KV_SECRET_NAME`).

### `notifyDocumentCreated` — behavior details

Lives in `apps/web/services/notifications/documentNotification.ts`. Per-call behavior:

1. Feature flag `ORDER_DOCUMENT_NOTIFICATIONS` is checked first — when `false`, function returns early and nothing is written or broadcast.
2. Resolves assigned orders for the patient via `we_infuse_id_IV`.
3. For each order with a non-empty `assignedTo`, inserts a `notifications` row with `type: 'user'`, `targetUserId: <assignedTo>`, `entities: [{ order }, { document }]`.
4. Calls `signalr.broadcastToAll('documentAdded', …)` once with the document + order context.
5. Errors inside the insert/broadcast loop are caught and logged — function is **fire-and-forget** and never throws back to the caller.

### Listeners (client → UI)

Both listener hooks subscribe to both events and use them as **signals to refetch**, not as a source of truth. Payloads are currently ignored.

| Hook | File | Events subscribed | Side effect | Refetch endpoint |
|---|---|---|---|---|
| `useDocumentsTabNotifications(orderId)` | `apps/web/app/orders-tracker/[category]/[orderId]/documents/useDocumentsTabNotifications.ts` | `documentAdded`, `notification:read` (lines 111–112) | Re-runs `fetchNotifications()` → updates per-document unread state and the Documents-tab badge | `GET /api/notifications/by-order/{orderId}` |
| `useOrderNotificationCounts(orderIds)` | `apps/web/app/orders-tracker/hooks/useOrderNotificationCounts.ts` | `documentAdded`, `notification:read` (lines 68–69) | Re-runs `fetchCounts()` → updates badge counts on the Orders list | `POST /api/notifications/counts` (body: `{ orderIds }`) |

**UI touchpoints:**
- `useOrderNotificationCounts` — `/orders-tracker/new` and `/orders-tracker/maintenance` tables: per-row badge next to `displayId`.
- `useDocumentsTabNotifications` — `/orders-tracker/{cat}/{orderId}/documents` page: tab badge (wired in `layout.tsx` via `tabItems[].badge`) and per-row New/unread chips inside the tab.

### Collections touched

- `notifications` — one row per assigned order per document, `type: 'user'`.
- `notificationreads` — one row per (user, notification) mark-as-read event.

---

## 3. What We Have vs. What We Don't

### ✅ What we have

- Negotiate endpoint with NextAuth-gated session check.
- Client provider mounted at the app root with auto-reconnect.
- Two server-side dispatchers (`documentAdded`, `notification:read`) covering the three major write paths (manual upload, JotForm job, mark-read).
- Two client-side listeners reacting to both events by refetching state.
- Feature flag `ORDER_DOCUMENT_NOTIFICATIONS` gating the backend dispatch.

### ❌ Gaps / things we don't have

1. **No per-user / per-location / per-group targeting.** Every connected client gets every event and relies on local filtering. Fine for low volume, potentially noisy at scale.
2. **No replay / catch-up on reconnect.** Events fired while a client is disconnected are lost. Mitigated in practice because both listener hooks do an initial `fetch*()` on mount/connect, so state becomes consistent again; no real-time update is retried.
3. **No SignalR event for notifications created outside `notifyDocumentCreated()`.** If some other code path writes directly to the `notifications` collection, connected clients won't know until they refetch or navigate.
4. **Payloads are not consumed.** Listeners ignore `documentAdded.payload` and always refetch the full state. This is simpler but means the client does N+1 fetches on bursts of events; no client-side "debouncing/coalescing" exists today.
5. **No UI indicator of SignalR connection health.** `isConnected` is tracked in context but not surfaced anywhere — if SignalR is down, badges silently stop updating in real time (users won't know; refreshing the page still works).
6. **Dispatch side is not feature-flagged separately from write side.** `notifyDocumentCreated()` checks `ORDER_DOCUMENT_NOTIFICATIONS` once and both writes notifications *and* broadcasts under the same flag; there's no way to "write but don't broadcast" for a staged rollout.
7. **No automated end-to-end (Playwright) tests** covering the broadcast→UI flow — current test coverage is unit-level (mocks for `SignalRService` and `connection.on`).
8. **No dead-letter / observability.** If `broadcastToAll` throws, the error is caught and logged; there's no retry and no alert.

---

## 4. Open Architectural Questions

1. Do we want a visible "disconnected from real-time updates" banner when `isConnected === false` for more than N seconds?
2. Do we want to start coalescing burst events on the client (e.g., debounce refetch by 500 ms)?
3. Do we want to move from broadcast-to-all toward per-user or per-location groups once we have another consumer besides MLID-2011?
4. Should the backend expose a `/api/signalr/health` that reports whether the REST API is reachable, so the SRE dashboard can page on outages?
