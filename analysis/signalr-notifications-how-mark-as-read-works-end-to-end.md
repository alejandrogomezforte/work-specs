# How "Mark as Read" Works End-to-End

A complete trace of one **Mark as Read** click on a document notification — from the button click to the UI re-render across every connected browser. Includes file paths, line numbers, timing estimates, and key insights about why the system is wired this way.

**Related**:
- `docs/signalr/signalr-integration.md` — generic SignalR layer (the foundation this builds on)
- `docs/signalr/order-documents-signalr-notifications.md` — feature documentation for engineers
- `docs/agomez/postmortem/why-signalr-negotiate-was-failing.md` — the three bugs that had to be fixed before this flow worked

---

## 1. Sequence diagram

```
┌─────────┐     ┌────────────────┐     ┌──────────────────┐     ┌──────────┐     ┌─────────────┐
│ Browser │     │ Next.js API    │     │ Mongo            │     │ Next.js  │     │ Azure       │
│ (you)   │     │ /notif/.../    │     │ notifications +  │     │ SignalR  │     │ SignalR     │
│         │     │ read/route.ts  │     │ notificationreads│     │ service  │     │ (REST + WS) │
└────┬────┘     └───────┬────────┘     └────────┬─────────┘     └────┬─────┘     └──────┬──────┘
     │                  │                       │                    │                  │
     │  1. PATCH        │                       │                    │                  │
     ├─────────────────▶│                       │                    │                  │
     │                  │  2. session + flag    │                    │                  │
     │                  │  3. mark as read      │                    │                  │
     │                  ├──────────────────────▶│                    │                  │
     │                  │  insertOne            │                    │                  │
     │                  │◀──────────────────────┤                    │                  │
     │                  │                       │                    │                  │
     │                  │  4. broadcast         │                    │                  │
     │                  ├───────────────────────────────────────────▶│                  │
     │                  │                       │                    │  5. POST :send   │
     │                  │                       │                    ├─────────────────▶│
     │                  │                       │                    │  202 Accepted    │
     │                  │                       │                    │◀─────────────────┤
     │                  │  6. void return       │                    │                  │
     │                  │◀───────────────────────────────────────────┤                  │
     │                  │                       │                    │                  │
     │  7. 200 OK       │                       │                    │                  │
     │◀─────────────────┤                       │                    │                  │
     │                                                                                  │
     │ ──────────────── SignalR fans out to ALL connected clients ─────────────────────│
     │                                                                                  │
     │  8. WS frame: {"target":"notification:read","arguments":[{notificationId,userId}]}│
     │◀─────────────────────────────────────────────────────────────────────────────────┤
     │                                                                                  │
     │  9. listener fires (useDocumentsTabNotifications, useOrderNotificationCounts)    │
     │                                                                                  │
     │ 10. GET /api/notifications/by-order/{orderId}                                    │
     ├─────────────────▶                                                                │
     │ 11. POST /api/notifications/counts                                               │
     ├─────────────────▶                                                                │
     │                                                                                  │
     │ 12. UI re-renders: chip removed, button hidden, badge decremented                │
```

---

## 2. Phase-by-phase timeline

Approximate wall-clock timing for a healthy local dev session. Cold-start (first hit on a route) can be 5–15× slower because of Next.js dev-mode JIT compilation.

### T+0ms — User clicks the button

The button is rendered in the Documents tab table at `apps/web/app/orders-tracker/[category]/[orderId]/documents/page.tsx:170-178`:

```tsx
<Button
  variant="secondary"
  size="xs"
  disabled={inFlightIds.has(summary.notificationId)}
  onClick={() => handleMarkAsRead(summary.notificationId)}
>
  Mark as Read
</Button>
```

`summary.notificationId` is the ObjectId hex string (e.g., `69dfe246adc73ef87c73c04d`), resolved by `useDocumentsTabNotifications` from the server-fetched notification list.

### T+1ms — `handleMarkAsRead` runs

Defined in the same `page.tsx` (around line 50–65). Three things happen:

1. **Optimistic UI guard.** `inFlightIds` set is updated to include this notification ID. The button's `disabled` prop flips to true, preventing double-clicks.
2. **Fire the PATCH.**
   ```ts
   await fetch(`/api/notifications/${notificationId}/read`, {
     method: 'PATCH',
     headers: { 'Content-Type': 'application/json' },
   });
   ```
3. **In a `finally`**, the ID is removed from `inFlightIds` regardless of success.

