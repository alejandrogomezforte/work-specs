# MLID-2355 — Add a notification counter for the Auth Review Tab (preplan / explanation)

- **Type:** Story
- **Parent epic:** MLID-2206 (Order Details v2 — Show Each Step Under Order Tracker)
- **Status:** In Development
- **Reporter:** Kammy Kuang (PO) · **Assignee:** Alejandro Gomez
- **Priority:** High

> This is a preplan (shared understanding), not the implementation plan. No code yet.

---

## 1. Objective

Show a numeric badge on the **Prior Auth tab** (the order-scoped Auth Review tab inside
Order Details v2) with the **number of prior-auth documents currently in the "Needs Review"
review status** for the currently open order.

### Acceptance criteria (from Jira)
1. The counter number must match the number of auth documents in "Needs Review" status.
2. After an auth doc is "Reviewed", the counter must decrease by 1.

---

## 2. Confirmed scope (the most important part)

The Jira comment thread deliberately narrowed this ticket:

- Alejandro (2026-06-12): asked whether this is just a count, or a full notification system
  (real-time updates + per-document "mark as read" — the complex, event-driven system built
  for Parker Baum). Put the ticket on hold pending specs.
- Kammy (2026-06-16): **"only calculate the total of documents with status in 'needs review'."**

So the confirmed scope is the **simple count**:
- NO notification engine, NO per-user read state, NO real-time event triggers, NO
  "mark as read" per document.
- AC #2 ("goes down by 1 after a review") is satisfied automatically: the badge is derived
  from the actual review status, so once a doc leaves "needs_review" it stops being counted.

### How AC #2 is satisfied without any live machinery (confirmed with PO)
When a prior-auth review status is changed, it is saved in the prior-auth detail page; after
save the app redirects back to the prior-auth list, which refetches. So the badge only needs
to be correct **on load** — no SignalR / real-time wiring required.

---

## 3. Where this lives in the code (investigation findings)

### 3.1 The "Auth Review tab" is the "Prior Auth" sub-tab
It is NOT one of the main order-detail tabs. The main tabs (Order Dashboard / Order History /
Documents) are defined in:
- `apps/web/app/orders-tracker/[category]/[orderId]/layout.tsx` → `tabItems`

The Prior Auth tab is a **sub-tab** inside the reviews panel, alongside Clinical Review and
Financial Review:
- `apps/web/app/orders-tracker/[category]/[orderId]/components/OrderDetailsReviewsPanel.tsx`
  → `reviewTabItems`, the item `{ label: 'Prior Auth', value: 'auth' }`
  (rendered only when `FeatureFlag.ORDER_DETAILS_AUTH_REVIEW` is enabled).

This `OrderDetailsReviewsPanel` is the component that owns the tab we badge.

> Route note: the user referred to `/orders-tracker/[category]/[id]/prior-auth`. The actual
> route segment is `auth-review` under `[category]/[orderId]`, and the tab label/value is
> "Prior Auth" / `'auth'`.

### 3.2 The badge MECHANISM to reuse (only the `@repo/ui` rendering, not the data)

We are reusing **only** the `@repo/ui` `Tabs` badge mechanism that the Documents tab uses to
draw the green pill. We are **not** reusing the Documents data (unread-notification count) and
we are **not** showing anything about unread documents on the Prior Auth tab. The Prior Auth
badge shows the **real count of prior-auth docs in `needs_review`** for this order.

The green "7" pill on the Documents tab (top-right nav of
`/orders-tracker/[category]/[orderId]/documents`) is drawn by three layers; only these layers
are the reusable mechanism:

**Layer 1 — attach a number as `badge` on the tab item** (`layout.tsx`):
```ts
{
  label: 'Documents',
  value: `${basePath}/documents`,
  ...(cond ? { badge: someNumber } : {}),   // <-- the only mechanism we reuse
}
```

**Layer 2 — visual render** (`packages/ui/src/components/Tabs/Tabs.tsx`):
`TabItem.badge?: number | string`; when present the label becomes the text plus a
`<TabBadge $size={size}>{tab.badge}</TabBadge>`.

**Layer 3 — the green pill style** (`packages/ui/src/components/Tabs/Tabs.styles.ts`,
`TabBadge` styled): rounded pill (`borderRadius: 999`), `backgroundColor:
theme.li.colors.interactive.primary` (the green), white text (`colors.text.inverse`).

**Why this transfers directly to the Prior Auth tab:** `OrderDetailsReviewsPanel` renders its
sub-tabs with the **same** `@repo/ui` `Tabs` component. So adding `badge: <needsReviewCount>`
to the `{ label: 'Prior Auth', value: 'auth' }` item yields an identical green pill, by
construction — no new UI component, no styling work.

**The number we pass is different from Documents.** The Documents tab feeds its badge the
notifications unread count from `useDocumentsTabNotifications` (server `unreadCount`,
SignalR-refreshed). The Prior Auth tab feeds its badge the plain count of this order's
prior-auth docs whose `reviewStatus === 'needs_review'` — no notifications collection, no
SignalR, no per-user read state. Only the `badge`/`<TabBadge>` rendering is shared.

