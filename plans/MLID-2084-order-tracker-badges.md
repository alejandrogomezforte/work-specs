# MLID-2084 — Order Tracker notification badge (Action Required filter deferred)

## Task Reference

- **Jira**: [MLID-2084](https://localinfusion.atlassian.net/browse/MLID-2084)
- **Story Points**: 5
- **Branch**: `feature/MLID-2084-order-tracker-badges`
- **Base Branch**: `epic/MLID-2011-order-document-notifications`
- **Status**: To Do
- **Epic Plan**: `docs/agomez/plans/MLID-2011-plan-progress.md`

---

## Summary

This task adds per-order unread notification badges to the Order ID column in the Order Tracker table and a toggleable "Action Required" quick filter chip that narrows the view to orders with unread notifications. Both features are gated by the `ORDER_DOCUMENT_NOTIFICATIONS` feature flag and update in real time via the existing SignalR `documentAdded` and `notification:read` events established in D1 and D2.

---

## Codebase Analysis

- **`getNewOrdersColumns()` in `NewOrders.tsx`** — pure function returning `GridColDef<FullOrder<'new'>>[]`. The `displayId` column's `renderCell` wraps content in `<Layout direction="row" gap={2} align="center">` with a `<Link>`. The badge will be inserted adjacent to the Link inside that Layout.
- **`page.tsx` (`[category]`)** — calls `getNewOrdersColumns()` inside a `useMemo` and renders `<Orders>` with the result. This is where the hook will be called and column props injected. The `quickFilters` array follows a `{ id, label, display, apply }` shape rendered as `<Chip>` components. Active filter is tracked via `activeQuickFilter` from `useGridUrlModels`.
- **`NewOrders.test.tsx`** — tests `getNewOrdersColumns` by calling it directly and rendering cells with `renderWithProviders` + `LiThemeProvider`. New badge tests follow this same pattern.
- **`page.test.tsx`** — mocks `useSession`, `useFeatureFlag`, `useOrdersTracker`, `useGridUrlModels`. New SignalR hook will also need to be mocked here.
- **Notification service** — `getCountForEntity()` accepts `{ entityType, entityId, userId, locationIds, role, unreadOnly }` and runs a per-entity MongoDB aggregation. A batch endpoint will call it once per visible order ID.
- **Existing notification API routes** — `GET /api/notifications`, `PATCH /api/notifications/:id/read`, `PATCH /api/notifications/bulk-read`. Pattern: auth check → feature flag → validate → execute → respond. New `POST /api/notifications/counts` follows this exact shape. Test file pattern: `route.test.ts` co-located in same folder, `@jest-environment node`, all Mongoose mocked at top.
- **SignalR context** — `useSignalR()` returns `{ connection: HubConnection | null, isConnected: boolean }`. Subscribe with `connection.on(event, handler)`, cleanup with `connection.off(event, handler)`, inside a `useEffect` gated on `connection && isConnected`.
- **Feature flag client pattern** — `useFeatureFlag(FeatureFlag.ORDER_DOCUMENT_NOTIFICATIONS)` returns boolean.
- **"Action Required" client-side filter** — since unread counts are not stored on order documents, filtering is done client-side by hiding rows whose `_id` is absent from the counts map. The DataGrid already accepts `filterModel` to filter visible rows; however, because this is a derived client-side concept (not a server-understood field), the filter must be applied as a row-level `filterModel` item using a custom operator OR by filtering the `orders` array passed as props before the grid renders. The cleanest approach without custom DataGrid operators is to hold a local `actionRequiredActive` boolean and, when true, derive a filtered row set from `unreadCountsByOrderId`.

---

## Implementation Steps

### Step 1 — RED: batch notification counts API route test

- **Files**:

  - `apps/web/app/api/notifications/counts/route.test.ts` (Create)

- **What**: Write the full test suite for `POST /api/notifications/counts` before any implementation exists. Tests cover: 401 when no session, 401 when no userObj, 404 when feature flag is disabled, 400 when body is missing `orderIds`, 400 when `orderIds` exceeds 100, 200 with a `Record<string, number>` response shape, 500 on service error. Use the same mock setup pattern as `apps/web/app/api/notifications/route.test.ts`: `@jest-environment node`, jest.mock calls at the top before imports, mock `getCountForEntity` from `@/services/mongodb/notification`. This step produces failing tests only — no implementation yet.

### Step 2 — GREEN + REFACTOR: batch notification counts API route implementation

- **Files**:

  - `apps/web/app/api/notifications/counts/route.ts` (Create)

- **What**: Implement `POST /api/notifications/counts`. The handler follows the standard pipeline: `getServerSession` → check `session.user` and `userObj` → `FeatureFlagService.getFeatureFlag(FeatureFlag.ORDER_DOCUMENT_NOTIFICATIONS)` → parse and validate body (`orderIds: string[]`, max 100 items) → call `getCountForEntity` once per orderId in `Promise.all` with `unreadOnly: true` → return `Record<string, number>`. Exact response shape:

  ```typescript
  // 200
  {
    counts: Record<string, number>;
  }

  // Error shapes follow existing convention
  {
    error: string;
  }
  ```

  Type for the request body:

  ```typescript
  type CountsRequestBody = {
    orderIds: string[];
  };
  ```

  The `getCountForEntity` call for each order:

  ```typescript
  getCountForEntity({
    entityType: 'order',
    entityId: orderId,
    userId: session.user.id,
    locationIds: userObj.locations ?? [],
    role: userObj.role,
    unreadOnly: true,
  });
  ```

  Run tests after implementation — all Step 1 tests must pass before moving to Step 3.

### Step 3 — RED: `useOrderNotificationCounts` hook test

- **Files**:

  - `apps/web/app/orders-tracker/hooks/useOrderNotificationCounts.test.ts` (Create)

- **What**: Write tests for the custom hook before implementing it. Use `renderHook` from `@testing-library/react`. Mock `useFeatureFlag`, `useSession`, `useSignalR`, and `global.fetch`. Test cases:

  - Returns empty `{}` when feature flag is disabled
  - Returns empty `{}` when no session
  - Calls `POST /api/notifications/counts` with the provided `orderIds` on mount
  - Returns the `counts` record from the API response keyed by orderId
  - Re-fetches when `orderIds` changes (stable reference comparison)
  - Subscribes to `documentAdded` SignalR event and re-fetches on receipt
  - Subscribes to `notification:read` SignalR event and re-fetches on receipt
  - Cleans up SignalR subscriptions on unmount
  - Returns `{}` when the API call fails (does not throw)

  This step produces failing tests only.

### Step 4 — GREEN + REFACTOR: `useOrderNotificationCounts` hook implementation

- **Files**:

  - `apps/web/app/orders-tracker/hooks/useOrderNotificationCounts.ts` (Create)

- **What**: Implement the hook. Signature:

  ```typescript
  export function useOrderNotificationCounts(
    orderIds: string[]
  ): Record<string, number>;
  ```

  Internal state: `counts: Record<string, number>`, initialized to `{}`. Logic:

  1. Read `isEnabled = useFeatureFlag(FeatureFlag.ORDER_DOCUMENT_NOTIFICATIONS)` and `session = useSession().data`.
  2. Read `{ connection, isConnected } = useSignalR()`.
  3. Fetch function (wrapped in `useCallback`): posts to `/api/notifications/counts` with current `orderIds`. On success sets `counts`; on failure logs with `logger.warn` and leaves counts unchanged (no throw to avoid crashing the table).
  4. `useEffect` triggered on `[isEnabled, userId, serializedOrderIds]` — where `serializedOrderIds` is `JSON.stringify(orderIds)` to avoid reference churn — calls the fetch function.
  5. SignalR `useEffect` triggered on `[connection, isConnected, fetch]`: registers `connection.on('documentAdded', fetch)` and `connection.on('notification:read', fetch)`, returns cleanup that calls `connection.off` for both.
  6. Early return `{}` if `!isEnabled || !session?.user.id`.

  Run Step 3 tests to confirm green before continuing.

### Step 5 — RED: badge in `displayId` column test

- **Files**:

  - `apps/web/app/orders-tracker/Orders/NewOrders.test.tsx` (Modify)

- **What**: Add a new `describe('notification badge')` block inside the existing `describe('Order ID column')` block. Tests:

  - Badge is not rendered when `unreadCountsByOrderId` is not provided (backward compatibility)
  - Badge is not rendered when `unreadCountsByOrderId[orderId]` is 0 or absent
  - Badge is not rendered when `order.assignedTo !== currentUserId`
  - Badge is rendered with the correct count when `unreadCountsByOrderId[orderId] > 0` and `order.assignedTo === currentUserId`
  - Badge renders with `data-testid="notification-badge-{orderId}"`
  - Clicking the badge icon navigates to the documents tab

  These tests will fail because `getNewOrdersColumns` does not yet accept `unreadCountsByOrderId` or `currentUserId`.

### Step 6 — GREEN + REFACTOR: badge in `displayId` column implementation

- **Files**:

  - `apps/web/app/orders-tracker/Orders/NewOrders.tsx` (Modify)

- **What**: Extend the `getNewOrdersColumns` parameter object with two optional fields:

  ```typescript
  unreadCountsByOrderId?: Record<string, number>;
  currentUserId?: string;
  ```

  In the `displayId` column's `renderCell`, inside the existing `<Layout direction="row" gap={2} align="center">`, after the `<Link>` element, conditionally render a document icon with count badge. The badge is shown when all three conditions are true: `unreadCountsByOrderId` is provided, `unreadCountsByOrderId[params.row._id] > 0`, and `params.row.assignedTo === currentUserId`.

  **Badge markup** (adapted from UI designer's prototype at `li-ui-playground`):

  ```tsx
  import DescriptionOutlinedIcon from '@mui/icons-material/DescriptionOutlined';
  import { colors } from '@repo/ui/tokens';
  import Link from 'next/link';

  // Inside renderCell, after the Order ID <Link>:
  {
    unreadCount > 0 && (
      <Link
        href={`/orders-tracker/new/${params.row._id}/documents`}
        data-testid={`notification-badge-${params.row._id}`}
        style={{
          position: 'relative',
          display: 'inline-flex',
          cursor: 'pointer',
        }}
      >
        <DescriptionOutlinedIcon
          style={{ fontSize: 18, color: colors.interactive.primary }}
        />
        <span
          style={{
            position: 'absolute',
            top: -4,
            right: -6,
            minWidth: 14,
            height: 14,
            borderRadius: 7,
            backgroundColor: colors.interactive.primary,
            color: '#fff',
            fontSize: 9,
            fontWeight: 700,
            display: 'flex',
            alignItems: 'center',
            justifyContent: 'center',
            padding: '0 3px',
          }}
        >
          {unreadCount}
        </span>
      </Link>
    );
  }
  ```

  Where `unreadCount = unreadCountsByOrderId?.[params.row._id] ?? 0`.

  **Design notes:**

  - `DescriptionOutlinedIcon` = document icon from `@mui/icons-material` (icons are always from MUI directly, not `@repo/ui`)
  - `colors.interactive.primary` (`#134649`) = project's primary brand color from design tokens
  - Badge is a custom absolute-positioned `<span>` overlay on the icon, not MUI `<Badge>` — matches the designer's exact visual
  - Clicking the icon navigates to the order's documents tab
  - Badge sits inside the existing `<Layout direction="row" gap={2}>` after the Order ID link, not wrapping it

  Run Step 5 tests to confirm green.

### Step 7 — RED: "Action Required" filter and page wiring tests

- **Files**:

  - `apps/web/app/orders-tracker/[category]/page.test.tsx` (Modify)

- **What**: Add a new `describe('ORDER_DOCUMENT_NOTIFICATIONS features')` block. Mock `useOrderNotificationCounts` at the top of the test file alongside the other mocks. Test cases:

  - "Action Required" chip is not present when feature flag is disabled
  - "Action Required" chip is present when feature flag is enabled
  - Clicking "Action Required" chip when inactive marks it active (chip variant becomes `soft`)
  - Clicking "Action Required" chip when already active deactivates it (back to `outlined`)
  - When "Action Required" is active and a row has a count > 0, it appears in the grid
  - When "Action Required" is active and a row has count 0, it is filtered from the grid
  - `useOrderNotificationCounts` is called with the IDs of the currently visible orders

  These tests fail because the page does not yet call the hook or add the filter.

### Step 8 — GREEN + REFACTOR: wire hook and filter into `page.tsx`

- **Files**:

  - `apps/web/app/orders-tracker/[category]/page.tsx` (Modify)

- **What**: Four changes to `page.tsx`:

  1. **Call the hook** — derive `visibleOrderIds` from the currently displayed orders (`newOrders.data`, `maintenanceOrders.data`, or `leadOrders.data` depending on `tab`). Call `useOrderNotificationCounts(visibleOrderIds)` gated by `isNotificationsEnabled`. Store the result as `unreadCountsByOrderId`.

  2. **Pass badge props to `getNewOrdersColumns`** — add `unreadCountsByOrderId` and `currentUserId: session?.user.id` to the `getNewOrdersColumns({...})` call inside `useMemo`. Also add `unreadCountsByOrderId` and `session?.user.id` to the `useMemo` dependency array.

  3. **Local filter state** — add `const [actionRequiredActive, setActionRequiredActive] = useState(false)`. When the tab changes, reset `actionRequiredActive` to false.

  4. **"Action Required" quick filter** — add a new entry to the `quickFilters` array:

     ```typescript
     {
       id: 'actionRequired',
       label: 'Action Required',
       display: isNotificationsEnabled && tab === 'new',
       apply: () => setActionRequiredActive(true),
     }
     ```

     The chip's `onClick` in the existing render already handles toggle: if `activeQuickFilter === qf.id` it calls `clearFilters()`, otherwise it calls `qf.apply()`. However "Action Required" is purely client-side and does not set `activeQuickFilter` via `updateModels`. Instead, `actionRequiredActive` tracks its state independently. Update the chip render block so that for the `actionRequired` id, it reads from `actionRequiredActive` for `variant`/`color` and calls `setActionRequiredActive(false)` to deactivate.

  5. **Apply the client-side row filter** — derive filtered arrays before passing to `<Orders>`:

     ```typescript
     const filteredNewOrders = actionRequiredActive
       ? newOrders.data.filter((o) => (unreadCountsByOrderId[o._id] ?? 0) > 0)
       : newOrders.data;
     ```

     Pass `filteredNewOrders` (and equivalents for maintenance/lead if the filter applies to those tabs) as the `orders` prop. Pass the original `newOrders.total` as `totalOrders` — the badge filter is a client-side view, not a server-side pagination change.

  Run Step 7 tests to confirm green, then run the full test suite with `npm run test -- --testPathPattern="orders-tracker"`.

---

## Files Affected

| File                                                                   | Action | Description                                                                                                           |
| ---------------------------------------------------------------------- | ------ | --------------------------------------------------------------------------------------------------------------------- |
| `apps/web/app/api/notifications/counts/route.ts`                       | Create | `POST /api/notifications/counts` — batch per-order unread count endpoint                                              |
| `apps/web/app/api/notifications/counts/route.test.ts`                  | Create | Unit tests for the counts API route (auth, flag, validation, happy path, error)                                       |
| `apps/web/app/orders-tracker/hooks/useOrderNotificationCounts.ts`      | Create | Custom hook: fetches batch counts, subscribes to SignalR events, returns `Record<string, number>`                     |
| `apps/web/app/orders-tracker/hooks/useOrderNotificationCounts.test.ts` | Create | Unit tests for the hook (flag gate, fetch, SignalR subscription, cleanup)                                             |
| `apps/web/app/orders-tracker/Orders/NewOrders.tsx`                     | Modify | Add `unreadCountsByOrderId` + `currentUserId` params; render MUI `Badge` in `displayId` cell                          |
| `apps/web/app/orders-tracker/Orders/NewOrders.test.tsx`                | Modify | Add badge rendering tests for the `displayId` column                                                                  |
| `apps/web/app/orders-tracker/[category]/page.tsx`                      | Modify | Call `useOrderNotificationCounts`, pass badge props to columns, add "Action Required" chip and client-side row filter |
| `apps/web/app/orders-tracker/[category]/page.test.tsx`                 | Modify | Add tests for "Action Required" chip visibility, toggle, and row filtering                                            |

---

## Testing Strategy

**Unit tests — API route (`counts/route.test.ts`)**:

- 401 with no session
- 401 with session but no `userObj`
- 404 when `ORDER_DOCUMENT_NOTIFICATIONS` flag is disabled
- 400 when body is not JSON or `orderIds` is missing
- 400 when `orderIds` has more than 100 items
- 200 returns `{ counts: { 'order-1': 2, 'order-2': 0 } }` for two order IDs
- 200 with empty array returns `{ counts: {} }`
- 500 when `getCountForEntity` throws

**Unit tests — hook (`useOrderNotificationCounts.test.ts`)**:

- Returns `{}` immediately when flag is off (no fetch called)
- Returns `{}` when session is absent
- Calls `POST /api/notifications/counts` with correct body on mount
- Returns correct `Record<string, number>` after successful fetch
- Re-fetches when `orderIds` array content changes (not just reference)
- Does NOT re-fetch when `orderIds` content is identical (stable serialization)
- `connection.on('documentAdded', ...)` registered when SignalR connected
- `connection.on('notification:read', ...)` registered when SignalR connected
- Both listeners cleaned up on unmount (`connection.off` called)
- Returns `{}` (no throw) when fetch returns non-ok response
- Returns `{}` (no throw) when fetch throws network error

**Component tests — badge (`NewOrders.test.tsx` additions)**:

- No badge rendered when `unreadCountsByOrderId` is undefined
- No badge rendered when count is 0 for the order
- No badge rendered when `order.assignedTo !== currentUserId`
- Badge renders when count > 0 and user is assigned (`data-testid="notification-badge-{orderId}"`)
- Badge content matches the count value

**Component tests — page filter (`page.test.tsx` additions)**:

- "Action Required" chip absent when feature flag is false
- "Action Required" chip present when feature flag is true and tab is 'new'
- Clicking inactive chip marks it active (soft variant)
- Clicking active chip deactivates it (outlined variant)
- When active: only orders with count > 0 rendered in the grid
- When inactive: all orders rendered
- `useOrderNotificationCounts` receives the IDs of currently displayed orders

**Mock boundaries**:

- API route tests: mock `getCountForEntity` from `@/services/mongodb/notification`, `getServerSession` from `next-auth`, `FeatureFlagService.getFeatureFlag`
- Hook tests: mock `global.fetch`, `useFeatureFlag`, `useSession` from `next-auth/react`, `useSignalR` from `@/utils/context/signalr-context`
- Page tests: mock `useOrderNotificationCounts` from `@/app/orders-tracker/hooks/useOrderNotificationCounts`

**Manual verification**:

- Enable `ORDER_DOCUMENT_NOTIFICATIONS` flag in `/admin/feature-flags`
- Assign a test order to the logged-in user and upload a document to it
- Verify badge appears on the Order ID cell with count = 1
- Verify badge does not appear on orders assigned to other users
- Toggle "Action Required" chip — verify only orders with badges remain visible
- Mark the notification as read (e.g., via the notification panel from D2) — verify badge clears without page refresh

---

## Security Considerations

- **Input validation**: `POST /api/notifications/counts` validates that `orderIds` is a non-empty string array, max 100 items. Each item is passed directly to MongoDB via `getCountForEntity` which already uses `$match` with `$elemMatch` — no raw query injection risk. Still, trim and reject obviously invalid IDs (empty string) at the route boundary.
- **Authorization**: The route enforces `getServerSession` auth before any data access. `getCountForEntity` uses `buildUserTargetFilter` which scopes results to the user's `userId`, `locationIds`, and `role` — users can only see counts for notifications they are authorized to receive.
- **Feature flag dual enforcement**: Feature flag checked at API route (server). Feature flag also checked client-side via `useFeatureFlag` in the hook, preventing the fetch from being initiated, and via `display: isNotificationsEnabled` on the quick filter chip.
- **PHI**: Order IDs passed in the request body and returned counts are not PHI. No patient names, SSNs, or clinical data in the request or response.
- **Over-fetching**: The 100-item cap on `orderIds` prevents abuse of the batch endpoint. The page already paginates at 100 rows per page, so this cap aligns naturally.

---

## Resolved Questions

1. **UI component imports**: **Resolved.** Prefer `@repo/ui`. If `Badge` isn't available there, fall back to `@mui/material` directly or build a custom component. Will check during implementation and ask if unsure.
2. **Scope — new orders only**: **Resolved.** Badges and "Action Required" filter are scoped to the new orders tab only. Maintenance/lead tabs are out of scope for this task.
3. **"Action Required" for maintenance/lead**: **Resolved.** Not applicable — question 2 scopes everything to new orders only.

---

## Update 2026-04-14 — Badge Counts Must Be Globally Visible

**Trigger:** Manual QA on `/orders-tracker/new` after merging this branch revealed that badges rendered only for the viewer's own assigned orders. Alejandro's expectation (business requirement): every authenticated user should see the same badge count on every order, regardless of who it's assigned to.

**Rule restated:**

- **Who sees the badge count** → everyone (team dashboard behavior)
- **Who can click "Mark as Read"** → only the notification's `targetUserId` (ownership unchanged)

See the epic doc's "Update 2026-04-14 — Decouple Badge Visibility from Notification Ownership" section for the architectural rationale.

### What this implementation needs to change

**1. Service layer (`apps/web/services/mongodb/notification.ts`)**

- `getCountForEntity()` — remove `buildUserTargetFilter(params)` from the `$match`. The entity filter alone is enough.
- For `type: 'user'` unread calculation in the same function: change the `notificationreads` `$lookup` to compare `userId` against `notification.targetUserId` (pull `targetUserId` into the sub-pipeline via `$let`), **not** `params.userId`. A notification is unread while its target owner hasn't acknowledged it; that's the same truth for every viewer.
- The `UserTargetParams` shape on `getCountForEntity` becomes partly vestigial — `userId` / `locationIds` / `role` are no longer used by this function. Simplify the param signature: drop them (or make them optional) so callers don't misleadingly supply viewer identity that does nothing.

**2. API route (`apps/web/app/api/notifications/counts/route.ts`)**

- Keep the session-auth check and the feature-flag gate — they don't change.
- Stop passing `userId` / `locationIds` / `role` to `getCountForEntity`. Passing viewer identity into a now-global count would be misleading.
- Route test must cover: a user with **no** notifications targeting them still gets non-zero counts for orders where someone _else_ has unread notifications.

**3. Tests to add / update**

- `notification.test.ts`:
  - ✅ Counts a `user`-type notification as unread when its `targetUserId` has no `NotificationRead` record — regardless of which user is viewing.
  - ✅ Counts drops to 0 only when the `targetUserId` (not the viewer) has a `NotificationRead` record.
  - ✅ Counts a `task`-type notification as unread based on `isRead` alone (unchanged).
  - ✅ Viewer with mismatched `userId` / `locationIds` / `role` still sees the count.
- `counts/route.test.ts`:
  - ✅ Returns non-zero counts for orders whose notifications target a different user.
  - ✅ Still returns 401 for unauthenticated requests; still returns 404 when feature flag is off.

**4. Frontend badge gating (`apps/web/app/orders-tracker/Orders/NewOrders.tsx`)**

The current column renderer contains a client-side gate that must also be removed:

```typescript
const showBadge =
  unreadCount > 0 &&
  !!currentUserId &&
  params.row.assignedTo === currentUserId; // ← remove this condition
```

- Drop the `params.row.assignedTo === currentUserId` check. Badge renders whenever `unreadCount > 0`.
- `currentUserId` prop on `getNewOrdersColumns` and the corresponding plumbing in `page.tsx` (step 8 from the original plan) become unused — remove them to avoid dead props.
- Update step 5's test for "Badge is not rendered when `order.assignedTo !== currentUserId`" → replace with "Badge **is** rendered regardless of `assignedTo` when count > 0".

**5. What does NOT change in this task**

- Step 5 badge markup itself (icon, styling, layout) — unchanged; only the render condition changes.
- `useOrderNotificationCounts` hook — no change.
- Header bell (`getNotificationsForUser` / `getUnreadCountForUser`) — out of scope. Header inbox remains strictly per-viewer.
- "Mark as Read" button visibility — already correctly gated to the targeted user; this task does not expose it to non-targets.

### Follow-up sequence

1. 🔴 Update `notification.test.ts` with new cross-user visibility cases (will fail).
2. 🟢 Change `getCountForEntity` pipeline: drop viewer filter, switch `notificationreads` `$lookup` to key on `targetUserId`.
3. 🔴 Update `counts/route.test.ts` to assert non-zero counts for other users' orders.
4. 🟢 Simplify route to stop passing viewer identity to `getCountForEntity`.
5. 🔴 Update `NewOrders.test.tsx` — replace the "only assignee sees badge" case with a "badge renders for any viewer" case.
6. 🟢 Remove the `assignedTo === currentUserId` check from `NewOrders.tsx`; strip the now-unused `currentUserId` prop from `getNewOrdersColumns` and its callsite in `page.tsx`.
7. 🔵 Refactor: tighten `getCountForEntity` param signature; run types/lint.
8. Manual verification with seeded notifications targeting _another_ user — confirm Alejandro sees the badge count on `/orders-tracker/new`.

---

## Update 2026-04-15 — "Action Required" Filter Deferred

**Trigger:** Product owner removed the "Action Required" quick filter from the MLID-2011 epic scope. The filter ships disabled/absent from the UI; we'll revisit it as a separate ticket later if demand returns.

**Rationale for deferral:**

- The filter's client-side implementation (local `useState`, post-pagination `.filter()`) diverged from the URL-driven quick-filter pattern used elsewhere on the page: no URL param, no active-filter chip, not reset by "Clear filters". Making it behave like the others requires pushing the `unreadCount > 0` predicate into the `neworders` MongoDB aggregation via a `$lookup` across `notifications` + `notificationreads` — larger than the ticket's remaining budget.
- The **badge** (the primary user-facing signal) works fine standalone. Users can visually scan for badges to find action-required orders until a proper server-side filter ships.

### What was removed from `apps/web/app/orders-tracker/[category]/page.tsx`

1. `actionRequiredActive` `useState` and its tab-change reset `useEffect`.
2. The `actionRequired` entry in the `quickFilters` array (and `isNotificationsEnabled` from that `useMemo`'s deps — it had no other consumer inside the memo).
3. The `isActionRequired` branch in the chip `onClick` handler (handler simplified back to toggle-or-apply on `activeQuickFilter`).
4. The client-side `.filter(...)` ternary wrapping the `orders={}` prop on `<Orders>`.

### What stayed — shared with the badge

Nothing in the data layer was specific to the filter. All of the following remain in place and serve the badge:

- `app/api/notifications/counts/route.ts` (+ test) — batch unread counts endpoint
- `app/orders-tracker/hooks/useOrderNotificationCounts.ts` (+ test) — fetches counts + subscribes to SignalR
- `app/orders-tracker/Orders/NewOrders.tsx` (+ test) — badge column
- `services/mongodb/notification.ts` (+ test) — `getCountForEntity` powering the endpoint

### When it comes back

Preferred direction: server-side predicate. Two paths:

- **(a)** Extend the orders aggregation with a `$lookup` into `notifications` + `notificationreads`, exposing `unreadCount` as a computed field so the standard `filterModel` can carry an `unreadCount > 0` item like any other filter. URL / chip / clear-all then come for free.
- **(b)** Accept `hasUnread: boolean` on the orders endpoint; intersect with IDs returned from a notifications-side aggregation. Cheaper change to the orders pipeline, still URL-driven.

Either way, revive as its own ticket so paginated counts and "total matching" stay consistent.