No optimistic state change beyond the disabled button — the actual chip removal happens via the SignalR refetch path, not by mutating local state.

### T+5–20ms — Browser dispatches the request

Browser sends the PATCH. NextAuth session cookie rides along automatically.

### T+25ms — Server receives PATCH

Handler at `apps/web/app/api/notifications/[id]/read/route.ts`. Sequence inside the handler:

```ts
// line 22-25 — auth gate
const session = await getServerSession(authOptions);
if (!session?.user) return 401;

// line 32-40 — feature flag
const isEnabled = await FeatureFlagService.getFeatureFlag(
  FeatureFlag.ORDER_DOCUMENT_NOTIFICATIONS
);
if (!isEnabled) return 404;

// line 42-54 — mutate, broadcast, respond
const { id } = params;
const userId = session.user.id;
await markUserNotificationAsRead(id, userId); // step A
const signalr = await getSignalRService();
await signalr.broadcastToAll('notification:read', {
  notificationId: id,
  userId,
}); // step B
return NextResponse.json({ success: true }, { status: 200 });
```

### T+30ms — Step A: `notificationreads.insertOne`

`markUserNotificationAsRead` (`apps/web/services/mongodb/notification.ts:317`) calls:

```ts
await NotificationRead.create({
  notificationId,
  userId,
  readAt: new Date(),
});
```

Mongoose translates this to one `insertOne` against the `notificationreads` collection. The unique compound index `(notificationId, userId)` makes it idempotent — re-clicking a stale UI returns silently rather than erroring (duplicate-key error 11000 is caught and logged at info level).

The `notifications` document itself is **not touched** — its `isRead` / `readBy` / `readAt` fields are inert for `type: 'user'` notifications.

### T+50ms — Step B: server-side broadcast

`broadcastToAll` (`apps/web/services/signalr/index.ts:106-147`):

1. Build the URL once: `broadcastPath = "${endpoint}/api/hubs/lihub/:send"`.
2. Mint a JWT with `aud = broadcastPath` (the full URL path including `/:send`, what Azure SignalR REST API 2022-06-01 requires).
3. POST to `${broadcastPath}?api-version=2022-06-01` with header `Authorization: Bearer <jwt>` and body:
   ```json
   {
     "target": "notification:read",
     "arguments": [
       { "notificationId": "69dfe246adc73ef87c73c04d", "userId": "<your userId>" }
     ]
   }
   ```

### T+90ms — Azure SignalR processes

Azure validates the JWT (signature against the AccessKey from the connection string + audience match) and fans the message out to every WebSocket client currently connected to hub `lihub`. Returns **202 Accepted** to the server.

### T+95ms — Server logs success and returns

`broadcastToAll` writes one log line:

```
[INFO] SignalR broadcast sent {
  eventName: 'notification:read',
  hubName: 'lihub',
  status: 202,
  payload: { notificationId: '...', userId: '...' }
}
```

The route handler returns `200 OK` to the browser.

### T+100ms — Browser receives 200

`handleMarkAsRead`'s `await fetch(...)` resolves. The `finally` block removes the notification ID from `inFlightIds`. Button is re-enabled — but it's about to disappear anyway.

**At this point, nothing visible has changed in the UI yet.** The PATCH succeeded, but the React state that drives the chip/button isn't connected to the PATCH response — it's connected to the SignalR refetch path.

### T+100–120ms — WebSocket frame arrives

In parallel with the HTTP response, Azure SignalR pushes a frame to **every connected client** (including the one that triggered it):

```json
{
  "type": 1,
  "target": "notification:read",
  "arguments": [{ "notificationId": "69dfe246adc73ef87c73c04d", "userId": "<userId>" }]
}
```

You can see this live in Chrome DevTools → Network → filter `signalr` → click the `wss://...` row → **Messages** tab.

### T+120ms — `@microsoft/signalr` SDK dispatches

The `HubConnection` built by `SignalRProvider` (`apps/web/utils/context/signalr-context.tsx`) reads the frame and invokes every handler registered with `connection.on('notification:read', ...)`.

There are **two** handlers registered (when the relevant pages are mounted):

1. `useDocumentsTabNotifications` — mounted on `/orders-tracker/[category]/[orderId]/documents`.
2. `useOrderNotificationCounts` — mounted on `/orders-tracker/new` and `/orders-tracker/maintenance`.

The handlers ignore the payload; they just call their refetch function.

### T+125ms — Refetch fires

#### From the Documents page

