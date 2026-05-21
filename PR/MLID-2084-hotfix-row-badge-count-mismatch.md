# [MLID-2084] Hotfix: Unify order document badge counts across orders-tracker table and Documents tab

## Summary

- Detected on staging (QA Failure #1 reported by Olha, 2026-05-18): the unread-notification badge on the orders-tracker table row disagreed with the badge on the same order's Documents tab. Example: order `6a078a236de202e6c06d1037` showed **6 on the table** and **4 on the Documents tab** for the same user.
- Root cause: the two surfaces applied **different definitions of "unread"**. The table called `getCountForEntity` which keyed the read-check on `notification.targetUserId` (the original recipient). The Documents tab fetched the notification list and computed `unreadCount` client-side from `isReadByCurrentUser`, which keyed on the **current viewer's** session id. For order `6a07…1037`, all 6 notifications target user `67fe9b98…` (the assignee), and 2 NotificationRead rows exist — both written by a different user (`695c25a5…`, the viewer). The table saw 0 reads against the target → 6 unread. The tab saw 2 reads against the viewer → 4 unread.
- After this hotfix both surfaces share the same source of truth (`getCountForEntity`) with `userReadScope: 'any'` — any reader's NotificationRead row marks the notification as acknowledged for the team, regardless of who the original target was. This was required to handle the order-reassignment case (assignee X → Y; Y's read should drop the badge for everyone).

**Branch:** `hotfix/MLID-2084-row-badge-count-mismatch` → `develop`

**Parent Jira:** MLID-2084 — original badge story already shipped to develop; this is a follow-up hotfix on top of that work, tracked back to the same Jira ID. No separate ticket was filed.

---

## Why the two surfaces diverged

```
Same notification, two questions:

  Table badge:
    "Has the NOTIFICATION'S TARGET USER read this?"
    → Global answer, same for every viewer.

  Documents tab badge (pre-fix):
    "Has the CURRENT VIEWER personally read this?"
    → Different answer per viewer.
```

For order `6a07…1037`:

```
notifications (6 docs)        notificationreads (2 rows)
  targetUserId: 67fe9b98…       userId: 695c25a5…  (a different user)
  isRead: false                 userId: 695c25a5…  (a different user)

  Table   asks: target's reads? → 0 found → count = 6   ❌
  Tab     asks: viewer's reads? → 2 found → count = 4   ❌
  After:  asks: any reads?      → 2 found → count = 4   ✅ (both surfaces)
```

The PO (Parker Baum, 2026-05-18) confirmed each notification is per-`(order, document)` pair, so marking a doc read in Order A still doesn't touch Order B's notification — the new "any user" semantics only collapse the *who-read-it-counts-for-the-team* question, not the per-order acknowledgement model.

---

## Changes

| File | Change |
|------|--------|
| `apps/web/services/mongodb/notification.ts` | Added `userReadScope: 'target' \| 'any'` parameter to `getCountForEntity` (default `'target'`, preserves all unrelated callers). Extracted the `userUnread` `$lookup` body into `buildUserUnreadLookup(userReadScope)`. When `'any'`, the inner `$match` only compares `notificationId`; the `userId == $$targetUserId` clause is dropped. The `taskUnread` branch and the `activeExpiryFilter` are untouched. |
| `apps/web/services/mongodb/notification.test.ts` | +5 tests covering: explicit `'target'` default, `'any'` returns server-side count, `'any'` drops the `userId` clause from the `$lookup`, `'any'` still respects `activeExpiryFilter`, and `'any'` does not affect the `taskUnread` branch. |
| `apps/web/app/api/notifications/counts/route.ts` | The orders-tracker table data source now passes `userReadScope: 'any'`. |
| `apps/web/app/api/notifications/counts/route.test.ts` | Added a new test asserting `userReadScope: 'any'` is passed; updated the existing full-call-shape assertion. |
| `apps/web/app/api/notifications/by-order/[orderId]/route.ts` | Added a `getCountForEntity({ entityType:'order', entityId, unreadOnly:true, userReadScope:'any' })` call and included the result as a top-level `unreadCount` in the JSON response, alongside the existing per-notification summaries. |
| `apps/web/app/api/notifications/by-order/[orderId]/route.test.ts` | Added `getCountForEntity` to the module mock; +3 tests covering response shape, call params, and the zero-count case. |
| `apps/web/app/orders-tracker/[category]/[orderId]/documents/useDocumentsTabNotifications.ts` | Added `ByOrderApiResponse` local type with `{ notifications, unreadCount }`. Stores the server-provided `serverUnreadCount` in state and returns it as `unreadCount`. Removed the client-side `notifications.filter(n => !n.isReadByCurrentUser).length` derivation. |
| `apps/web/app/orders-tracker/[category]/[orderId]/documents/useDocumentsTabNotifications.test.tsx` | Updated all `fetch` mocks to include `unreadCount` in the response. Updated 2 existing tests whose mechanism was the now-removed client-side filter (intent preserved — the assertion that the tab badge is global is now enforced server-side). Added 1 new test (`'reads unreadCount from the server response rather than computing it client-side'`) that mocks `unreadCount: 99` with 1 unread item and asserts the hook returns `99` — proves the server value wins. |

