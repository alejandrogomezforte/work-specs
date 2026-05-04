# MLID-2011 — SignalR Integration Testing

> **Scope**: verify that SignalR events emitted from the backend reach the browser and cause the Order Tracker UI to update in real time.
>
> **Companion docs**:
> - `how-signalR-implementation-works.md` — architecture, dispatchers, listeners, gaps. Read this first if you need to understand *what* we're testing.
> - `MLID-2011-testing-backend-triggers.md` — covers the DB write side (notification inserts, reads).

---

## Goal

Prove that an event emitted by the backend causes at least one other connected client to re-render within a reasonable time. Secondary: prove that the client survives disconnects/reconnects without stale state.

---

## 1. Prerequisites

- VPN enabled (staging MongoDB + staging SignalR service).
- Feature flag `ORDER_DOCUMENT_NOTIFICATIONS` turned **ON** in `/admin/feature-flags`.
- A seeded test order with `assignedTo` set to a known user — e.g., order `69cf9aee03a194bd1a65a199` (`05Gy`) from `MLID-2011-data-testing.md`.
- **Two browser sessions** (either two browsers, or one normal + one incognito) — we need a "dispatcher" tab and a "listener" tab. Using the same browser profile for both tabs is fine; same user session is fine and actually simpler (both tabs will care about the same order).
- DevTools open in the "listener" tab (Network panel + Console).

---

## 2. Pre-flight: verify the WebSocket is alive

In the listener tab, open DevTools → Network → filter by `WS` (WebSocket).

- **Expected**: one WebSocket connection to the Azure SignalR Service URL (hostname matches the value returned by `/api/signalr/negotiate`).
- **Expected**: the connection status is `101 Switching Protocols`.
- **If missing**: the provider is not connecting. Check:
  - Console for `'SignalR connection failed: …'`.
  - `/api/signalr/negotiate` response in Network — status should be 200 with `{ url, accessToken }`.
  - If negotiate returned 401, the session isn't recognised — sign out and back in.

---

## 3. Scenarios

### Scenario A — Order list badge updates live

**Purpose**: prove `useOrderNotificationCounts` receives `documentAdded` and the Orders list badge ticks up without a refresh.

1. Listener tab: go to `/orders-tracker/new`. Scroll to see order `05Gy`. Note its current badge count (if any).
2. Dispatcher tab: upload a new document for Justin Tracey QA (patient `69cf89d202323d384fe6dc8e`) via `POST /api/documents/split-pdf` (or the UI equivalent).
3. **Expected in listener tab within ~2 s**:
   - Without any user action, the badge next to `05Gy` increments by 1.
   - Network panel shows a `POST /api/notifications/counts` request fired from the listener tab (triggered by the `documentAdded` event handler).
   - WS frames panel shows an incoming message with `target: "documentAdded"`.

### Scenario B — Documents-tab badge updates live

**Purpose**: prove `useDocumentsTabNotifications` receives `documentAdded` and the per-order Documents tab badge updates.

1. Listener tab: navigate to `/orders-tracker/new/69cf9aee03a194bd1a65a199/documents`.
2. Dispatcher tab: upload a document for the same patient.
3. **Expected in listener tab within ~2 s**:
   - The Documents tab badge on that page increases.
   - Network panel shows a `GET /api/notifications/by-order/69cf9aee03a194bd1a65a199` request fired.
   - If a new row appears in the documents list, it should carry a `New` chip.

### Scenario C — `notification:read` propagates

**Purpose**: prove that marking a notification as read in one tab clears the badge in another.

1. Tab A and Tab B: both on `/orders-tracker/new`; `05Gy` shows a non-zero badge.
2. Tab A: open the `05Gy` Documents tab. Click "Mark as Read" on a notification row (or the bulk action).
3. **Expected in Tab B within ~2 s**:
   - The `05Gy` badge decrements (or disappears) without a page refresh.
   - WS frames panel shows an incoming message with `target: "notification:read"`.
   - Network panel shows Tab B re-fetching counts after the event.

### Scenario D — Reconnect catch-up

**Purpose**: verify that a client reconnecting picks up missed state via the initial `fetch*()` calls (even though individual events fired during disconnect are lost).

1. Listener tab: on `/orders-tracker/new`, badge count visible.
2. Put the machine to sleep or turn off Wi-Fi for ~60 s.
3. While offline, have the dispatcher tab upload 2 documents (which will emit 2 `documentAdded` events the listener cannot receive).
4. Bring the listener tab back online.
5. **Expected**:
   - `isConnected` flips back to `true`.
   - Listener runs initial `fetchCounts()` / `fetchNotifications()` and the badge reflects the new total (not just +0, i.e., both missed uploads show up).
   - No console errors.

### Scenario E — Feature flag OFF behavior

**Purpose**: prove the flag cleanly disables real-time updates without breaking the UI.

1. In `/admin/feature-flags`, turn `ORDER_DOCUMENT_NOTIFICATIONS` **OFF**.
2. Dispatcher tab: upload a document.
3. **Expected**:
   - No row inserted in `notifications` (check Mongo).
   - No `documentAdded` frame in the WS channel.
   - Listener tab badge does not change.
   - No errors in the console.

### Scenario F — Cross-tab noise boundary (architecture sanity check)

**Purpose**: confirm the broadcast-to-all model doesn't visibly leak another user's state. (See gap #1 in `how-signalR-implementation-works.md`.)

1. Log two browser profiles in as **different users**: User X (not assigned to `05Gy`) in Tab A; User Y (assigned to `05Gy`) in Tab B.
2. Trigger `documentAdded` for `05Gy` (via a doc upload on Justin Tracey QA).
3. **Expected**:
   - Tab A receives the WS frame (confirming broadcast-to-all).
   - Tab A's `useOrderNotificationCounts` refetches `/api/notifications/counts`, but the server-side query filters to notifications for User X — so X's badge for `05Gy` stays 0.
   - Tab B's badge ticks up.
4. **This is the expected architecture** per the "broadcast-to-all, filter locally" design. If X's badge incorrectly shows Y's notifications, that is a real bug (most likely in the `counts` or `by-order` API).

---

## 4. Automated tests to consider (future work)

- **Playwright**: drive two browser contexts; trigger an upload in context A; assert the badge in context B within a timeout.
- **Contract tests on payload shapes**: pin the shape of `documentAdded` + `notification:read` payloads so refactors don't drift.
- **Unit tests for `broadcastToAll`**: verify error paths (SignalR REST 5xx → logged + swallowed) and that payload is JSON-encoded correctly.

---

## 5. Debugging Tips

- **See what's on the wire**: Chrome DevTools → Network → WS → click the connection → "Messages" tab. Incoming frames carry `type: 1, target: 'documentAdded', arguments: […]`.
- **Force a reconnect**: toggle the Network tab's "Offline" switch, wait ~5 s, turn it back online — the provider will hit its backoff schedule and reconnect. (The `HubConnection` instance is not exported globally; for harder-to-reproduce issues, temporarily add `window.__signalr = connection` in the provider.)
- **Check negotiate response directly**:
  ```bash
  curl -b <cookie-jar> -X POST https://<host>/api/signalr/negotiate
  ```
  Should return `{ "url": "https://<signalr>.service.signalr.net/client/...", "accessToken": "<jwt>" }`.
- **Server-side broadcast failures** are logged via `logger.error` — grep staging logs for `'Failed to broadcast documentAdded event'` / `'broadcast'` near the timestamp of the failing test.
- **Stale badge after reconnect?** Inspect `fetchCounts` / `fetchNotifications` in the Network tab after `onreconnected` fires — they should run exactly once.