`useDocumentsTabNotifications` (`apps/web/app/orders-tracker/[category]/[orderId]/documents/useDocumentsTabNotifications.ts`) calls `fetchNotifications()` → `GET /api/notifications/by-order/${orderId}`.

The route returns the latest list of notifications for the order, including a per-document join against `notificationreads` so the client can compute `isDocumentUnreadForUser` for every document.

#### From the Orders list (if open in another tab)

`useOrderNotificationCounts` (`apps/web/app/orders-tracker/hooks/useOrderNotificationCounts.ts`) calls `fetchCounts()` → `POST /api/notifications/counts` with `{ orderIds: string[] }`.

The route runs an aggregation that returns total + unread counts per order. Unread is computed via `$facet` + a `$lookup` against `notificationreads`, matching the **target** user (not the viewer) so badges are globally consistent.

### T+200ms — Server query completes

Both endpoints query MongoDB. `notifications` indexes serve them:

- `by-order/[orderId]` uses `(entities.entityType, entities.entityId)`.
- `counts` uses the same index plus a `$lookup` into `notificationreads`.

For the notification you just acted on, the query now joins it with the freshly inserted `notificationreads` row, so:

- `isDocumentUnreadForUser(summary, userId) === false`
- `unreadCount` for the order decremented by 1.

### T+220ms — UI re-renders

The hook's state updates with the new server data. React reconciles, and the page renders with the new state:

#### Documents page (the page that triggered the click)

`page.tsx:142-180` — for the row of the notification just acted on:

- `summary.isReadByTarget` is now true.
- `showNewChip = isNotificationsEnabled && isDocumentUnreadForUser(summary, userId)` evaluates to **false**.
- The "New" chip element is no longer rendered.
- The "Mark as Read" button cell renders nothing (the conditional `{showNewChip && summary && <Button .../>}` short-circuits).

#### Documents tab badge

The badge is wired in the layout file (`apps/web/app/orders-tracker/[category]/[orderId]/layout.tsx`) via `tabItems[].badge`, fed from the same hook. The badge value drops by 1.

#### Orders list (other tabs, if any)

If another browser tab has `/orders-tracker/new` open, `useOrderNotificationCounts` already refetched. The per-row badge next to the order's `displayId` decrements by 1.

### T+220ms — UI is consistent

