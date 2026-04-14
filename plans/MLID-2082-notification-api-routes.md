# MLID-2082 — Notification API Routes (list, mark read)

## Task Reference

- **Jira**: [MLID-2082](https://localinfusion.atlassian.net/browse/MLID-2082)
- **Story Points**: 3
- **Branch**: `feature/MLID-2082-notification-api-routes`
- **Base Branch**: `epic/MLID-2011-order-document-notifications`
- **Epic Plan**: `docs/agomez/plans/MLID-2011-plan-progress.md`
- **Status**: Done

---

## Summary

Creates three API routes for the notifications feature: a paginated GET to list notifications for the current user, a PATCH to mark a single notification as read, and a PATCH to bulk-mark multiple notifications as read. All routes delegate to the already-built notification service and broadcast SignalR events after mutations, gated by the `ORDER_DOCUMENT_NOTIFICATIONS` feature flag.

---

## Codebase Analysis

- **Notification service** lives at `apps/web/services/mongodb/notification.ts` and exposes all required functions: `getNotificationsForUser`, `getUnreadCountForUser`, `markUserNotificationAsRead`, `bulkMarkUserNotificationsAsRead`. No new service logic is needed.
- **SignalR** is accessed via `getSignalRService()` from `apps/web/services/signalr/index.ts`. After mutations, broadcast `notification:read` with `{ notificationId, userId }` (single) or `{ notificationIds, userId }` (bulk).
- **Auth pattern**: `getServerSession(authOptions)` → check `session.user` → check `session.user.userObj`. Extract `userId = session.user.id`, `locationIds = userObj.locations ?? []`, `role = userObj.role`.
- **Feature flag pattern**: `FeatureFlagService.getFeatureFlag(FeatureFlag.ORDER_DOCUMENT_NOTIFICATIONS)` — return 404 if disabled.
- **Error handling**: `getErrorResponse(error, publicMessage)` from `@/utils/helpers/errorHandling` returns a `NextResponse` with status 500.
- **GET response shape**: `{ data: INotification[], unreadCount: number }` — no totalCount (service does not return it). Route accepts `limit` and `offset` query params; defaults are 50 and 0 respectively (matching service defaults).
- **Route directory**: `apps/web/app/api/notifications/` — does not yet exist. Static `bulk-read/` segment takes precedence over dynamic `[id]/` in App Router, so there is no naming conflict.
- **Test pattern**: `@jest-environment node`, mock `next-auth`, `connect`, `notification`, `featureFlagService`, `signalr`, and `logger`. Construct requests with `NextRequest`. Session is shaped with a `createMockSession` factory.

---

## Implementation Steps

### Step 1 — GET /api/notifications (list with unread count)

- **Files**:
  - `apps/web/app/api/notifications/route.test.ts` (RED — create test)
  - `apps/web/app/api/notifications/route.ts` (GREEN — implement)

- **What**:

  **RED — write the test first.** Cover the following behaviors:

  | Test case | Expected |
  |-----------|----------|
  | No session | 401 `{ error: 'Unauthorized' }` |
  | Session has no `userObj` | 401 `{ error: 'Unauthorized' }` |
  | Feature flag disabled | 404 `{ error: 'Feature not available' }` |
  | Happy path, default params | 200 `{ data: INotification[], unreadCount: number }` |
  | Happy path, custom `limit` and `offset` params | calls service with parsed values |
  | `limit` exceeds 100 | clamps to 100 |
  | Service throws | 500 via `getErrorResponse` |

  Mock `getNotificationsForUser` to return `[mockNotification]` and `getUnreadCountForUser` to return `3`.

  **GREEN — implement the route.** Pipeline:
  1. `getServerSession(authOptions)` — return 401 if no session or no `userObj`.
  2. `FeatureFlagService.getFeatureFlag(FeatureFlag.ORDER_DOCUMENT_NOTIFICATIONS)` — return 404 if disabled.
  3. Parse `limit` and `offset` from `request.nextUrl.searchParams`; default to 50 / 0; clamp `limit` to max 100.
  4. Call `getNotificationsForUser({ userId, locationIds, role, limit, offset })`.
  5. Call `getUnreadCountForUser({ userId, locationIds, role })`.
  6. Return `NextResponse.json({ data, unreadCount }, { status: 200 })`.
  7. Wrap steps 3–6 in try/catch → `getErrorResponse(error, 'Failed to fetch notifications')`.

  **Exported handler name**: `GET`.

---

### Step 2 — PATCH /api/notifications/:id/read (mark single read)

- **Files**:
  - `apps/web/app/api/notifications/[id]/read/route.test.ts` (RED — create test)
  - `apps/web/app/api/notifications/[id]/read/route.ts` (GREEN — implement)

- **What**:

  **RED — write the test first.** Cover:

  | Test case | Expected |
  |-----------|----------|
  | No session | 401 |
  | Session has no `userObj` | 401 |
  | Feature flag disabled | 404 |
  | Happy path — valid `id` param | calls `markUserNotificationAsRead(id, userId)`, broadcasts SignalR, returns 200 |
  | SignalR broadcast receives correct payload | `{ notificationId: id, userId }` |
  | Service throws (non-duplicate) | 500 via `getErrorResponse` |

  Mock `markUserNotificationAsRead` as resolved void. Mock `getSignalRService` to return `{ broadcastToAll: jest.fn() }`.

  **GREEN — implement the route.** Pipeline:
  1. `getServerSession(authOptions)` — return 401 if no session or no `userObj`.
  2. `FeatureFlagService.getFeatureFlag(FeatureFlag.ORDER_DOCUMENT_NOTIFICATIONS)` — return 404 if disabled.
  3. Extract `id` from the route segment params (`context.params.id`).
  4. Call `markUserNotificationAsRead(id, userId)`.
  5. Broadcast: `(await getSignalRService()).broadcastToAll('notification:read', { notificationId: id, userId })`.
  6. Return `NextResponse.json({ success: true }, { status: 200 })`.
  7. Wrap steps 3–6 in try/catch → `getErrorResponse(error, 'Failed to mark notification as read')`.

  **Exported handler name**: `PATCH`.

  **Params type**: `{ params: { id: string } }` passed as second argument to `PATCH(request, context)`.

---

### Step 3 — PATCH /api/notifications/bulk-read (mark multiple read)

- **Files**:
  - `apps/web/app/api/notifications/bulk-read/route.test.ts` (RED — create test)
  - `apps/web/app/api/notifications/bulk-read/route.ts` (GREEN — implement)

- **What**:

  **RED — write the test first.** Cover:

  | Test case | Expected |
  |-----------|----------|
  | No session | 401 |
  | Session has no `userObj` | 401 |
  | Feature flag disabled | 404 |
  | Body missing `notificationIds` | 400 `{ error: 'notificationIds must be a non-empty array of strings' }` |
  | `notificationIds` is an empty array | 400 (same message) |
  | `notificationIds` contains a non-string element | 400 (same message) |
  | `notificationIds` has more than 100 elements | 400 `{ error: 'notificationIds exceeds maximum of 100' }` |
  | Happy path — valid array | calls `bulkMarkUserNotificationsAsRead(ids, userId)`, broadcasts SignalR, returns 200 |
  | SignalR broadcast receives correct payload | `{ notificationIds: ids, userId }` |
  | Service throws | 500 via `getErrorResponse` |

  Mock `bulkMarkUserNotificationsAsRead` as resolved void. Mock `getSignalRService` as in Step 2.

  **GREEN — implement the route.** Pipeline:
  1. `getServerSession(authOptions)` — return 401 if no session or no `userObj`.
  2. `FeatureFlagService.getFeatureFlag(FeatureFlag.ORDER_DOCUMENT_NOTIFICATIONS)` — return 404 if disabled.
  3. Parse `request.json()` body; extract `notificationIds`.
  4. Validate:
     - `notificationIds` must be a non-empty `string[]`.
     - Every element must be of type `string`.
     - Length must not exceed 100.
     - Return 400 with appropriate message on any failure.
  5. Call `bulkMarkUserNotificationsAsRead(notificationIds, userId)`.
  6. Broadcast: `(await getSignalRService()).broadcastToAll('notification:read', { notificationIds, userId })`.
  7. Return `NextResponse.json({ success: true }, { status: 200 })`.
  8. Wrap steps 3–7 in try/catch → `getErrorResponse(error, 'Failed to mark notifications as read')`.

  **Exported handler name**: `PATCH`.

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/app/api/notifications/route.test.ts` | Create | Tests for GET /api/notifications |
| `apps/web/app/api/notifications/route.ts` | Create | GET handler — list notifications + unread count |
| `apps/web/app/api/notifications/[id]/read/route.test.ts` | Create | Tests for PATCH /api/notifications/:id/read |
| `apps/web/app/api/notifications/[id]/read/route.ts` | Create | PATCH handler — mark single notification read |
| `apps/web/app/api/notifications/bulk-read/route.test.ts` | Create | Tests for PATCH /api/notifications/bulk-read |
| `apps/web/app/api/notifications/bulk-read/route.ts` | Create | PATCH handler — bulk mark notifications read |

---

## Testing Strategy

- **Unit tests**: All behavior is covered via unit tests co-located with each route file. Every route has tests for: unauthenticated request (401), missing `userObj` (401), feature flag disabled (404), input validation failures (400), happy path (200), and service/broadcast errors (500).
- **Integration tests**: None — all external boundaries (MongoDB, SignalR) are mocked at the module level. No in-process database connection is used by these tests.
- **Manual verification**:
  1. Enable `ORDER_DOCUMENT_NOTIFICATIONS` flag in `/admin/feature-flags`.
  2. `GET /api/notifications` — confirm response shape `{ data: [...], unreadCount: N }` with a real session.
  3. `PATCH /api/notifications/:id/read` — confirm SignalR event appears in browser console via Socket.IO listener.
  4. `PATCH /api/notifications/bulk-read` — send `{ notificationIds: ['id1', 'id2'] }`, confirm 200 and SignalR broadcast.
  5. Repeat all three with flag disabled — confirm 404.

**Specific test cases per route:**

GET route:
- `should return 401 when there is no session`
- `should return 401 when session has no userObj`
- `should return 404 when feature flag is disabled`
- `should return 200 with data and unreadCount`
- `should pass parsed limit and offset to getNotificationsForUser`
- `should clamp limit to 100 when it exceeds the maximum`
- `should return 500 when getNotificationsForUser throws`

PATCH single read:
- `should return 401 when there is no session`
- `should return 401 when session has no userObj`
- `should return 404 when feature flag is disabled`
- `should call markUserNotificationAsRead with id and userId`
- `should broadcast notification:read event with notificationId and userId`
- `should return 200 with success true`
- `should return 500 when service throws`

PATCH bulk read:
- `should return 401 when there is no session`
- `should return 401 when session has no userObj`
- `should return 404 when feature flag is disabled`
- `should return 400 when notificationIds is missing`
- `should return 400 when notificationIds is an empty array`
- `should return 400 when notificationIds contains a non-string element`
- `should return 400 when notificationIds exceeds 100 elements`
- `should call bulkMarkUserNotificationsAsRead with ids and userId`
- `should broadcast notification:read event with notificationIds and userId`
- `should return 200 with success true`
- `should return 500 when service throws`

---

## Security Considerations

- **Input validation**: The GET route clamps `limit` to a max of 100 to prevent oversized queries. The bulk-read route validates `notificationIds` is a non-empty `string[]` with no more than 100 elements, rejecting any non-string values. All validation occurs at the API boundary before any service call.
- **Authorization**: All three routes require a valid NextAuth session with a `userObj`. The feature flag check is performed after auth, following the standard pipeline: auth → feature flag → validate → execute → respond. There is no role restriction beyond authentication — any authenticated user may read their own notifications.
- **PHI handling**: Notifications do not contain PHI fields directly. The notification service filters by `userId` and `locationIds`, ensuring a user cannot read another user's notifications. No additional masking is required.
- **No over-exposure**: The GET route scopes results to the current user via `userId` and `locationIds` extracted from the server-side session — the client cannot supply these values.

---

## Open Questions

None.
