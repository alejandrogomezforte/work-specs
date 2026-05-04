# [MLID-2011] Hotfix: Enforce target-user ownership on mark-as-read endpoints

## Summary

- The single (`PATCH /api/notifications/:id/read`) and bulk (`PATCH /api/notifications/bulk-read`) endpoints were writing a `NotificationRead` record for `session.user.id` without verifying the caller was the notification's `targetUserId`. Any authenticated user who guessed or discovered a notification id could create a read receipt against it, allowing read-state manipulation and existence probing via response timing.
- The UI-side permission gate (`hasUnreadForCurrentUser`) was **not mirrored at the API**, so the protection was bypassable by anyone with a valid session and a notification id.
- Fix: push the ownership check into the service layer. Both functions now verify ownership against the `Notification` collection before any write to `notificationreads`, and return a value the route uses to decide its response and broadcast behavior.

**Branch:** `hotfix/MLID-2011-notification-read-authz` → `epic/MLID-2011-order-document-notifications`

**Source:** Code-review bot finding ("Notification read state can be written by any authenticated user").

---

## Changes

### Service layer

| File | Change |
|------|--------|
| `apps/web/services/mongodb/notification.ts` | `markUserNotificationAsRead` now returns `Promise<boolean>`. Looks up `Notification.findOne({ _id, targetUserId: userId })` first; returns `false` (no write) when caller is not the target OR the id doesn't exist (same shape — prevents existence probing). Idempotent re-mark (duplicate-key 11000) still returns `true`. |
| `apps/web/services/mongodb/notification.ts` | `bulkMarkUserNotificationsAsRead` now returns `Promise<string[]>`. Filters input ids via `Notification.find({ _id: { $in }, targetUserId })` before any write; insertMany only the owned subset. Returns the subset that was actually marked. |
| `apps/web/services/mongodb/notification.test.ts` | 11 service tests rewritten/added covering: ownership match, ownership failure (no write, no leak), non-existent id (same shape as not-owned), idempotent duplicate, bulk subset filtering, and bulk no-op when caller owns nothing. |

### Route layer

| File | Change |
|------|--------|
| `apps/web/app/api/notifications/[id]/read/route.ts` | Maps service `false` → **404** (deliberately not 403 — same status whether the id exists or not, so callers can't probe). Skips the `notification:read` broadcast on the unauthorized branch. |
| `apps/web/app/api/notifications/[id]/read/route.test.ts` | Added "not target user → 404 + no broadcast" cases. |
| `apps/web/app/api/notifications/bulk-read/route.ts` | Broadcasts only the owned subset returned by the service; skips the broadcast entirely when nothing was marked. Stays **200** on no-op (clients legitimately submit mixed lists, e.g. background tab refresh). |
| `apps/web/app/api/notifications/bulk-read/route.test.ts` | Added "filters subset" + "skips broadcast on empty" cases. |

### NOT touched

- Client UI / hooks — the existing UI gate was correct for legitimate users; only forged or malicious requests now hit the new 404 / no-op branches. No UX regression possible.
- The notification-read broadcast payload shape — preserved as `{ notificationId(s), userId }` for legitimate cases; clients re-fetch via REST.

---

## Why 404 (not 403) on the single endpoint

Returning a different status for "exists but not owned" vs "does not exist" lets a caller probe whether arbitrary ids exist by issuing PATCH requests. Returning **404 in both cases** keeps the unauthorized branch indistinguishable from the not-found branch, closing the existence-leak side channel called out by the bot.

## Why 200 (not 404) on the bulk endpoint

The bulk endpoint legitimately receives **mixed lists** (e.g. a tab refresh that includes ids the caller no longer owns after a permission change). Returning 404 would break the happy path. Instead, the service silently filters; the response stays 200 and the broadcast carries only the owned subset.

---

## Verification

| Check | Result |
|-------|--------|
| `notification.test.ts` (service) | 44/44 passed |
| `[id]/read/route.test.ts` | 11/11 passed |
| `bulk-read/route.test.ts` | 11/11 passed |
| TypeScript (`tsc --noEmit`) | Clean |
| Backend-only diff — no UI behavior change for legitimate users | n/a |

---

## Test Plan

- [ ] As the assigned user of a notification, click "Mark as read" → 200, badge clears, broadcast fires (regression: legitimate happy path unchanged).
- [ ] As the assigned user, bulk-mark a list of owned ids → 200, broadcast contains exactly those ids.
- [ ] Forge a `PATCH /api/notifications/:id/read` request from a different authenticated session for someone else's notification id → expect **404**, no `notificationreads` row written, no broadcast.
- [ ] Forge a bulk request with a mix of owned + foreign ids → expect 200, only owned ids appear in the broadcast payload, only owned ids appear as new `notificationreads` rows.
- [ ] Forge a bulk request with **only** foreign ids → expect 200, **no** broadcast emitted, **no** rows written.
- [ ] Verify a non-existent id returns the same 404 as a not-owned id (no timing or status-code distinction visible to the caller).
