# MLID-2011 — Plan Adjustment After Data Model Review

**Date:** 2026-04-08
**Context:** After fixing PR #1049 review bugs and syncing the plan document, we compared epic tasks against the actual data model. A key requirement change on 2026-04-07 changed the notification type from `task` to `user`: only the user assigned to an order can view and acknowledge documents.

---

## Key Requirement Change (2026-04-07)

> "Only 'user' who is 'assigned to' the order can view / select ('Acknowledge') — Both at document level — At all documents level"

**Impact:**
- Notification type: **`user`** (not `task`)
- Targeting: **`targetUserId`** (order's assigned user, not `targetLocationId`)
- Read tracking: via **`NotificationRead` collection** (per-user read receipts, not inline `isRead`/`readBy`/`readAt`)

---

## Service Function Mapping Per Task

| Task | Action | Service Function |
|------|--------|-----------------|
| **D2-T2** — New document added | Create notification | `createNotification({ type: 'user', targetUserId: order.assignedTo, entities: [{entityType: 'order', entityId}, {entityType: 'document', entityId}], createdBy: 'system' })` |
| **D2-T1** — List notifications API | Get user's notifications | `getNotificationsForUser({ userId, locationIds: [], role })` |
| **D2-T1** — Mark read API route | Mark single notification read | `markUserNotificationAsRead(notificationId, userId)` |
| **D3-T1** — Order Tracker badge count | Count unread per order | `getCountForEntity({ entityType: 'order', entityId, userId, unreadOnly: true })` |
| **D3-T2** — "New" tag per document | Check if user read a notification | `isReadByUser(notificationId, userId)` |
| **D3-T2** — Single "Mark as Read" | Mark one notification read | `markUserNotificationAsRead(notificationId, userId)` |
| **D3-T2** — Bulk "Mark all as Read" | Mark all for an order | `bulkMarkUserNotificationsAsRead(notificationIds, userId)` |

---

## Resolved Gaps (from type change to `user`)

| # | Original Gap | Resolution |
|---|-------------|------------|
| 1 | D2-T1 said "TBD: document model changes" | **Resolved.** No document model changes needed. API routes use `markUserNotificationAsRead()` (not `markTaskAsRead()`). |
| 2 | Missing `bulkMarkTasksAsRead()` for bulk acknowledge | **Resolved.** `bulkMarkUserNotificationsAsRead()` already exists and works for `user` type. No new function needed. |
| 3 | Open question #4 — `acknowledged` flag on documents | **Resolved.** Not needed. Badge counts driven entirely from `Notification` + `NotificationRead` queries. |
| 4 | D2-T2 needed order's `locationId` for `targetLocationId` | **Resolved.** Now targets `targetUserId` (order's assigned user) — simpler, and `assignedTo` is already on the order. |

---

## Remaining Gaps

| # | Gap | Severity | When to Fix |
|---|-----|----------|-------------|
| 1 | `$elemMatch` bug in `getCountForEntity()` — dot-notation on `entities` array can cause cross-element false positives | High | Before D3-T1 (can fix now in PR #1049) |