### NOT touched

- **Per-row indicators on each document row** (`byDocumentId`, `notificationIds`, `unreadNotificationIdsForCurrentUser`, `hasUnreadForCurrentUser`) stay per-viewer — they drive the row-level UI ("have I personally read this row?"). Out of scope; PO can revisit later if needed.
- **`taskUnread` branch of `getCountForEntity`** — task-type notifications still use the single `isRead` field on the document. Only the `userUnread` branch changed.
- **`markUserNotificationAsRead`** — already handles the reassignment scenario via `isCallerOrderAssignee` (notification.ts:328-357). The mark-as-read auth model didn't need to change.
- **Other notification entity types** (`role`, `location`) — out of scope; the `userReadScope` API is forward-compatible but only `'order'` callers opt in today.
- **The other QA-failure issues** from Olha's 2026-05-18 comment (counter not decreasing on order unlink, no counter for maintenance orders, no counter for "Orange Park" location) — separate tickets.

---

## Verification

| Check | Result |
|-------|--------|
| `npx jest` (scoped, 4 suites) | 106/106 passed (+9 new, 2 updated) |
| `npx jest` (full apps/web suite) | 15802/15805 passed; 2 pre-existing failures unrelated to this hotfix (`services/jobs/pulse.test.ts`, `app/api/jobs/analytics-process-intake-overview/export/route.test.ts`) |
| `npm run types:check` (apps/web) | Clean |
| `npx eslint` on 8 changed files | Only 2 pre-existing `no-use-before-define` warnings on `activeExpiryFilter` (lines 44 and 234) — both exist on `develop`. No new lint debt introduced by this hotfix. |
| `npx prettier --check` on 8 changed files | All clean (unchanged) |
| Branch/line coverage on changed source files | 95.45% – 100% (lowest: `useDocumentsTabNotifications.ts`) |

### RED-before-GREEN evidence

Phase A (service layer) — added 5 new tests; only the one that meaningfully exercises the new code path failed pre-fix:

```
× should NOT count a user-type notification as unread when ANY notificationreads
  row exists regardless of userId (userReadScope: any)

  Expected: not ObjectContaining {"targetUserId": Anything}
  Received:     {"notifId": "$_id", "targetUserId": "$targetUserId"}
```

Phase B (counts route) — 2 failures pre-fix (1 new + 1 updated assertion), both citing the missing `userReadScope: 'any'`.

Phase C (by-order route) — 3 failures pre-fix; the response had no `unreadCount` field and `getCountForEntity` was not called.

Phase D (hook) — 1 failure pre-fix:

```
× reads unreadCount from the server response rather than computing it client-side
  Expected: 99
  Received: 1
```

All went green after the corresponding implementation step.

---

## Manual verification (already completed on local by the engineer before commit)

- [x] Logged in as a user who is NOT the assignee of an order with unread document notifications. Noted the orders-tracker table badge count, opened the Documents tab — the two badges matched exactly.
- [x] Clicked a document to mark it read. Both badges decremented by 1 after the SignalR refetch.
- [x] Logged in as the assignee — same counts visible (global, not per-viewer).
- [x] For repro order `6a07…1037`: table badge dropped from 6 → 4 as predicted (the 2 NotificationRead rows previously written by another user now correctly count for the team).

---

## Test Plan (staging)

- [ ] On `/orders-tracker/new`, find an order whose unread badge had previously disagreed with its Documents tab badge. Both should now show the same number.
- [ ] Open the Documents tab → click an unread document → both the tab badge AND the orders-tracker table badge for that order should decrement by 1 (SignalR-driven).
- [ ] Sanity: logging in as different users should produce the same count for the same order (the count is global, not per-viewer).
- [ ] Sanity: an order with **zero** unread document notifications should still show no badge on either surface.
- [ ] Sanity: marking a document read on Order A should NOT touch the badge on Order B even if both orders share the same patient and document (per-order acknowledgement still holds — separate `notification` rows per `(order, document)` pair).
- [ ] Regression: the per-row "unread" visual indicators inside the Documents tab should still reflect *the current viewer's* personal read state (unchanged).
- [ ] Regression: task-type notifications elsewhere in the app should be unaffected (only the `userUnread` `$lookup` branch changed).
