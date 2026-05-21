# MLID-2084 — Row Badge Count Mismatch (HOTFIX, not a re-implementation)

## Task Reference

- **Parent Jira**: [MLID-2084](https://localinfusion.atlassian.net/browse/MLID-2084) — original story "Order Tracker notification badge + Action Required filter" (under epic MLID-2011). The badge feature itself **already shipped to develop** and is in staging. This work is a **follow-up hotfix** triggered by QA Failure #1 reported by Olha Turovych (2026-05-18). No separate Jira ticket — the team tracks this back to the parent ID, which is why the branch name is prefixed with `hotfix/MLID-2084-...` rather than `feature/MLID-XXXX-...`.
- **Story Points (parent)**: 5
- **Branch**: `hotfix/MLID-2084-row-badge-count-mismatch` (already created, off `develop`)
- **Base Branch**: `develop`
- **Status**: Hotfix in development on top of an already-delivered story
- **Fix version**: Release 5/19/2026

> Branching note: this is a **hotfix continuation** of MLID-2084's previously-delivered code, not a fresh implementation of MLID-2084. MLID-2011 (the parent epic) and the original MLID-2084 work are already merged to develop. Do NOT create a new branch — verify we are on `hotfix/MLID-2084-row-badge-count-mismatch` with `git branch --show-current` before any commit.

---

## Summary

The originally-shipped MLID-2084 introduced an unread-notification badge that appears on both the orders-tracker table row (sourced from `POST /api/notifications/counts`) and on the Documents tab inside the order detail (sourced from `GET /api/notifications/by-order/[orderId]`). QA found that the two surfaces report **different counts for the same order** because they apply different read-detection logic: the table keys on `notification.targetUserId`, while the Documents tab keys on the current viewer's session ID. When a notification was created targeting user X but viewed (and read) by a different user Y — common after order reassignment — the table cannot see Y's NotificationRead row and reports the notification as still unread.

This hotfix unifies both surfaces on `getCountForEntity` as the single source of truth for `unreadCount`, adding a `userReadScope` parameter that controls whether the `notificationreads` lookup must match the notification's `targetUserId` (`'target'`, the existing default — preserves all unrelated callers) or any reader (`'any'`). Both route-level callers opt into `'any'`, producing consistent counts across the table and the tab.

---

## Codebase Analysis

### getCountForEntity (notification.ts lines 162–253)

The function builds a Mongoose aggregation pipeline. When `unreadOnly: true`, the pipeline includes a `$facet` with two branches:
- `taskUnread`: matches `type: 'task'` and `isRead: false` directly on the document.
- `userUnread`: matches `type: 'user'`, then does a correlated `$lookup` into `notificationreads` using `let: { notifId: '$_id', targetUserId: '$targetUserId' }` and an inner `$match` that requires both `notificationId == $$notifId` AND `userId == $$targetUserId`.

The `userId == $$targetUserId` clause in the `userUnread` lookup is the only line that needs to change for the `'any'` mode. In `'any'` mode, the lookup drops the `userId` condition entirely — any NotificationRead row for the notification is sufficient.

The `taskUnread` branch is completely unaffected by `userReadScope`.

### POST /api/notifications/counts (route.ts line 79)

The single call to `getCountForEntity` currently passes `{ entityType: 'order', entityId: orderId, unreadOnly: true }`. It needs to add `userReadScope: 'any'`. The existing test at line 187 asserts the exact call shape — that test will need updating in parallel.

### GET /api/notifications/by-order/[orderId] (route.ts)

Currently calls `getNotificationsForEntity` + per-notification `isReadByUser` to build summaries. It returns `{ notifications: NotificationSummary[] }`. It does NOT call `getCountForEntity` at all. The fix adds a second service call — `getCountForEntity({ entityType: 'order', entityId: orderId, unreadOnly: true, userReadScope: 'any' })` — and includes the result as a top-level `unreadCount` in the JSON response. The import block needs `getCountForEntity` added alongside the existing imports.

### useDocumentsTabNotifications.ts (lines 89–92, 127–129)

The hook currently types the API response as `{ notifications: NotificationSummaryResponse[] }` (line 89) and computes `unreadCount` inline as `notifications.filter(n => !n.isReadByCurrentUser).length` (lines 127–129). After this change:
- The response type is widened to include `unreadCount: number`.
- The `useState` that stores notifications remains (`byDocumentId`, `notificationIds`, and per-row indicators still derive from `notifications`).
- A second piece of state `serverUnreadCount` is stored from the API response.
- The `unreadCount` returned from the hook is `serverUnreadCount`, not the client-side filter result.

### Existing tests

- `notification.test.ts` describe block `'getCountForEntity'` (lines 317–517): 8 tests covering `'target'` mode behavior. The test at lines 399–449 specifically asserts that the `$lookup` uses `$$targetUserId` — this test must be updated to assert this is the behavior only when `userReadScope` is `'target'` (or omitted). New tests cover the `'any'` path.
- `counts/route.test.ts` (line 187): the `toHaveBeenCalledWith` assertion matches the current call shape without `userReadScope`. This test fails after the route change and must be updated to include `userReadScope: 'any'`.
- `by-order/[orderId]/route.test.ts`: currently does not mock `getCountForEntity` at all. New tests require adding it to the module mock.
- `useDocumentsTabNotifications.test.tsx` (lines 99–117 and 119–144): two tests mock `fetch` returning only `{ notifications: [...] }` and assert `unreadCount` derived by client-side filter. These must be updated to mock `{ notifications: [...], unreadCount: N }` and assert the hook returns the server-provided value directly.

---

## Implementation Steps

> **Commit strategy:** all work lands in a **single commit** at the very end. TDD discipline still applies — each phase below is a logical red → green block executed sequentially, with `npm run test` validated between phases. Nothing is committed until all four phases are green and the manual browser verification has been run.

### Phase A — service layer: extend `getCountForEntity` with `userReadScope`

**Files**: `apps/web/services/mongodb/notification.ts`, `apps/web/services/mongodb/notification.test.ts`

**RED — add 5 new tests inside the existing `describe('getCountForEntity', ...)` block (after line 516). Do not modify any existing test yet:**
1. `'should default userReadScope to "target" and key the $lookup on targetUserId'` — asserts that calling without `userReadScope` produces a `$lookup` inner pipeline containing `{ $eq: ['$userId', '$$targetUserId'] }`. Re-expresses the existing behavior (already tested at lines 399–449) but makes the default explicit.
2. `'should count a user-type notification as unread when no notificationreads row exists (userReadScope: any)'` — calls with `userReadScope: 'any'`, mocks aggregate resolving to `[{ totalCount: 3 }]`, asserts result is `3`.
3. `'should NOT count a user-type notification as unread when ANY notificationreads row exists regardless of userId (userReadScope: any)'` — calls with `userReadScope: 'any'`, inspects the pipeline: asserts the `$lookup` inner pipeline in `userUnread` does NOT contain a `{ $eq: ['$userId', ...] }` clause — only `{ $eq: ['$notificationId', '$$notifId'] }`.
4. `'should still respect activeExpiryFilter when userReadScope is any'` — calls with `userReadScope: 'any'`, inspects `$match` stage, asserts expiry guard is present (same shape check used in the existing test at line 368).
5. `'should not affect taskUnread branch when userReadScope is any'` — calls with `userReadScope: 'any'`, inspects the `$facet` stage, asserts `taskUnread` contains `{ $match: { type: 'task', isRead: false } }` unchanged.

Run: `npm run test -- --testPathPattern="services/mongodb/notification" --watch=false` — the five new tests should fail because `userReadScope` is not yet a recognised parameter.

**GREEN — modify `getCountForEntity` in `notification.ts`:**

Extend the params type:
```typescript
params: {
  entityType: string;
  entityId: string;
  unreadOnly?: boolean;
  userReadScope?: 'target' | 'any';
}
```
Default: `userReadScope = 'target'`.

In the `userUnread` pipeline branch (lines 194–224), make the `$lookup` conditional on `userReadScope`:
- When `userReadScope === 'target'` (default): keep the existing lookup body — `let` binds `notifId` and `targetUserId`; inner `$match` requires both `notificationId == $$notifId` AND `userId == $$targetUserId`.
- When `userReadScope === 'any'`: build the lookup without a `userId` clause — `let` binds only `notifId`; inner `$match` requires only `notificationId == $$notifId`.

The `taskUnread` facet branch and all other pipeline stages are unchanged. Update the `logger.info` call to include `userReadScope` in the structured log payload.

Run: `npm run test -- --testPathPattern="services/mongodb/notification" --watch=false` — all tests should pass.

---

### Phase B — counts route opts into `'any'`

**Files**: `apps/web/app/api/notifications/counts/route.ts`, `apps/web/app/api/notifications/counts/route.test.ts`

**RED — update tests:**
- Add `'should pass userReadScope: any to getCountForEntity'`:
  ```typescript
  expect(mockGetCountForEntity).toHaveBeenCalledWith({
    entityType: 'order',
    entityId: 'order-1',
    unreadOnly: true,
    userReadScope: 'any',
  });
  ```
- Update the existing `'should call getCountForEntity without passing viewer identity'` (line 178) to include `userReadScope: 'any'` in its `toHaveBeenCalledWith` assertion.

Run: tests should fail.

**GREEN — at line 79 of `counts/route.ts`, add `userReadScope: 'any'`:**
```typescript
const count = await getCountForEntity({
  entityType: 'order',
  entityId: orderId,
  unreadOnly: true,
  userReadScope: 'any',
});
```

Run: all tests pass. At this point the table badge for the example order already drops from 6 → 4 (intentional — see Manual Verification).

---

### Phase C — by-order route exposes server-side `unreadCount`

**Files**: `apps/web/app/api/notifications/by-order/[orderId]/route.ts`, `apps/web/app/api/notifications/by-order/[orderId]/route.test.ts`

**RED — update test file:**
1. Add `getCountForEntity` to the existing module mock at the top:
   ```typescript
   jest.mock('@/services/mongodb/notification', () => ({
     getNotificationsForEntity: jest.fn(),
     isReadByUser: jest.fn(),
     getCountForEntity: jest.fn(),
   }));
   ```
2. Add `getCountForEntity` to the import block and create a typed mock variable alongside the existing ones.
3. In `beforeEach`, add: `mockGetCountForEntity.mockResolvedValue(0);`
4. Add three new tests:
   - `'should include unreadCount in the response'` — mocks `getCountForEntity` resolving to `4`, mocks `getNotificationsForEntity` resolving to `[]`, calls `GET`, asserts `data.unreadCount === 4`.
   - `'should call getCountForEntity with entityType order, entityId, unreadOnly true, and userReadScope any'` — calls `GET` with orderId `'order-xyz'`, asserts `mockGetCountForEntity` was called with `{ entityType: 'order', entityId: 'order-xyz', unreadOnly: true, userReadScope: 'any' }`.
   - `'should return unreadCount 0 when getCountForEntity returns 0'` — mocks `getCountForEntity` returning `0`, asserts `data.unreadCount === 0`.

Run: the three new tests should fail.

**GREEN — modify `by-order/[orderId]/route.ts`:**
1. Add `getCountForEntity` to the import from `@/services/mongodb/notification`.
2. Inside the `try` block, after the existing `getNotificationsForEntity` + per-notification read summaries are built, add:
   ```typescript
   const unreadCount = await getCountForEntity({
     entityType: 'order',
     entityId: orderId,
     unreadOnly: true,
     userReadScope: 'any',
   });
   ```
3. Update the `NextResponse.json` call:
   ```typescript
   return NextResponse.json({ notifications: summaries, unreadCount }, { status: 200 });
   ```

The per-notification `isReadByUser` calls, the `toNotificationSummary` helper, and the auth/feature-flag guards are all unchanged.

Run: all tests pass.

---

### Phase D — hook reads `unreadCount` from the server

**Files**: `apps/web/app/orders-tracker/[category]/[orderId]/documents/useDocumentsTabNotifications.ts`, `apps/web/app/orders-tracker/[category]/[orderId]/documents/useDocumentsTabNotifications.test.tsx`

**RED — update the existing hook tests:**

Test 1 — `'returns correct unreadCount based on isReadByCurrentUser values'` (line 99):
- Update fetch mock to return `{ notifications: [...], unreadCount: 2 }`.
- Keep the assertion `result.current.unreadCount === 2`. Intent preserved; mechanism changes from "count unread items" to "trust server value".

Test 2 — `'unreadCount counts every unread notification regardless of orderAssignedTo (global tab badge)'` (line 119):
- Update fetch mock to return `{ notifications: [...], unreadCount: 2 }`.
- Keep the assertion. The "global tab badge" intent is now enforced server-side via `getCountForEntity` with `userReadScope: 'any'`.

All other tests in this file must have their `fetch` mocks updated to include `unreadCount` in the response shape (`unreadCount: 0` is fine for tests that do not assert on it) — required because the response type assertion will now expect the field.

Add one new test:
- `'reads unreadCount from the server response rather than computing it client-side'` — mocks `fetch` returning `{ notifications: [makeNotification({ isReadByCurrentUser: false })], unreadCount: 99 }`. Asserts `result.current.unreadCount === 99` (server overrides what the client-side filter of one unread item would yield).

Run: the new test should fail; the two updated existing tests will fail until the hook is rewired.

**GREEN — modify `useDocumentsTabNotifications.ts`:**

1. Add a new local type for the API response:
   ```typescript
   interface ByOrderApiResponse {
     notifications: NotificationSummaryResponse[];
     unreadCount: number;
   }
   ```
2. Update the type assertion on `response.json()` (line 89):
   ```typescript
   const data = (await response.json()) as ByOrderApiResponse;
   ```
3. Add state to hold the server-provided count:
   ```typescript
   const [serverUnreadCount, setServerUnreadCount] = useState(0);
   ```
4. In the `fetchNotifications` callback, after `setNotifications(data.notifications)`:
   ```typescript
   setServerUnreadCount(data.unreadCount);
   ```
5. Delete the inline `unreadCount` computation (lines 127–129).
6. Update the return value:
   ```typescript
   return {
     unreadCount: serverUnreadCount,
     byDocumentId,
     notificationIds,
     unreadNotificationIdsForCurrentUser,
     hasUnreadForCurrentUser,
     isLoading,
   };
   ```

`byDocumentId`, `notificationIds`, `unreadNotificationIdsForCurrentUser`, and `hasUnreadForCurrentUser` continue to derive from the `notifications` state (per-viewer per-row UI, unchanged). Only `unreadCount` is now sourced from the server.

Run: all tests pass.

---

### Final gates before commit

Run the full scoped suite plus type/lint/format:
```
npm run test -- --testPathPattern="services/mongodb/notification|notifications/counts/route|by-order/.*/route|useDocumentsTabNotifications" --watch=false
npm run types:check
npm run lint:fix
npm run format
```

All must pass green.

### Manual browser verification (required before commit)

Per the user's `feedback_visual_test_before_commit` preference — do the steps in the Manual Verification section under "Testing Strategy" below, then wait for explicit user sign-off before committing.

### The single commit

```
[MLID-2084] - fix(notifications): unify order document badge counts across surfaces

Both the orders-tracker table badge (POST /api/notifications/counts) and
the Documents tab badge (GET /api/notifications/by-order/[orderId]) now
use getCountForEntity as the single source of truth, resolving the
mismatch where the same order showed different counts on each surface.

Service layer:
- Add userReadScope: 'target' | 'any' to getCountForEntity, default
  'target' (preserves existing behavior for all unrelated callers).
- When 'any', the userUnread $lookup drops the userId == targetUserId
  clause and matches only on notificationId — any reader's NotificationRead
  row marks the notification as acknowledged for the team. This is required
  to handle the order-reassignment scenario (assignee X → Y) where Y's
  read should drop the badge for everyone.
- The taskUnread branch and all expiry filtering are unchanged.

Counts route (table badge):
- POST /api/notifications/counts now passes userReadScope: 'any'.

By-order route (Documents tab):
- GET /api/notifications/by-order/[orderId] now calls getCountForEntity
  with userReadScope: 'any' and includes the result as a top-level
  unreadCount in the response, alongside the existing per-notification
  summaries.

Documents tab hook:
- useDocumentsTabNotifications now reads unreadCount from the API
  response instead of computing it client-side from isReadByCurrentUser.
- Per-row indicators (byDocumentId, notificationIds, hasUnreadForCurrentUser,
  unreadNotificationIdsForCurrentUser) continue to derive from per-viewer
  isReadByCurrentUser values and are unchanged.

Both badges now report the same number for the same order, regardless of
which user is viewing.
```

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/services/mongodb/notification.ts` | Modify | Add `userReadScope: 'target' \| 'any'` parameter; conditional `$lookup` body in `userUnread` facet branch |
| `apps/web/services/mongodb/notification.test.ts` | Modify | Add 5 new tests for `userReadScope: 'any'` behavior |
| `apps/web/app/api/notifications/counts/route.ts` | Modify | Pass `userReadScope: 'any'` to `getCountForEntity` |
| `apps/web/app/api/notifications/counts/route.test.ts` | Modify | Add 1 new test; update 1 existing assertion to include `userReadScope: 'any'` |
| `apps/web/app/api/notifications/by-order/[orderId]/route.ts` | Modify | Import + call `getCountForEntity`; add `unreadCount` to JSON response |
| `apps/web/app/api/notifications/by-order/[orderId]/route.test.ts` | Modify | Add `getCountForEntity` to mock; add 3 new tests |
| `apps/web/app/orders-tracker/[category]/[orderId]/documents/useDocumentsTabNotifications.ts` | Modify | Add `ByOrderApiResponse` type; store `serverUnreadCount` state; remove client-side filter computation |
| `apps/web/app/orders-tracker/[category]/[orderId]/documents/useDocumentsTabNotifications.test.tsx` | Modify | Update all `fetch` mocks to include `unreadCount`; update 2 existing tests; add 1 new test |

---

## Testing Strategy

### Unit tests — notification.ts

- Behavior: `userReadScope` defaults to `'target'` — `$lookup` inner pipeline includes `{ $eq: ['$userId', '$$targetUserId'] }`.
- Behavior: `userReadScope: 'any'` — `$lookup` inner pipeline does NOT include a `userId` equality clause; only `notificationId` is matched.
- Behavior: `userReadScope: 'any'` — `activeExpiryFilter` is still applied in `$match`.
- Behavior: `userReadScope: 'any'` — `taskUnread` facet branch is structurally identical to `'target'` mode.
- Behavior: `userReadScope: 'any'` — aggregate result is returned as-is.

Mocked boundary: `Notification.aggregate` (already mocked in the file via `jest.mock`).

### Unit tests — counts route

- Behavior: `getCountForEntity` is called with `userReadScope: 'any'` for every orderId in the batch.
- All existing behavior tests (401, 403, 400, 500, 200 with counts) remain intact.

Mocked boundary: `getCountForEntity` (already mocked).

### Unit tests — by-order route

- Behavior: `getCountForEntity` is called with `{ entityType: 'order', entityId, unreadOnly: true, userReadScope: 'any' }`.
- Behavior: response includes `unreadCount` equal to the value returned by `getCountForEntity`.
- Behavior: `unreadCount` is `0` when `getCountForEntity` returns `0`.
- All existing behavior tests (401, 404, 400, 200 with notifications, 500) remain intact.

Mocked boundary: `getCountForEntity` and `getNotificationsForEntity` (both mocked).

### Unit tests — useDocumentsTabNotifications hook

- Behavior: `unreadCount` in hook return equals the `unreadCount` field from the API response, NOT the count of `isReadByCurrentUser: false` items in the notifications array.
- Behavior: when `fetch` returns `unreadCount: 99` with one unread notification, hook returns `unreadCount: 99`.
- Behavior: `byDocumentId`, `notificationIds`, `unreadNotificationIdsForCurrentUser`, and `hasUnreadForCurrentUser` are still derived from per-notification `isReadByCurrentUser` (per-viewer, unchanged).
- All existing SignalR refetch, loading state, and zeroed-result tests remain intact.

Mocked boundary: `global.fetch` (already mocked).

### Manual verification

Before opening the PR, verify in a running dev environment:

1. Log in as a user who is NOT the original assignee of an order that has unread document notifications.
2. Note the orders-tracker table badge count for that order.
3. Open the order detail and navigate to the Documents tab.
4. Confirm the Documents tab badge matches the table badge exactly.
5. Click a document to mark it read (triggering a `notification:read` SignalR event).
6. Confirm BOTH badges decrement by 1 after the refetch.
7. Log out and log in as the original assignee.
8. Confirm the same counts are visible (the badge is global, not per-viewer).

Expected behavior change (intentional, NOT a regression): for the confirmed repro order `6a078a236de202e6c06d1037`, the table badge will drop from 6 to 4 after this fix because 2 NotificationRead rows already exist. This is correct — those 2 notifications were acknowledged by a team member after reassignment, and the badge should reflect that globally.

---

## Security Considerations

- **Input validation**: no new inputs added to any route. The `userReadScope` parameter is internal to the service layer and never accepted from HTTP request bodies.
- **Authorization**: no change to auth/feature-flag guards. Both routes are still gated by session + `ORDER_DOCUMENT_NOTIFICATIONS` feature flag. The `userReadScope: 'any'` option does not weaken access control — it only changes which NotificationRead rows count as acknowledgements, not who can query the data.
- **PHI handling**: notification documents do not contain PHI. The change does not touch patient data fields, `getNotificationsForEntity`, or any PHI-masking path.

---

## Open Questions

None. All design decisions have been confirmed by the PO (Parker Baum, 2026-05-18 09:59) and the engineer. The per-order acknowledgement model is intentional and compatible with the `userReadScope: 'any'` approach.