All connected clients now reflect the new state. Total wall-clock from click to fully-updated UI: typically **200–300 ms** in local dev (longer in cold-start scenarios because Next.js dev mode JIT-compiles routes the first time they're hit).

---

## 3. Key design insights

### "The PATCH response doesn't drive the UI"

You might expect the PATCH to update local React state directly, but that's not how this is wired. The PATCH returns 200 to confirm the server work succeeded, but it doesn't return updated state. The UI updates **only via the SignalR-triggered refetch**.

This has two consequences:

1. **Same code path for every client.** The user who clicked sees the UI update via the same mechanism as everyone else — there's no "fast path" for the actor and "slow path" for spectators. Single source of truth: MongoDB, surfaced via REST refetch.
2. **Robust to missed events.** If the SignalR connection happened to be dropping at the moment of the broadcast, the user-who-clicked still has the new state on their next page navigation (REST is the source of truth). But typically the user-who-clicked is connected, so they get the live update.

### "Listeners ignore the payload"

The handler doesn't pull `notificationId` from the frame and surgically update local state. It calls a full refetch. Why:

- **Simpler.** No payload-shape changes required when the data model evolves.
- **Authoritative.** The refetched data reflects whatever else may have changed in MongoDB since the last fetch.
- **Self-healing.** If a previous frame was missed (e.g., the page just mounted), this frame's refetch corrects all accumulated drift in one shot.

The cost: more REST round-trips. For a click-once, occasionally-fired event, that's negligible. For a high-frequency event source (which this is not), debouncing would be worth adding.

### "Broadcast-to-all means the actor's tab also gets the frame"

When you click Mark as Read in tab A, your *own* WebSocket connection receives the frame just like every other client's does. There's no per-user routing. This is why a single tab works end-to-end — your refetch is triggered by Azure echoing your own broadcast back to you.

### "Two listener hooks, one event"

Because broadcasts are fire-and-forget and clients filter locally, multiple hooks on the same page (Documents tab badge in the layout + chip rendering in the table) share one `connection.on('notification:read', ...)`-style subscription per hook. Each hook independently:

- Subscribes on mount.
- Unsubscribes on unmount via `connection.off`.
- Refetches on event.

If both hooks are mounted simultaneously, you'll see two REST calls in DevTools (one per hook) — that's normal.

### "Notification doc is shared, read state is per-user"

The notification document in `notifications` represents one event ("a document arrived for this order"). It is **not** copied per recipient. Each user's read receipt is a row in `notificationreads`. This:

- Saves write amplification when many users target the same notification.
- Decouples authorship of the notification from per-user dismissal semantics.
- Makes "global badge, per-user dismissal" possible — the count query joins both collections to compute team-visible totals while preserving individual read state.

### "The route does not return updated counts"

The PATCH returns `{ success: true }`. It does not return the new badge count or the updated notification list. That's intentional:

- The PATCH is a **command** ("mark this read"), not a **query**.
- The query side is a separate concern, served by `GET /api/notifications/by-order/...` and `POST /api/notifications/counts`.
- Clients never need to combine the two, because the SignalR event triggers the refetch automatically. CQRS-flavored separation, even if not formal.

---

## 4. Failure modes mapped to this timeline

| Broken link | Where in the timeline | Symptom |
| --- | --- | --- |
| PATCH 401 | T+25ms | NextAuth session expired. UI shows nothing changed. Toast or redirect (depending on app-level handling). |
| PATCH 404 | T+32ms | `ORDER_DOCUMENT_NOTIFICATIONS` feature flag is off. Should not reach this state in normal use because the UI hides the button when the flag is off. |
| PATCH 500 | T+50ms or T+90ms | Either `markUserNotificationAsRead` threw (DB unreachable) or `broadcastToAll` threw (SignalR rejection — most likely 401 from Azure). The notification was already inserted in `notificationreads` if the failure was at T+90ms; only the broadcast failed. The user who clicked won't see the UI update until they refresh. Other users won't see it until the next refetch trigger. Server log `[ERROR] SignalR broadcast failed { ..., bodyText }` shows the exact reason. |
| 200 response, no WebSocket frame | T+90ms | Hub-name validation error or WebSocket disconnected. Server log shows `[INFO] SignalR broadcast sent` but DevTools WebSocket Messages tab is silent. Check `wss://...` connection state in Network tab. |
| Frame arrives, no UI change | T+120ms | Listener hook isn't subscribed (e.g., `connection` was null at mount time and the effect didn't re-run when it became truthy). Add a `console.log` in the handler to confirm reception, then trace the refetch path. |
| Refetch returns stale data | T+200ms | Read replica lag (rare in this setup). If reproducible, check the DB connection string and any read-preference settings. |

The server logs `[INFO] SignalR broadcast sent` on successful broadcasts and `[ERROR] SignalR broadcast failed` (with `bodyText` from Azure) on failures. Those two lines are the canonical diagnostic — if you ever see weird behavior, start there.

---

## 5. Manual reproduction

Setup and execution of this exact flow are documented in `docs/order-documents-signalr-notifications.md` §6. To revert state for re-testing:

```js
db.notificationreads.deleteOne({
  notificationId: ObjectId('<notification _id>'),
  userId: '<your userId>',
});
```

Do **not** delete from `notifications` to revert — the notification doc is shared across the team and unrelated to read state for `type: 'user'`.

---

## 6. File index for this flow

In timeline order:

| Phase | File |
| --- | --- |
| Click target | `apps/web/app/orders-tracker/[category]/[orderId]/documents/page.tsx` |
| `handleMarkAsRead` | same file (line ~50–65) |
| PATCH route | `apps/web/app/api/notifications/[id]/read/route.ts` |
| `markUserNotificationAsRead` | `apps/web/services/mongodb/notification.ts:317` |
| Mutated collection | `notificationreads` (model: `apps/web/models/NotificationRead.ts`) |
| `broadcastToAll` | `apps/web/services/signalr/index.ts:106` |
| Client SDK + connection | `apps/web/utils/context/signalr-context.tsx` |
| Documents-tab listener | `apps/web/app/orders-tracker/[category]/[orderId]/documents/useDocumentsTabNotifications.ts` |
| Orders-list listener | `apps/web/app/orders-tracker/hooks/useOrderNotificationCounts.ts` |
| Refetch endpoint (per-order) | `apps/web/app/api/notifications/by-order/[orderId]/route.ts` |
| Refetch endpoint (counts) | `apps/web/app/api/notifications/counts/route.ts` |
| UI re-render | `page.tsx` (rows) + `apps/web/app/orders-tracker/[category]/[orderId]/layout.tsx` (tab badge) |
