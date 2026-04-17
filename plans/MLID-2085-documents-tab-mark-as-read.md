# MLID-2085 — Documents Tab: New Tags, Mark as Read (Single + Bulk)

## Task Reference

- **Jira**: [MLID-2085](https://localinfusion.atlassian.net/browse/MLID-2085)
- **Epic**: [MLID-2011](https://localinfusion.atlassian.net/browse/MLID-2011) — Orders: Document Notifications
- **Epic Plan**: `docs/agomez/plans/MLID-2011-plan-progress.md`
- **Story Points**: 5
- **Branch**: `feature/MLID-2085-documents-tab-mark-as-read`
- **Base Branch**: `epic/MLID-2011-order-document-notifications`
- **Status**: To Do

---

## Summary

Adds per-document notification awareness to the Documents tab: a badge on the tab header showing the unread count for the order, "New" chips on unread document rows, and per-row and bulk "Mark as Read" buttons. The mark-as-read controls are gated UI-only to the order's assigned user. All behavior is gated by the `ORDER_DOCUMENT_NOTIFICATIONS` feature flag and wired to SignalR for real-time updates.

---

## Scope & Acceptance Criteria

- [ ] Badge on the "Documents" tab header showing the unread notification count for the current order (globally visible — same count every authenticated user sees, consistent with D3-T1 behavior)
- [ ] **Section header strip** introduced above the documents table containing the title "Order Documents" (h5) on the left and the "Mark all as Read" button on the right, with a visible vertical gap separating it from the table. Built with `@repo/ui` primitives: outer `Layout gap={4}` for the vertical gap, inner `Layout direction="row" justify="between" align="center"` for the strip
- [ ] "New" chip displayed **inline in the "Received" column**, next to the received date, on each document row that has an unread notification — visible only to the assigned user. Use `@repo/ui` `Chip` with `label="New" size="sm" color="success" variant="soft"` (matches the `li-ui-playground` designer reference)
- [ ] "Mark as Read" button rendered as a **new, dedicated column** on the right side of the table (after "Category", after/replacing the "View" column placement — the "View" link stays; the new column sits next to it). Visible only to the assigned user. Use `@repo/ui` `Button variant="secondary" size="xs"`. Calls `PATCH /api/notifications/:id/read`.
- [ ] "Mark all as Read" button lives in the new section header strip described above (top-right). Use `@repo/ui` `Button size="sm"` (default/primary variant). **Always rendered** when the feature flag is on; **disabled** when there are zero unread notifications targeting the current user (matches the designer reference — `disabled={docCounts.newCount === 0}` — not conditionally hidden). Calls `PATCH /api/notifications/bulk-read`, actionable only for the assigned user
- [ ] Real-time updates via SignalR events `documentAdded` and `notification:read` — refetch notification data when either fires
- [ ] All notification controls hidden when `ORDER_DOCUMENT_NOTIFICATIONS` feature flag is off (the section header strip itself remains when the flag is on even if there are no unread notifications — the "Mark all as Read" button just hides; when the flag is off, the strip reverts to the previous layout to avoid an empty header)
- [ ] Unit/component tests for all new behavior

---

## Codebase Analysis

**Page under modification:**
- `apps/web/app/orders-tracker/[category]/[orderId]/documents/page.tsx` — client component, renders `<table>` from `useOrderDetails().data.documents`. Each row has fields: `id`, `name`, `receivedDate`, `category`, `viewUrl`. Existing test: `documents/page.test.tsx`.
- `apps/web/app/orders-tracker/[category]/[orderId]/layout.tsx` — client component, renders tab bar with the `Tab` component. Existing test: `layout.test.tsx`.

**Tab component:**
- `apps/web/components/UI/Tab/Tab.tsx` — accepts `tabTitle: string`, `isActive: boolean`, `onClick`. Renders a `<div>` with a `<span>`. No badge slot. Must be extended with `badge?: ReactNode`.
- `apps/web/components/UI/Tab/index.ts` — re-exports from `./Tab`.

**`OrderDetailsData` type:**
- `apps/web/types/orders.ts` line 214 — does not include `assignedTo`. The raw `OrderBaseDTO` at line 109 has `assignedTo: UserDTO['_id'] | string`. The API route at `apps/web/app/api/orders/details/route.ts` constructs the `OrderDetailsData` payload manually and omits this field. Must be added.

**Details API route:**
- `apps/web/app/api/orders/details/route.ts` — GET handler builds `OrderDetailsData` from `trackerOrder`. `trackerOrder.assignedTo` is available from `getOrderById`. The `assignedTo` value is a string (user `_id`) or possibly populated as an object (per `FullOrder` type). Need to safely coerce to `string | null`.

**Notification service:**
- `apps/web/services/mongodb/notification.ts` — has `getCountForEntity`, `markUserNotificationAsRead`, `bulkMarkUserNotificationsAsRead`, `isReadByUser`. Does not have a query that returns notification documents scoped to an order along with viewer-scoped read status. A new `getNotificationsForEntity` helper is needed.

**Existing notification API routes:**
- `PATCH /api/notifications/[id]/read` — `apps/web/app/api/notifications/[id]/read/route.ts` — marks single notification read, broadcasts `notification:read` SignalR event.
- `PATCH /api/notifications/bulk-read` — `apps/web/app/api/notifications/bulk-read/route.ts` — accepts `{ notificationIds: string[] }` (max 100), broadcasts `notification:read`.
- `POST /api/notifications/counts` — per-order unread counts (global).
- `GET /api/notifications` — user inbox.
- No endpoint returns notification documents scoped to a single order. Must be created.

**Hook precedent:**
- `apps/web/app/orders-tracker/hooks/useOrderNotificationCounts.ts` — fetches `POST /api/notifications/counts`, subscribes to `documentAdded` and `notification:read`, gates on feature flag + session. The new per-order hook follows the exact same shape.

**SignalR:**
- `useSignalR()` from `@/utils/context/signalr-context` — exposes `connection` and `isConnected`. Events subscribed via `connection.on(event, handler)`, cleaned up with `connection.off`.

**UI primitives:**
- `@repo/ui` has a `Chip` component — use `size="sm"`, `color="primary"`, `variant="filled"` for the "New" tag.
- No count-badge component in `@repo/ui`. Use `@mui/material/Badge` with `badgeContent` to wrap the tab title in `Tab`, or pass the badge as a `ReactNode` child that renders it inline. The inline `ReactNode` approach (via the new `badge` prop on `Tab`) is preferred — it keeps `Tab` generic and avoids coupling the component to MUI Badge.

---

## Gap Resolutions

### Gap 1 — `assignedTo` not on `OrderDetailsData`

**Decision: Option (a) — add `assignedTo: string | null` to `OrderDetailsData`.**

Rationale: the field is already available on `trackerOrder` returned by `getOrderById`. Adding it to `OrderDetailsData` is a one-line type change plus a one-line addition to the payload construction. A second round-trip would add latency and complexity with no benefit.

`trackerOrder.assignedTo` is typed as `UserDTO['_id'] | string` on the base DTO. After `getOrderById` with relations it could be a populated `UserDTO` object (see `FullOrder.assignedToUser`). The route must safely coerce: if `typeof trackerOrder.assignedTo === 'string'`, use directly; otherwise fall back to `null`. This mirrors the `toStringValue` pattern already used in the route for other fields.

### Gap 2 — No per-document notification lookup endpoint

**Decision: Create `GET /api/notifications/by-order/[orderId]/route.ts`.**

Response shape:
```
{
  notifications: Array<{
    _id: string
    entities: Array<{ entityType: string; entityId: string }>
    isReadByCurrentUser: boolean
    targetUserId: string | null
    createdAt: string  // ISO
  }>
}
```

The endpoint returns all non-expired notifications whose `entities` array contains `{ entityType: 'order', entityId: orderId }`. For each notification, `isReadByCurrentUser` is computed using `isReadByUser(notif._id, session.user.id)` — this is viewer-scoped. The "New" chip visibility is decided by the UI (assignee check + `!isReadByCurrentUser`).

A new service helper `getNotificationsForEntity({ entityType, entityId })` returns the raw notification documents (lean). The route calls it, then resolves `isReadByCurrentUser` for each in `Promise.all`. The existing `isReadByUser` function is used as-is — no new service function is needed beyond `getNotificationsForEntity`.

The endpoint is authenticated + feature-flag gated (same pattern as all other notification routes). No server-side assignee check is performed — consistent with the existing `PATCH /api/notifications/[id]/read` behavior. This is documented explicitly as UI-only gating.

### Gap 3 — `Tab` component has no badge slot

**Decision: Add `badge?: ReactNode` prop to `Tab`.**

The badge prop is rendered inside the tab `<div>` after the `<span>` containing `tabTitle`. The layout component passes whatever count-badge element it wants — keeping `Tab` generic. No MUI import is added to `Tab` itself. The badge content in `layout.tsx` will be a small inline span styled inline (matching the D3-T1 hand-rolled pattern), keeping the implementation minimal.

---

## Implementation Steps

### Step 1 — Add `assignedTo` to `OrderDetailsData` type and API route

**Commit**: `[MLID-2085] - feat(order-details): expose assignedTo on OrderDetailsData`

**Files**:
- `apps/web/types/orders.ts` (modify)
- `apps/web/app/api/orders/details/route.ts` (modify)
- `apps/web/app/api/orders/details/route.test.ts` (modify — add test case)

**RED**: Add a test case to the existing route test file asserting that the GET response payload includes `assignedTo` as a string when the order has an `assignedTo` string value, and `null` when it is absent or unpopulated.

**GREEN**: Add `assignedTo: string | null` to `OrderDetailsData` in `types/orders.ts`. In the route, extract `assignedTo` from `trackerOrder`: if `typeof trackerOrder.assignedTo === 'string'`, use it; otherwise `null`. Add `assignedTo` to the `payload` object.

**REFACTOR**: Ensure the existing `toStringValue` helper or a comparable extraction is used consistently. Verify existing tests still pass.

---

### Step 2 — Service helper: `getNotificationsForEntity`

**Commit**: `[MLID-2085] - feat(notification-service): add getNotificationsForEntity helper`

**Files**:
- `apps/web/services/mongodb/notification.ts` (modify)
- `apps/web/services/mongodb/notification.test.ts` (modify — add test cases)

**RED**: Write tests for `getNotificationsForEntity({ entityType, entityId })`:
- returns all non-expired notifications matching the entity
- excludes notifications where `expiresAt` is in the past
- returns an empty array when none match
- throws and logs on database error

**GREEN**: Add `getNotificationsForEntity` to the service. Uses `Notification.find({ entities: { $elemMatch: { entityType, entityId } }, $and: [{ $or: [{ expiresAt: null }, { expiresAt: { $gt: new Date() } }] }] }).sort({ createdAt: -1 }).lean()`. Returns `INotification[]`.

**REFACTOR**: Extract the shared expiry filter expression into a named constant `ACTIVE_EXPIRY_FILTER` to avoid duplication with `buildUserTargetFilter` and `getCountForEntity`.

---

### Step 3 — New API endpoint: `GET /api/notifications/by-order/[orderId]`

**Commit**: `[MLID-2085] - feat(notifications-api): add GET by-order endpoint`

**Files**:
- `apps/web/app/api/notifications/by-order/[orderId]/route.ts` (create)
- `apps/web/app/api/notifications/by-order/[orderId]/route.test.ts` (create)

**RED**: Write tests:
- returns 401 when no session
- returns 401 when session has no `userObj`
- returns 404 when feature flag is disabled
- returns 200 with `notifications` array; each item has `_id`, `entities`, `isReadByCurrentUser`, `targetUserId`, `createdAt`
- `isReadByCurrentUser` is `true` when `NotificationRead` record exists for session user
- `isReadByCurrentUser` is `false` when no read record exists
- returns 400 when `orderId` param is missing or empty string
- returns 500 when service throws
- calls `getNotificationsForEntity` with `{ entityType: 'order', entityId: orderId }`

**GREEN**: Create the route. Auth + feature-flag guard (same boilerplate as existing routes). Call `getNotificationsForEntity({ entityType: 'order', entityId: orderId })`. For each notification, call `isReadByUser(String(notif._id), userId)` inside `Promise.all`. Map to response shape. Return 200.

**REFACTOR**: Extract the response item mapping to a typed helper `toNotificationSummary`. Define `NotificationSummary` type in the route file.

---

### Step 4 — Extend `Tab` component with `badge` prop

**Commit**: `[MLID-2085] - feat(tab): add optional badge slot`

**Files**:
- `apps/web/components/UI/Tab/Tab.tsx` (modify)
- `apps/web/components/UI/Tab/Tab.test.tsx` (create)

**RED**: Write tests for `Tab`:
- renders `tabTitle` text
- applies active class when `isActive` is true
- does not render badge area when `badge` prop is omitted
- renders badge content when `badge` prop is provided
- badge content appears alongside the tab title

**GREEN**: Add `badge?: React.ReactNode` to `TabProps`. In the JSX, render `{badge}` after `<span>{tabTitle}</span>` inside the existing `<div>`. No style changes needed for the badge slot — the badge element itself is responsible for its appearance.

**REFACTOR**: Ensure `TabProps` interface is clean. Confirm `Tab.test.tsx` covers all prop combinations.

---

### Step 5 — Hook: `useDocumentsTabNotifications(orderId)`

**Commit**: `[MLID-2085] - feat(documents-tab): add useDocumentsTabNotifications hook`

**Files**:
- `apps/web/app/orders-tracker/[category]/[orderId]/documents/useDocumentsTabNotifications.ts` (create)
- `apps/web/app/orders-tracker/[category]/[orderId]/documents/useDocumentsTabNotifications.test.ts` (create)

**Return shape**:
```typescript
type DocumentNotificationSummary = {
  notificationId: string
  documentId: string | null   // entityId where entityType === 'document', or null
  isReadByCurrentUser: boolean
  targetUserId: string | null
}

type UseDocumentsTabNotificationsResult = {
  unreadCount: number
  byDocumentId: Record<string, DocumentNotificationSummary>
  notificationIds: string[]   // all notification _ids for the order
  unreadNotificationIdsForCurrentUser: string[]   // subset: unread + targetUserId === session user — posted to bulk-read
  hasUnreadForCurrentUser: boolean   // derived convenience flag for the bulk button's enabled state
  isLoading: boolean
}
```

**Behavior**:
- Gates on `ORDER_DOCUMENT_NOTIFICATIONS` feature flag and `session?.user?.id`. Returns zeroed result when either is absent.
- On mount and when `orderId` changes: fetch `GET /api/notifications/by-order/${orderId}`.
- Compute `byDocumentId` by finding the entity with `entityType === 'document'` in each notification's `entities` array and keying by that `entityId`.
- Compute `unreadCount` as the count of notifications where `!isReadByCurrentUser`.
- Subscribe to `documentAdded` and `notification:read` SignalR events via `connection.on` — each triggers a refetch.
- Clean up listeners on unmount.

**RED**: Write tests:
- returns zeroed result when feature flag is off
- returns zeroed result when no session
- calls `GET /api/notifications/by-order/${orderId}` on mount
- returns correct `unreadCount` based on `isReadByCurrentUser` values
- returns correct `byDocumentId` mapping
- `notificationIds` contains all notification `_id` values
- `unreadNotificationIdsForCurrentUser` excludes already-read and notifications for other users
- `hasUnreadForCurrentUser` is `true` iff `unreadNotificationIdsForCurrentUser` is non-empty
- refetches when `documentAdded` SignalR event fires
- refetches when `notification:read` SignalR event fires
- returns `isLoading: true` while fetch is in flight

**GREEN**: Implement the hook. Follow the `useOrderNotificationCounts` shape exactly for SignalR wiring.

**REFACTOR**: Ensure `byDocumentId` derivation is in a pure function `buildByDocumentId(notifications)` that is unit-testable independently.

---

### Step 6 — Tab header badge wiring in `layout.tsx`

**Commit**: `[MLID-2085] - feat(documents-tab): show unread count badge on tab header`

**Files**:
- `apps/web/app/orders-tracker/[category]/[orderId]/layout.tsx` (modify)
- `apps/web/app/orders-tracker/[category]/[orderId]/layout.test.tsx` (modify — add test cases)

**What**: Import `useDocumentsTabNotifications`. Call it with `orderId`. When `isDocumentsEnabled` is true, pass a badge `ReactNode` to the Documents `Tab` — a `<span>` with the unread count, rendered only when `unreadCount > 0`. Use an inline badge span styled similarly to the D3-T1 hand-rolled badge in the orders tracker table.

**RED**: Add test cases to `layout.test.tsx`:
- Documents tab renders without a badge when `unreadCount` is 0
- Documents tab renders a badge with count "3" when `unreadCount` is 3
- badge is not rendered when `ORDER_DOCUMENT_NOTIFICATIONS` flag is off (mock `useDocumentsTabNotifications` returning `unreadCount: 0`)

Mock `useDocumentsTabNotifications` in the test file: `jest.mock('./documents/useDocumentsTabNotifications', ...)`.

**GREEN**: Wire in the hook call and badge prop in `layout.tsx`. Gate badge rendering on `isNotificationsEnabled` (read `FeatureFlag.ORDER_DOCUMENT_NOTIFICATIONS` using `useFeatureFlag`).

**REFACTOR**: Ensure the badge span has a `data-testid` for reliable test selection. Confirm no regressions in existing layout tests.

---

### Step 7 — "New" chip inline in the Received column

**Commit**: `[MLID-2085] - feat(documents-tab): show New chip in Received column for unread rows`

**Files**:
- `apps/web/app/orders-tracker/[category]/[orderId]/documents/page.tsx` (modify)
- `apps/web/app/orders-tracker/[category]/[orderId]/documents/page.test.tsx` (modify — add test cases)

**What**: Import `useDocumentsTabNotifications`, `useSession`, `useFeatureFlag`, and from `@repo/ui`: `Chip`, `Layout`, `Typography`. For each document row, check `byDocumentId[document.id]`.

The Received cell structure (mirrors `li-ui-playground/src/pages/OrderDetail.tsx:193-197` — the canonical designer reference):

```tsx
<Layout direction="row" gap={2} align="center">
  <Typography variant="bodySmall">{formatDate(document.receivedDate)}</Typography>
  {isDocumentUnreadForUser(summary, session?.user?.id) && (
    <Chip label="New" size="sm" color="success" variant="soft" />
  )}
</Layout>
```

The chip renders when all of:
1. `isNotificationsEnabled` (feature flag)
2. `byDocumentId[document.id]` exists
3. `!byDocumentId[document.id].isReadByCurrentUser`
4. `byDocumentId[document.id].targetUserId === session?.user?.id`

The `orderId` needed for the hook is available from `useOrderDetails().data.id`. The existing "Received Date" column header stays — only the cell content changes.

No custom CSS class is needed for the cell — `Layout` from `@repo/ui` handles the row/gap/alignment.

**RED**: Add test cases:
- "New" chip is shown for a document whose `byDocumentId` entry has `isReadByCurrentUser: false` and `targetUserId` matching the current user
- "New" chip is not shown when `isReadByCurrentUser` is true
- "New" chip is not shown when `targetUserId` does not match the current user
- "New" chip is not shown when feature flag is off
- "New" chip is not shown when `byDocumentId` has no entry for the document id

Mock `useDocumentsTabNotifications` and `useSession` in the test file.

**GREEN**: Implement the conditional rendering. Import `Chip` from `@repo/ui`.

**REFACTOR**: Extract the visibility predicate to a named function `isDocumentUnreadForUser(summary, userId)` for clarity and testability.

---

### Step 8 — "Mark as Read" per-row button (new column)

**Commit**: `[MLID-2085] - feat(documents-tab): add per-row Mark as Read column`

**Files**:
- `apps/web/app/orders-tracker/[category]/[orderId]/documents/page.tsx` (modify)
- `apps/web/app/orders-tracker/[category]/[orderId]/documents/page.test.tsx` (modify — add test cases)

**What**: Add a **new dedicated column** to the documents table after "Category". The column header is an explicit empty string (matches the designer reference — `headerName: ''` at `li-ui-playground/src/pages/OrderDetail.tsx:209`). Column placement: **between "Category" and "View"** (keeps the View link intact). Column cells right-aligned. Column is always rendered when the feature flag is on (avoids table layout shift on mark-as-read); cells are empty for rows that don't qualify.

For each row where the assignee condition holds (same predicate as the "New" chip), render the "Mark as Read" button inside this new column cell. Use `@repo/ui` `Button` with `variant="secondary"` and `size="xs"` — matches the designer reference exactly (`li-ui-playground/src/pages/OrderDetail.tsx:219-225`):

```tsx
<Button variant="secondary" size="xs" onClick={() => handleMarkAsRead(summary.notificationId)}>
  Mark as Read
</Button>
```

On click, call `PATCH /api/notifications/:id/read` where `:id` is `byDocumentId[document.id].notificationId`. While the request is in flight, disable the button. On success, the SignalR `notification:read` event broadcast by the server triggers a refetch in the hook — no manual state update needed.

Visibility (button): same four conditions as the "New" chip. Column (header + empty cells): rendered whenever the feature flag is on.

**RED**: Add test cases:
- the new column header is rendered when the feature flag is on
- the new column is not rendered when the feature flag is off
- "Mark as Read" button is rendered in the new column for rows meeting the assignee+unread condition
- "Mark as Read" button is not rendered (cell is empty) when conditions are not met (not assignee, already read)
- clicking "Mark as Read" calls `PATCH /api/notifications/:id/read` with the correct notification ID
- button is disabled while the request is in flight

**GREEN**: Conditionally render the new `<th>` and `<td>`s based on `isNotificationsEnabled`. Add the handler `handleMarkAsRead(notificationId: string)` inside the component. Use a `Set<string>` in local state to track in-flight requests. Render the button inside the new `<td>` only when the assignee predicate is true; otherwise render an empty `<td>`.

**REFACTOR**: Extract `handleMarkAsRead` and loading-state tracking into a small custom hook `useMarkAsRead()` co-located in the documents directory.

---

### Step 9 — Section header strip + "Mark all as Read" bulk button

**Commit**: `[MLID-2085] - feat(documents-tab): add Order Documents section header with Mark all as Read`

**Files**:
- `apps/web/app/orders-tracker/[category]/[orderId]/documents/page.tsx` (modify)
- `apps/web/app/orders-tracker/[category]/[orderId]/documents/page.test.tsx` (modify — add test cases)

**What**: Restructure the page to split the previous single table into two stacked regions (header strip + table) separated by a vertical gap. Use `@repo/ui` `Layout` + `Typography` + `Button` to match the designer reference at `li-ui-playground/src/pages/OrderDetail.tsx:673-678` — **no custom CSS is needed**.

```tsx
<Layout gap={4} padding={[4, 0]}>
  {/* Section header strip */}
  <Layout direction="row" justify="between" align="center">
    <Typography variant="h5">Order Documents</Typography>
    {isNotificationsEnabled && (
      <Button
        size="sm"
        onClick={handleMarkAllAsRead}
        disabled={!hasUnreadForCurrentUser || isBulkLoading}
      >
        Mark all as Read
      </Button>
    )}
  </Layout>

  {/* Table below — existing <table> structure, unchanged beyond the new column from Step 8 */}
  <table>…</table>
</Layout>
```

1. **Outer `Layout gap={4} padding={[4, 0]}`** — produces the vertical gap between the header strip and the table. No custom spacer element, no CSS class. `@repo/ui` tokens handle the spacing.
2. **Inner `Layout direction="row" justify="between" align="center"`** — the strip itself.
3. **Title** — `<Typography variant="h5">Order Documents</Typography>`.
4. **"Mark all as Read" button** — `@repo/ui` `Button size="sm"` (default/primary variant; green per the designer).

The "No documents available" empty state path also gets the section header strip (same structure) to keep visual consistency.

**"Mark all as Read" visibility & enablement** — matches the designer reference (`disabled={docCounts.newCount === 0}`):
- **Rendered** whenever the feature flag is on (does not depend on assignee)
- **Disabled** when:
  - `notificationIds` is empty, OR
  - no unread notification targets the current user (i.e., `hasUnreadForCurrentUser === false`), OR
  - a request is in flight (`isBulkLoading === true`)

Defining `hasUnreadForCurrentUser`: derived from the hook — `true` iff some `byDocumentId` entry has `!isReadByCurrentUser && targetUserId === session.user.id`. Expose this as a field on the hook's return (`hasUnreadForCurrentUser: boolean`) to keep the page thin.

On click, call `PATCH /api/notifications/bulk-read` with `{ notificationIds }` (the full list from the hook — scoped to unread notifications targeting the current user; filter the list before POST to avoid touching notifications the user can't act on). While in flight, `isBulkLoading` is true and the button is disabled. On success, the SignalR `notification:read` broadcast triggers the hook refetch.

**RED**: Add test cases:
- section header strip is rendered with title "Order Documents"
- section header strip is rendered even when the documents list is empty
- "Mark all as Read" button is rendered when the feature flag is on
- "Mark all as Read" button is not rendered when the feature flag is off
- "Mark all as Read" button is **disabled** when `hasUnreadForCurrentUser` is false (including: `notificationIds` empty; all notifications already read; only notifications targeting other users)
- "Mark all as Read" button is **enabled** when there is at least one unread notification targeting the current user
- clicking "Mark all as Read" calls `PATCH /api/notifications/bulk-read` with the list of **unread, current-user-targeted** notification IDs
- button is disabled while the request is in flight

**GREEN**: Introduce the outer `Layout` wrapper as the first element returned by the page. Add `handleMarkAllAsRead` handler. Reuse/extend `useMarkAsRead` from Step 8 to also expose `handleMarkAllAsRead` and `isBulkLoading`, or add a sibling `useMarkAllAsRead` hook — whichever keeps the component thinner. Extend `useDocumentsTabNotifications` to surface `hasUnreadForCurrentUser` (derived from `byDocumentId` + session userId) and `unreadNotificationIdsForCurrentUser` (the filtered subset posted to `bulk-read`).

**REFACTOR**: Review the final `page.tsx` for readability. Extract any inline JSX conditionals that exceed two conditions into well-named predicates. If the section header strip logic is non-trivial, extract it into a small local component `DocumentsSectionHeader` within the documents directory.

---

### Step 10 — Manual QA

No commit. Verification checklist against a seeded order with notifications (see QA checklist below).

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/types/orders.ts` | Modify | Add `assignedTo: string \| null` to `OrderDetailsData` |
| `apps/web/app/api/orders/details/route.ts` | Modify | Populate `assignedTo` in payload from `trackerOrder.assignedTo` |
| `apps/web/app/api/orders/details/route.test.ts` | Modify | Add test for `assignedTo` in response |
| `apps/web/services/mongodb/notification.ts` | Modify | Add `getNotificationsForEntity` helper; extract shared expiry filter |
| `apps/web/services/mongodb/notification.test.ts` | Modify | Add tests for `getNotificationsForEntity` |
| `apps/web/app/api/notifications/by-order/[orderId]/route.ts` | Create | GET endpoint returning order-scoped notifications with `isReadByCurrentUser` |
| `apps/web/app/api/notifications/by-order/[orderId]/route.test.ts` | Create | Tests for the new endpoint |
| `apps/web/components/UI/Tab/Tab.tsx` | Modify | Add `badge?: ReactNode` prop |
| `apps/web/components/UI/Tab/Tab.test.tsx` | Create | Tests for `Tab` with and without badge |
| `apps/web/app/orders-tracker/[category]/[orderId]/documents/useDocumentsTabNotifications.ts` | Create | Hook: fetches by-order notifications, maps to document-keyed data, wires SignalR |
| `apps/web/app/orders-tracker/[category]/[orderId]/documents/useDocumentsTabNotifications.test.ts` | Create | Tests for the hook |
| `apps/web/app/orders-tracker/[category]/[orderId]/layout.tsx` | Modify | Wire badge into Documents `Tab` using `useDocumentsTabNotifications` |
| `apps/web/app/orders-tracker/[category]/[orderId]/layout.test.tsx` | Modify | Add badge visibility tests |
| `apps/web/app/orders-tracker/[category]/[orderId]/documents/page.tsx` | Modify | Section header strip (via `@repo/ui` `Layout` + `Typography` + `Button`), "New" chips inline in the Received column (via `@repo/ui` `Chip` + `Layout`), new "Mark as Read" column on the right of the table |
| `apps/web/app/orders-tracker/[category]/[orderId]/documents/page.test.tsx` | Modify | Tests for section header, chips, and buttons |

---

## Testing Strategy

### Unit tests — service layer (`notification.test.ts`)

- `getNotificationsForEntity` returns all non-expired notifications matching entity
- `getNotificationsForEntity` excludes expired notifications
- `getNotificationsForEntity` returns empty array when no match
- `getNotificationsForEntity` throws and logs on DB error

### Integration tests — API routes (node environment, jest.mock at boundaries)

**`GET /api/orders/details` (existing test extended)**:
- response includes `assignedTo` as a string when `trackerOrder.assignedTo` is a string
- response includes `assignedTo: null` when `trackerOrder.assignedTo` is an object (populated relation)

**`GET /api/notifications/by-order/[orderId]`**:
- 401 with no session
- 401 with no `userObj`
- 404 when feature flag disabled
- 200 with correct `notifications` array shape
- `isReadByCurrentUser: true` when `NotificationRead` record exists
- `isReadByCurrentUser: false` when no record
- 500 when `getNotificationsForEntity` throws

### Component tests (RTL)

**`Tab.test.tsx`**:
- renders title
- applies active style class
- omits badge when prop not passed
- renders badge content when prop passed

**`useDocumentsTabNotifications.test.ts`**:
- zeroed result when flag off
- zeroed result when no session
- correct `unreadCount` computation
- correct `byDocumentId` keying
- correct `notificationIds` list
- refetch on `documentAdded`
- refetch on `notification:read`
- `isLoading` state transitions

**`layout.test.tsx` (extended)**:
- Documents tab badge renders with count when `unreadCount > 0`
- Documents tab badge absent when `unreadCount === 0`
- badge absent when notifications flag off

**`page.test.tsx` (extended)**:
- "New" chip shown for unread document matching current user
- "New" chip absent when already read
- "New" chip absent when `targetUserId` does not match
- "New" chip absent when flag off
- "Mark as Read" button shown for qualifying row
- "Mark as Read" button absent for non-qualifying row
- clicking "Mark as Read" calls `PATCH /api/notifications/:id/read`
- "Mark all as Read" button shown when unread count > 0 for current user
- "Mark all as Read" button absent when no unread
- clicking "Mark all as Read" calls `PATCH /api/notifications/bulk-read`

### Mock boundaries

- `@/services/mongodb/notification` — mocked in all API route tests
- `@/services/featureFlags/featureFlagService` — mocked in all API route tests
- `next-auth` (`getServerSession`) — mocked in all API route tests
- `@/utils/context/signalr-context` (`useSignalR`) — mocked in hook and component tests
- `@/utils/contexts/FeatureFlagContext` (`useFeatureFlag`) — mocked in component tests
- `next-auth/react` (`useSession`) — mocked in component tests
- `./documents/useDocumentsTabNotifications` — mocked in `layout.test.tsx`
- `../OrderDetailsContext` (`useOrderDetails`) — already mocked in `page.test.tsx`
- `global.fetch` — mocked in hook tests

---

## Security Considerations

- **Authentication**: All new API endpoints (`GET /api/notifications/by-order/[orderId]`) require a valid NextAuth session and `userObj`. 401 returned otherwise.
- **Feature flag gating**: New endpoint gated by `ORDER_DOCUMENT_NOTIFICATIONS` server-side via `FeatureFlagService`. Client components gate via `useFeatureFlag`. Both levels enforced.
- **Assignee check — UI only**: "Mark as Read" buttons and "New" chips are rendered only for the assigned user (`targetUserId === session.user.id`). The server does NOT enforce assignee identity on `PATCH /api/notifications/[id]/read` or `PATCH /api/notifications/bulk-read` — consistent with the existing architecture. Any authenticated user who knows a notification ID can mark it read via direct API call. This is an accepted risk, documented here, and may be hardened in a future task.
- **Input validation**: The `orderId` route segment is passed to `getNotificationsForEntity` after basic string validation. No ObjectId casting is performed — the service uses string-based `entityId` matching, consistent with how all other entity IDs are stored.
- **PHI**: Notifications contain order and document IDs only — no patient names, SSNs, or clinical data are returned by the new endpoint.

---

## Risks & Known Issues

1. **BSON ESM issue**: Existing `documents/page.test.tsx` uses RTL and has no known BSON ESM conflict. The new tests follow the same RTL pattern. If BSON import issues surface (as seen in order-history tests), add `@jest-environment jsdom` or ensure the test does not import from `apps/web/types/orders.ts` directly — use mock data inline.

2. **UI-only assignee gate**: As documented above, any authenticated user can call `PATCH /api/notifications/:id/read` with any notification ID. This is the existing behavior and is not introduced by this task. If enforcement is needed, it must be a separate task adding an `assignedTo` check to the PATCH route.

3. **`assignedTo` type variance**: `trackerOrder.assignedTo` may be a string (raw ID) or a populated `UserDTO` object depending on whether `getOrderById` populates it. The route must guard against the populated case using `typeof value === 'string'` and fall back to `null`. A test covering the object case is included in Step 1.

4. **Notification `_id` as string**: `INotification._id` is a Mongoose `ObjectId`. After `.lean()`, it is a plain BSON `ObjectId`. The route must call `String(notif._id)` before including it in the JSON response. This is a consistent pattern in existing routes.

5. **Stale `byDocumentId` after mark-as-read**: When the user clicks "Mark as Read", the SignalR `notification:read` broadcast triggers a refetch in the hook. There is a brief interval (~network RTT) where the chip and button remain visible. This is acceptable — the refetch updates the UI shortly after.

---

## QA Checklist

Manual verification steps using a seeded notification (use the admin panel or a direct MongoDB insert to create a `user`-type notification with `entities: [{entityType:'order', entityId:'<orderId>'}, {entityType:'document', entityId:'<docId>'}]` targeting a specific user):

- [ ] Section header strip is visible above the table with the title "Order Documents" on the left and (when applicable) the "Mark all as Read" button on the right, separated from the table by a visible vertical gap
- [ ] Open the order in the Documents tab as the **assigned user** — "New" chip visible inline in the **Received** column next to the date on the seeded document row; "Mark as Read" button visible in the new right-side column
- [ ] Open the order as a **different authenticated user** — "New" chip not visible; "Mark as Read" column cell is empty for that row; section header strip is still visible but without the "Mark all as Read" button
- [ ] Badge on the "Documents" tab header shows the correct unread count (same for both users — globally visible)
- [ ] Click "Mark as Read" on a row — chip and button disappear after SignalR event is received; badge count decrements
- [ ] Click "Mark all as Read" — all chips and row buttons disappear; the strip button becomes **disabled** (not hidden); badge reaches 0
- [ ] Disable `ORDER_DOCUMENT_NOTIFICATIONS` flag in admin panel — all badges, chips, per-row buttons, the new column, and the "Mark all as Read" button disappear without page reload; the section header strip remains with the title only
- [ ] Re-enable flag — all elements reappear on next navigation
- [ ] In a second browser session, add a new document to the order — badge on "Documents" tab updates in the first session without reload (`documentAdded` event)

---

## Out of Scope

- **Order History tab (D3-T3)** — separate Jira task, not planned here
- **Server-side assignee enforcement** on `PATCH /api/notifications/:id/read` and `PATCH /api/notifications/bulk-read` — accepted as UI-only gate per current architecture
- **Notification expiry logic changes** — expiry filter behavior is unchanged
- **Inbox/notification bell** — separate feature (D1/D2 of this epic), already implemented
