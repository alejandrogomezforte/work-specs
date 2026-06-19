# MLID-2447 — Maintenance Orders Table: Unread Notification Badge

## Task Reference

- **Jira**: [MLID-2447](https://localinfusion.atlassian.net/browse/MLID-2447)
- **Story Points**: 2
- **Branch**: `feature/MLID-2447-maintenance-orders-table-badge`
- **Base Branch**: `epic/MLID-2417-document-notifications-maintenance`
- **Epic Plan**: `docs/agomez/plans/MLID-2417-plan-progress.md` (D1-T1 section)
- **Status**: Done — merged into epic (local, `--no-ff` `230529ce3`) 2026-06-19

---

## Summary

Add a per-row unread notification badge to the `/orders/maintenance` orders-tracker table, mirroring what already exists on the `/orders/new` table. The badge is driven by the existing `useOrderNotificationCounts` hook and `POST /api/notifications/counts` endpoint — no new back-end work. The task also extracts the badge JSX into a shared `OrderNotificationBadge` component so neither column factory duplicates it.

---

## Codebase Analysis

**Three active touch points identified from source:**

1. `apps/web/app/orders-tracker/[category]/page.tsx`
   - `isNotificationsEnabled` already reads `FeatureFlag.ORDER_DOCUMENT_NOTIFICATIONS`.
   - The `visibleOrderIds` useMemo early-returns `[]` when `tab !== 'new'` — the only change needed is adding the `tab === 'maintenance'` branch that maps `maintenanceOrders.data`.
   - The `getMaintenanceOrdersColumns(...)` call passes no `unreadCountsByOrderId` today. Must add the same gated prop pattern used by the adjacent `getNewOrdersColumns(...)` call.
   - `unreadCountsByOrderId` is already in the columns `useMemo` dependency array — no dep-array change needed.
   - `maintenanceOrders.data` is already destructured from `useOrdersTracker()` — available to the `visibleOrderIds` useMemo.

2. `apps/web/app/orders-tracker/Orders/NewOrders.tsx`
   - `getNewOrdersColumns` already accepts `unreadCountsByOrderId?: Record<string, number>`.
   - Badge JSX lives in the `displayId` column `renderCell`: reads `unreadCountsByOrderId?.[params.row._id]`, renders a native `<a>` (not Next `<Link>`) pointing to `/orders-tracker/new/${params.row._id}/documents`, with `data-testid="notification-badge-${params.row._id}"`.
   - The `<div onMouseDown={(e) => e.stopPropagation()}>` wrapper must be preserved.
   - After extraction, this factory delegates to `OrderNotificationBadge`.

3. `apps/web/app/orders-tracker/Orders/MaintenanceOrders.tsx`
   - `getMaintenanceOrdersColumns` has no `unreadCountsByOrderId` param.
   - The `displayId` column `renderCell` renders the order-ID link but has no badge block.
   - Deep-link target must be `/orders-tracker/maintenance/${params.row._id}/documents`.
   - After this task, the factory accepts `unreadCountsByOrderId?: Record<string, number>` and renders `OrderNotificationBadge`.

**No `OrderNotificationBadge` component exists** — `apps/web/app/orders-tracker/Orders/OrderNotificationBadge*` returned no matches.

**Existing test files:**
- `apps/web/app/orders-tracker/Orders/NewOrders.test.tsx` — has a `Notification badge` describe block with 4 cases (no-prop absent, count-0, global-visibility regardless of assignee, correct count). These must continue to pass after extraction.
- `apps/web/app/orders-tracker/Orders/MaintenanceOrders.test.tsx` — has no `Notification badge` describe block. A mirrored describe block must be added.

**`useOrderNotificationCounts` hook** (`apps/web/app/orders-tracker/hooks/useOrderNotificationCounts.ts`): accepts `orderIds: string[]`, returns `Record<string, number>`. Listens to `documentAdded`, `documentUnlinked`, `notification:read`, `order:reassigned`. Category-agnostic — no change needed.

**Why `OrderNotificationBadge` is app-local, not in `@repo/ui`:** The badge links to an app route (`/orders-tracker/[category]/[id]/documents`) and is only meaningful inside the DataGrid cell context where a native `<a>` is required to avoid triggering the row click. It has no standalone reuse value in the design system.

---

## Implementation Steps

### Step 1 (RED) — Write failing `OrderNotificationBadge` component tests

**Commit message:** `[MLID-2447] - test(orders-tracker): add failing tests for OrderNotificationBadge component`

**Files:**
- Create: `apps/web/app/orders-tracker/Orders/OrderNotificationBadge.test.tsx`

**What:** Write tests that import `OrderNotificationBadge` from `./OrderNotificationBadge` (which does not exist yet). Tests verify the component's exact rendered output and prop contract. Tests will fail with a module-not-found error.

**Test cases to include (all inside `describe('OrderNotificationBadge')`):**

```
it('should not render anything when count is 0')
  — render <OrderNotificationBadge orderId="ord-1" href="/test" count={0} />
  — expect queryByTestId('notification-badge-ord-1') to not be in the document

it('should not render anything when count is negative')
  — render <OrderNotificationBadge orderId="ord-1" href="/test" count={-1} />
  — expect queryByTestId('notification-badge-ord-1') to not be in the document

it('should render badge with correct data-testid when count is positive')
  — render <OrderNotificationBadge orderId="ord-42" href="/some-path" count={5} />
  — expect getByTestId('notification-badge-ord-42') to be in the document

it('should display the count inside the badge')
  — render <OrderNotificationBadge orderId="ord-42" href="/some-path" count={5} />
  — expect getByTestId('notification-badge-ord-42') to have text content '5'

it('should link to the provided href')
  — render <OrderNotificationBadge orderId="ord-42" href="/orders-tracker/new/ord-42/documents" count={3} />
  — expect getByTestId('notification-badge-ord-42') to have attribute href '/orders-tracker/new/ord-42/documents'

it('should render as a native anchor element (not Next Link) to avoid DataGrid row click')
  — render <OrderNotificationBadge orderId="ord-99" href="/test" count={1} />
  — expect getByTestId('notification-badge-ord-99').tagName to equal 'A'
```

Run `npm run test -- OrderNotificationBadge.test` from `apps/web/`. Expected: all tests fail (module not found).

---

### Step 2 (GREEN) — Create `OrderNotificationBadge` component

**Commit message:** `[MLID-2447] - feat(orders-tracker): add shared OrderNotificationBadge component`

**Files:**
- Create: `apps/web/app/orders-tracker/Orders/OrderNotificationBadge.tsx`

**What:** Create the component with exactly the prop signature and visual implementation extracted from `NewOrders.tsx`. The component renders nothing when `count <= 0`.

**Prop type:**
```typescript
type OrderNotificationBadgeProps = {
  orderId: string;
  href: string;
  count: number;
};
```

**Implementation:** Extract the badge `<a>` block from the `displayId` `renderCell` in `NewOrders.tsx`. The component receives `orderId`, `href`, and `count` directly — no dependency on `unreadCountsByOrderId`. Preserves `data-testid={`notification-badge-${orderId}`}`, `DescriptionOutlinedIcon`, and the inline style. No named barrel export needed because it is consumed only within the same `Orders/` directory.

Run `npm run test -- OrderNotificationBadge.test` from `apps/web/`. Expected: all 6 tests pass.

---

### Step 3 (REFACTOR) — Refactor `NewOrders.tsx` to use `OrderNotificationBadge`; regression-verify existing badge tests

**Commit message:** `[MLID-2447] - refactor(orders-tracker): use OrderNotificationBadge in getNewOrdersColumns`

**Files:**
- Modify: `apps/web/app/orders-tracker/Orders/NewOrders.tsx`

**What:** Replace the inline badge block in `getNewOrdersColumns` `displayId` `renderCell` with `<OrderNotificationBadge orderId={params.row._id} href={`/orders-tracker/new/${params.row._id}/documents`} count={unreadCountsByOrderId?.[params.row._id] ?? 0} />`. Remove the `unreadCount`/`showBadge` local variables from that scope. Import `OrderNotificationBadge` from `./OrderNotificationBadge`.

Run `npm run test -- NewOrders.test` from `apps/web/`. Expected: all 4 existing `Notification badge` cases still pass (rendered output is identical, `data-testid` is preserved via the shared component). No behavior change.

---

### Step 4 (RED) — Write failing badge tests for `getMaintenanceOrdersColumns`

**Commit message:** `[MLID-2447] - test(orders-tracker): add failing notification badge tests for maintenance orders column factory`

**Files:**
- Modify: `apps/web/app/orders-tracker/Orders/MaintenanceOrders.test.tsx`

**What:** Add a `Notification badge` describe block that mirrors the 4 cases in `NewOrders.test.tsx`, adapted for the maintenance factory:

```
describe('Notification badge') {
  it('should not render badge when unreadCountsByOrderId is not provided')
    — columns = getMaintenanceOrdersColumns({ onDeleteClick: () => {} })
    — render displayId renderCell for row with _id 'order-1'
    — expect queryByTestId('notification-badge-order-1') not to be in document

  it('should not render badge when unread count is 0')
    — columns = getMaintenanceOrdersColumns({ ..., unreadCountsByOrderId: { 'order-1': 0 } })
    — render displayId renderCell for row with _id 'order-1'
    — expect queryByTestId('notification-badge-order-1') not to be in document

  it('should render badge regardless of who the order is assigned to')
    — columns = getMaintenanceOrdersColumns({ ..., unreadCountsByOrderId: { 'order-1': 3 } })
    — row has assignedTo: 'some-other-user'
    — render displayId renderCell for row with _id 'order-1'
    — expect getByTestId('notification-badge-order-1') to be in document
    — expect badge to have text content '3'

  it('should render badge with correct count when unread > 0')
    — columns = getMaintenanceOrdersColumns({ ..., unreadCountsByOrderId: { 'order-1': 3 } })
    — render displayId renderCell for row with _id 'order-1'
    — expect getByTestId('notification-badge-order-1') to have text content '3'

  it('should link badge to the maintenance documents route')
    — columns = getMaintenanceOrdersColumns({ ..., unreadCountsByOrderId: { 'order-m1': 2 } })
    — render displayId renderCell for row with _id 'order-m1'
    — expect getByTestId('notification-badge-order-m1') to have attribute href '/orders-tracker/maintenance/order-m1/documents'
}
```

Run `npm run test -- MaintenanceOrders.test` from `apps/web/`. Expected: 5 new tests fail because `getMaintenanceOrdersColumns` does not accept `unreadCountsByOrderId` and renders no badge.

---

### Step 5 (GREEN) — Add `unreadCountsByOrderId` prop and badge to `getMaintenanceOrdersColumns`

**Commit message:** `[MLID-2447] - feat(orders-tracker): add unread notification badge to maintenance orders column factory`

**Files:**
- Modify: `apps/web/app/orders-tracker/Orders/MaintenanceOrders.tsx`

**What:**

1. Add `unreadCountsByOrderId?: Record<string, number>` to the params type of `getMaintenanceOrdersColumns` (alongside the existing props, mirroring `getNewOrdersColumns`).

2. Import `OrderNotificationBadge` from `./OrderNotificationBadge`.

3. In the `displayId` column `renderCell`, add `OrderNotificationBadge` after the `<Link>` element, inside the existing `<Layout direction="row" gap={2} align="center">`. Pass `orderId={params.row._id}`, `href={`/orders-tracker/maintenance/${params.row._id}/documents`}`, `count={unreadCountsByOrderId?.[params.row._id] ?? 0}`.

4. Import `DescriptionOutlinedIcon` is NOT needed in `MaintenanceOrders.tsx` — it is encapsulated inside `OrderNotificationBadge`.

Run `npm run test -- MaintenanceOrders.test` from `apps/web/`. Expected: all 5 new badge tests pass; all pre-existing tests continue to pass.

---

### Step 6 (GREEN) — Wire `visibleOrderIds` maintenance branch and pass `unreadCountsByOrderId` to `getMaintenanceOrdersColumns` in `page.tsx`

**Commit message:** `[MLID-2447] - feat(orders-tracker): wire maintenance order IDs into notification counts and pass badge prop`

**Files:**
- Modify: `apps/web/app/orders-tracker/[category]/page.tsx`

**What:**

1. Extend the `visibleOrderIds` useMemo to cover maintenance:

   Current:
   ```typescript
   const visibleOrderIds = useMemo(() => {
     if (tab !== 'new' || !isNotificationsEnabled) return [];
     return (newOrders.data ?? []).map((o) => o._id);
   }, [tab, isNotificationsEnabled, newOrders.data]);
   ```

   Replace with:
   ```typescript
   const visibleOrderIds = useMemo(() => {
     if (!isNotificationsEnabled) return [];
     if (tab === 'new') return (newOrders.data ?? []).map((o) => o._id);
     if (tab === 'maintenance') return (maintenanceOrders.data ?? []).map((o) => o._id);
     return [];
   }, [tab, isNotificationsEnabled, newOrders.data, maintenanceOrders.data]);
   ```

2. In the `getMaintenanceOrdersColumns(...)` call, add `unreadCountsByOrderId: isNotificationsEnabled ? unreadCountsByOrderId : undefined` alongside the other props, matching the pattern already used for the adjacent `getNewOrdersColumns(...)` call.

No dep-array change is needed in the columns useMemo because `unreadCountsByOrderId` is already in the dependency array.

**Note:** There is no `page.test.tsx`. The `visibleOrderIds` logic change is covered indirectly by the column-factory tests (which test what the factory renders when given a `unreadCountsByOrderId` map) and verified directly via browser. This is the right trade-off: the page is a `'use client'` component with heavy Next.js / session dependencies, writing a unit test for the useMemo alone would require large mock scaffolding for marginal benefit over the existing column-factory coverage.

Run `npm run test -- NewOrders.test MaintenanceOrders.test OrderNotificationBadge.test` from `apps/web/`. Expected: all tests pass.

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/app/orders-tracker/Orders/OrderNotificationBadge.tsx` | Create | Shared badge component used by both column factories |
| `apps/web/app/orders-tracker/Orders/OrderNotificationBadge.test.tsx` | Create | Unit tests for the shared badge component (6 cases) |
| `apps/web/app/orders-tracker/Orders/NewOrders.tsx` | Modify | Replace inline badge block with `<OrderNotificationBadge>` |
| `apps/web/app/orders-tracker/Orders/MaintenanceOrders.tsx` | Modify | Add `unreadCountsByOrderId` prop; render `<OrderNotificationBadge>` in `displayId` cell |
| `apps/web/app/orders-tracker/Orders/MaintenanceOrders.test.tsx` | Modify | Add `Notification badge` describe block (5 cases) |
| `apps/web/app/orders-tracker/[category]/page.tsx` | Modify | Extend `visibleOrderIds` useMemo to cover `tab === 'maintenance'`; pass `unreadCountsByOrderId` to `getMaintenanceOrdersColumns` |

---

## Testing Strategy

**Unit tests — column factory level:**
- `MaintenanceOrders.test.tsx` new describe block (5 cases): no badge when prop absent, no badge when count 0, badge renders regardless of assignee, badge shows correct count, badge links to `/orders-tracker/maintenance/[id]/documents`.
- `NewOrders.test.tsx` existing 4 badge cases: must still pass after extraction into shared component.

**Unit tests — shared component level:**
- `OrderNotificationBadge.test.tsx` (6 cases): count 0 renders nothing, count negative renders nothing, positive count renders badge with correct `data-testid`, badge displays the count, badge links to provided `href`, element tag is `A` (native anchor, not Next Link).

**No API or hook tests required** — `useOrderNotificationCounts` is category-agnostic and already tested; no new endpoints introduced.

**Manual browser verification** (after `npm run dev:web`, flag ON):
- Navigate to `/orders/maintenance`. Rows with unread documents show the description icon + count badge. Rows with zero unread documents show no badge.
- Navigate to `/orders/new`. Badge behavior is unchanged.
- Receive a new document upload notification (or toggle a read state). Both tables' badges refresh without page reload via existing SignalR events.
- Turn `ORDER_DOCUMENT_NOTIFICATIONS` OFF in the admin panel. Badges disappear from both tables.

**Regression gate:** Run `npm run test -- --testPathPattern="NewOrders|MaintenanceOrders|OrderNotificationBadge"` from `apps/web/` after step 6. All tests must pass before requesting review.

---

## Security Considerations

**Input validation:** Not applicable — the component renders a count derived from an API call already gated by session auth on the server. No user input enters the badge.

**Authorization:** The `POST /api/notifications/counts` endpoint that backs `useOrderNotificationCounts` already performs session auth. The badge is visible to all authenticated users for any order (global-visibility product decision from 2026-04-14 — the `assignedTo` field does not gate badge rendering, matching the behavior in `NewOrders.tsx`).

**Feature flag gating:** `ORDER_DOCUMENT_NOTIFICATIONS` gates at both levels:
- Server: the `/api/notifications/counts` API route checks the flag and returns early when it is off.
- Client: `isNotificationsEnabled` is checked in the `visibleOrderIds` useMemo (returning `[]` when off) and in the `unreadCountsByOrderId` prop passed to both column factories (`undefined` when off), so no badge is ever constructed on the client side when the flag is off.

**PHI handling:** No patient data flows through the badge. The badge renders only an order ID (used as a DOM testid) and a numeric count. No PHI exposure.

---

## Open Questions

None. Architecture is settled; all integration points verified against current source.