### 3.3 The data source — "Needs Review" status
"Review Status" is **not** `Intake.status` (that is the intake lifecycle:
pending / processing / processed / failed / needsReview / skipped / completed).

The prior-auth review status is a separate field on the Intake document:
- `Intake.additionalInfo.priorAuthData.reviewStatus`
- Typed `PriorAuthReviewStatus = 'needs_review' | 'reviewed'` (`apps/web/types/priorAuth.ts`).

"Needs Review" = `reviewStatus === 'needs_review'`.

(Model: `apps/web/models/Intake.ts`; the flattened UI shape `PriorAuthLetter.reviewStatus`
is in `apps/web/types/priorAuth.ts`.)

### 3.4 The existing data fetch for this order's prior-auth letters
- Hook: `apps/web/app/orders-tracker/[category]/[orderId]/auth-review/_components/hooks/useOrderAuthReview.ts`
- It calls `GET /api/orders/new/${orderId}/auth-review` with `page`, `pageSize`, `sortOrder`,
  and filters (`letterType`, `payor`, `drug`, `reviewStatus`).
- Response shape includes `letters`, `total`, `totalPages`.
- The list is **server-paginated** (`paginationMode="server"` in `AuthReviewIndex.tsx`) and
  the results vary with the active UI filters.
- Note: the endpoint path hardcodes `new` even for other categories — existing behavior; the
  badge should reuse the same endpoint/path pattern, not invent a new one.

---

## 4. Count source — DECIDED: server `total` filtered to `needs_review`

**Decision (locked):** the badge value comes from the **server `total`**, filtered to
`needs_review` — NOT from counting the current in-memory page.

Why, given the prior-auth list is **server-paginated and filter-dependent**:

- Counting the in-memory `letters` array would:
  1. miss needs_review docs on other pages (page size 25/50/100), and
  2. break whenever the user applies a filter (e.g. filter by "Reviewed" → the array has
     zero needs_review rows even though some exist).
  Both contradict AC #1 ("counter must match the number of docs in needs review").

- The accurate, page/filter-independent count is already available from the server: the same
  list endpoint returns a `total`. We get it with a dedicated, tiny fetch:
  `GET /api/orders/new/${orderId}/auth-review?reviewStatus=needs_review&page=1&pageSize=1`
  and read `total`.

**Implementation of the decision:** add a small hook (working name
`useAuthReviewNeedsReviewCount(orderId)`) that `OrderDetailsReviewsPanel` calls to drive the
`'auth'` tab badge. It fetches the needs_review `total` from the existing endpoint —
page-independent and independent of whatever UI filters the user has applied. This keeps AC #1
correct and matches the established "separate hook feeds the tab badge" structure (the same
*structure* as `useDocumentsTabNotifications`, with the real needs_review count as the value).

> Why a panel-level hook (not reading `AuthReviewIndex`'s state): the badge sits on the tab in
> `OrderDetailsReviewsPanel`, which renders regardless of which sub-tab is active.
> `AuthReviewIndex` (and its `useOrderAuthReview`) only mounts when the 'auth' tab is active,
> so its `letters`/`total` are not available to the tab badge when another sub-tab is open.

---

## 5. Open decisions to confirm before writing the plan

1. **Count source** — DECIDED (see section 4): server `total` filtered to `needs_review`, via
   a small panel-level hook. No longer open.
2. **Feature flag** — DECIDED: reuse the existing `ORDER_DETAILS_AUTH_REVIEW` flag (the tab's
   own flag). No new flag — this is a plain count, not a separate feature. No longer open.
3. **Hide-when-zero** — DECIDED: yes, hide the badge when the needs_review count is zero
   (mirrors the Documents badge `count > 0 ? { badge } : {}`). No longer open.
4. **Backend reuse (verification, not a decision)** — confirm during planning that the
   existing `GET .../auth-review?reviewStatus=needs_review` endpoint returns a `total` that
   already counts only this order's letters (it does, per `useOrderAuthReview`), so NO new API
   route is needed. Pure verification step.

---

## 6. Rough implementation sketch (for the eventual plan, not committing yet)

Subject to the decisions above; following TDD (tests first):

1. **Hook** `useAuthReviewNeedsReviewCount(orderId)` (new, next to or under
   `auth-review/_components/hooks/`): fetches the existing list endpoint with
   `reviewStatus=needs_review&page=1&pageSize=1`, returns `{ count }` from `total`.
   Returns 0 when the auth-review flag is off / no orderId / on error (silent, like the
   Documents hook).
2. **Wire the badge** in `OrderDetailsReviewsPanel.tsx`: call the hook, add
   `...(count > 0 ? { badge: count } : {})` to the `{ label: 'Prior Auth', value: 'auth' }`
   tab item.
3. **Tests:** hook unit test (count from total, zero states, error → 0); panel test (badge
   shown with count, hidden at zero, hidden when flag off).
4. **No new API route**, **no new collection/field**, **no SignalR**, **no env vars**.

---

## 7. Out of scope (explicitly)

- Real-time / live badge updates (SignalR). Confirmed unnecessary — badge is correct on load.
- Per-user "mark as read" / per-document read state.
- Any notification engine or event triggers.
- Changes to the prior-auth detail/save flow.
