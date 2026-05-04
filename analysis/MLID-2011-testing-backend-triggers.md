# MLID-2011 — Backend Trigger Testing (scoping doc)

> **Status**: scoping only. The detailed test strategy will be written in a later session.
> **Scope**: verify the backend paths that create a notification and trigger a SignalR event.
> **Companion doc**: `MLID-2011-testing-signalr-integration.md` covers the broadcast/receive side.

## What "backend triggers" means here

The flows where a write to MongoDB (or an incoming webhook/job) results in:
1. A new row in `notifications`
2. A `documentAdded` broadcast over SignalR
3. (on read) A new row in `notificationreads` + a `notification:read` broadcast

The backend-trigger doc focuses on **(1) and the MongoDB side of (3)**. The broadcast side is covered in the SignalR doc.

## Dispatchers to exercise

| Trigger | Entry point | Orchestrator | Notes |
|---|---|---|---|
| Manual document upload (UI) | `POST /api/documents/split-pdf` (`apps/web/app/api/documents/split-pdf/route.ts`) | `notifyDocumentCreated()` | Two code paths in this route: full upload + split-PDF upload. Both must be exercised. |
| JotForm intake → WeInfuse | Background job: `apps/web/services/jobs/definitions/jotFormSubmission/weinfuseUploader.ts` | `notifyDocumentCreated()` | Needs to run via `/admin/jobs` UI (see memory note `reference_admin_jobs_ui`) or by receiving a Jotform webhook. |
| Mark notification as read (single) | `PATCH /api/notifications/[id]/read` | `markUserNotificationAsRead` | Writes to `notificationreads`. |
| Mark notifications as read (bulk) | `PATCH /api/notifications/bulk-read` | `bulkMarkUserNotificationsAsRead` | Writes to `notificationreads`. |

## Dispatcher logic — `notifyDocumentCreated()`

Lives in `apps/web/services/notifications/documentNotification.ts`.

Per-call behavior to verify:
1. Feature flag `ORDER_DOCUMENT_NOTIFICATIONS` is checked first — when `false`, function returns early and nothing is written or broadcast.
2. Resolves assigned orders for the patient via `we_infuse_id_IV`.
3. For each order with a non-empty `assignedTo`, inserts a `notifications` row with `type: 'user'`, `targetUserId: <assignedTo>`, `entities: [{ order }, { document }]`.
4. Errors inside the insert loop are caught and logged — function is **fire-and-forget** and never throws back to the caller.

## Areas the backend-trigger test strategy must cover (to fill out next time)

- [ ] Happy path: upload → exactly one notification per assigned order; correct `targetUserId`, `entities`, `createdBy`, `type`.
- [ ] Feature flag OFF: no writes, no logs of broadcast.
- [ ] Order without `assignedTo`: notification skipped (don't write one targeted at an empty string).
- [ ] Legacy email-shaped `assignedTo`: skipped (only 24-hex ObjectId-shaped IDs get a notification).
- [ ] Patient with no active orders: no writes, no error.
- [ ] JotForm path equivalence: upload via Jotform webhook must produce the same notification shape as a UI upload.
- [ ] Read endpoints idempotency: double PATCH of `/read` should not create duplicate `notificationreads` rows.
- [ ] Bulk read: partial list — any ID belonging to another user's notification must be ignored silently.
- [ ] PHI masking: notifications table/logs must not contain PHI fields.
- [ ] Mongoose-only rule: verify new code in `documentNotification.ts` uses Mongoose (per memory `feedback_mongoose_only`).

## Prerequisites checklist for exercising these paths manually

- VPN enabled (staging MongoDB)
- `ORDER_DOCUMENT_NOTIFICATIONS` flag ON in `/admin/feature-flags`
- A seeded test order with `assignedTo` set to a known user (see `MLID-2011-data-testing.md` — e.g., order `69cf9aee03a194bd1a65a199` / `05Gy`)
- A test patient with uploadable documents
- Logged in as a user whose `_id` is in `assignedTo` (to see the notification land on your badge)

## Useful Mongo Compass filters

**All `user`-type notifications (recently created):**
```json
{ "type": "user" }
```
Sort: `{ "createdAt": -1 }`

**Notifications for a specific assignee:**
```json
{ "type": "user", "targetUserId": "<user _id>" }
```

**Read receipts for a specific notification:**
```json
{ "notificationId": "<notification _id>" }
```
(collection: `notificationreads`)

## Cleanup after a test run

Same as section "Step 7 — Clean Up" in `MLID-2011-data-testing.md` — delete demo notifications and their read receipts.

## Open questions for the next session

1. Do we add automated integration tests for `notifyDocumentCreated()` that hit a real (ephemeral) Mongo, or do we rely on unit tests with mocks?
2. Do we want a job to backfill notifications from historical `documents` rows, or only forward-going?
3. How do we test the "order had no `assignedTo` at doc-create time, then got assigned later" race?
