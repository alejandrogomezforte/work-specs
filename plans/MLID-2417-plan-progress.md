# MLID-2417 — Orders: Document Notifications V2 (Maintenance & Order Linking)

## Epic Workflow

This document follows the [SW Workflow](../dx/sw-workflow.md). For each sub-task, run the workflow cycle once and update this document to track progress. Read this document first to understand overall context before starting each cycle.

For the git branch strategy, see [epic-git-branch-strategy.md](../dx/epic-git-branch-strategy.md).

---

## Current State (updated 2026-06-30) — **MERGED TO DEVELOP / DONE**

**Status: DONE.** The epic → develop PR was approved and merged. All of D1–D5 is on `develop`. Two review-driven follow-ups landed on the epic before merge: the `OrderRef._id` → `$toString` projection fix (`getOrdersByPatient` lean path) plus its two pipeline-shape test corrections, and a reconciling merge of a teammate's develop-sync + MLID-2659 work. Two other Claude-review notes were evaluated and intentionally skipped (the `LinkOrdersModal` silent-`'new'`-category fallback — a real but lower-priority regression with an incorrect suggested fix; and the `UpdateOrderForm` stale-category-on-deselect — cosmetic form-state hygiene only).

**Done:**

- **D0** — all four new-orders retrofits are on `develop`: MLID-2391 (+ initial-assignment hotfix), MLID-2413, MLID-2439 (parts 1 and 2).
- **D1-T1 (MLID-2447)** — maintenance orders table unread badge — merged into the epic branch (local, `--no-ff`).
- **D2-T1 (MLID-2448)** — maintenance order-detail **Documents tab** notifications (tab badge, "New" chip, mark-as-read, mark-all-as-read) — merged into the epic branch (local, `--no-ff`) 2026-06-19. Verification + maintenance test coverage only; no production code change (the shared `[category]` route had no category gate). Manually verified on maintenance order `0yn` across two sessions: assignee sees the controls, a non-target sees the same badge count but no per-row controls, mark-as-read decremented the badge live via SignalR, and feature-flag OFF hid everything. Branch `feature/MLID-2448-maintenance-documents-tab-notifications` is kept.
- **D3-T1 (MLID-2449)** — maintenance order-detail **History tab** document log entries — merged into the epic branch (`--no-ff` `6278462c9`) 2026-06-22. Verification + maintenance test coverage only; no production code change — the History tab (`order-history/page.tsx`) reads only `orderId` from `useParams` (never `[category]`), and the audit-log stack + `documents` formatter are category-agnostic. Manually verified on maintenance order `0yn` by seeding 5 `auditLogs` rows: all three Change-text render paths (Order added, Order removed, non-Order "New Document received") plus actor resolution rendered correctly alongside the order's real audit rows. Branch `feature/MLID-2449-maintenance-history-tab-document-log` is kept.
- **D4-T1 (MLID-2450)** — maintenance fan-out on **intakes upload** — merged into the epic branch (`--no-ff` `40e6babb8`) 2026-06-22. First real backend code change of the epic: `getOrdersByPatient`'s lean (`expandRefs: false`) path gained an optional `collections` option (default `['new']`, zero change for existing callers) and now fans out across the requested collections; `notifyDocumentCreated` passes `['new', 'maintenance']`. **This single change also covers D4-T2 (MLID-2451)** — e-order/fax routes through the same `notifyDocumentCreated`. Also added durable `[Notification Creation]` observability logging through the chain. Verified end-to-end on `0yn`: notification row created + Documents-tab "New" chip / "Mark as Read" appeared when uploader ≠ assignee. **Key gotcha:** the MLID-2434 rule (epic MLID-2444, Egor, commit `9ad4efc8d`) suppresses the in-app notification when the uploader IS the order's assignee — so testing requires uploader ≠ assignee. Branch `feature/MLID-2450-maintenance-intakes-upload-propagation` is kept.
- **D4-T2 (MLID-2451)** — maintenance fan-out on **e-order/fax background-job upload** — merged into the epic branch (`--no-ff` `64e45c422`) 2026-06-24. Verification + test coverage only; **no production code change** — the e-order/fax path (`WeInfuseUploader.uploadPDFToWeInfuse` in `weinfuseUploader.ts`) routes through the same `notifyDocumentCreated` → `getOrdersByPatient({ collections: ['new','maintenance'] })` chain that D4-T1 already extended. **Plan correction:** the original task plan assumed no uploader test file existed and called for a new one; in fact a comprehensive suite already lives at `services/jobs/definitions/jotFormSubmission/__tests__/weinfuseUploader.test.ts` (the `__tests__/` subfolder, not the folder root — the investigation missed it). The 9 new notify-propagation tests were folded into that canonical file (asserting source `e-order`/`fax` mapping, MongoDB patient `_id` threading, `actorId: ''` + `createdBy: 'system'`, `auditSource: 'webhook'`, `linkedOrderIds` + `[]` fallback, fire-and-forget, and error isolation). Because `actorId` is always `''` on this path, the MLID-2434 self-upload gate never fires here. Branch `feature/MLID-2451-eorder-fax-maintenance-propagation` is kept.
- **D4-T3 (MLID-2452, link → maintenance)** — merged into the epic (`--no-ff` `03ddf3410`) 2026-06-25. Verification + tests only; no production code change — the existing `'new' ?? 'maintenance'` fallback in `handleOrderLink` already resolves maintenance orders. Manually verified on order `010w` (linker ≠ assignee). Branch `feature/MLID-2452-maintenance-link-notification` is kept.
- **D4-T4 (MLID-2453, unlink → maintenance)** — merged into the epic (`--no-ff` `a0bea60da`) 2026-06-26. Verification + tests only; no production code change — the unlink path carries no order lookup and no collection branching (the link-orders route hands `handleOrderUnlink` bare orderId strings; `deleteNotificationsForOrderDocument` deletes by the `(order, document)` entity tuple regardless of collection), so a maintenance order's cleanup is identical to a new order's. Added maintenance-framed describe blocks to the link-orders route suite and `handleOrderUnlink.test.ts`. Branch `feature/MLID-2453-maintenance-unlink-notification` is kept.
- **D4-T5 (MLID-2454, maintenance reassignment cascade)** — merged into the epic (`--no-ff` `8d522de66`) 2026-06-26. **The one real production change in D4.** Added the reassignment cascade to the maintenance `PUT` route mirroring the D0-T1 wiring, reusing the collection-agnostic `handleOrderReassignment` helper unmodified. 7 route cascade tests + 1 helper characterization; D0-T1 new-orders behavior unchanged. Branch `feature/MLID-2454-maintenance-reassignment-cascade` is kept. **→ Deliverable 4 (D4-T1 … D4-T5) is now complete on the epic branch.**
- **D5-T1 (MLID-2628, link-orders category pass-through)** — merged into the epic (`--no-ff` `0b22bbf41`) 2026-06-29 and pushed to origin. Collapsed the planned 3-stage rollout into one pass (atomic Next.js deploy, epic not yet shipped, so no dual-shape window needed). `handleOrderLink` now resolves each linked order in a single `getOrderById(id, category)` call (the `'new' ?? 'maintenance'` guess is deleted); `PUT /api/documents/[documentId]/link-orders` accepts only `{ linkedOrders: [{ id, category }] }`; both senders send the category (`UpdateOrderForm` persists `selectedExistingOrderCategory`). Document `linkedOrderIds` storage + `/api/intakes/[id]/link-orders` unchanged. Verified end-to-end on `010w` (both senders); testing notes in `docs/agomez/testing/MLID-2628.md`. Branch `feature/MLID-2628-link-orders-category-passthrough` is kept. **→ Deliverable 5 is complete — the epic is now feature-complete.**

**Epic is feature-complete (all of D1–D5 merged into the epic branch) and the epic → develop PR is open and IN QA.**

**Completed before opening the PR (2026-06-30):**
- Synced `develop` into the epic per workflow: fast-forwarded local `develop`, trial-merged on a throwaway branch (`test/MLID-2417-develop-sync-check`, deleted) to confirm a clean merge, then merged local `develop` into the epic — no rebase, no `git merge origin/develop`. Merge was clean; the only overlap file `ordersTracker.ts` auto-merged with no conflict. Merge commit `ee99c888d`, pushed to origin.
- Wrote the epic-level PR doc (covering **D1–D5 only**; D0 excluded — it shipped off `develop` independently) at `docs/agomez/PR/MLID-2417-epic-to-develop.md`. The stray per-subtask PR doc `MLID-2628.md` was deleted (epic convention: no per-subtask PRs, only the single epic → develop PR).

**Epic branch state:** **synced with latest `develop` and pushed to origin (merge commit `ee99c888d`, 2026-06-30).** On top of `develop` the epic carries D1-T1, D2-T1, D3-T1, all five D4 merges, D5-T1, and the develop-sync merge — **all deliverables complete.** **The single epic → develop PR is open; review requested from the team.**

---

## Purpose

Port the notifications + SignalR event implementation from new orders to **maintenance orders** (the `maintenanceorders` collection), and retrofit two QA-identified edge cases into the existing new-orders implementation along the way.

**Relationship to MLID-2011.** MLID-2011 was the *first* implementation of notifications + SignalR events in the app. Because it was first, it required building the foundation: SignalR service layer, `notifications` and `notificationreads` schemas, the propagation orchestrator, the badge/chip UI, the supporting API routes. Its collection target was `neworders`.

**MLID-2417 is the second consumer of that foundation, not a rebuild.** No net-new infrastructure should be introduced — the work is wiring the existing notification/SignalR machinery into the maintenance-orders surfaces (table, Documents tab, History tab, link-orders route, reassignment route) and extending shared helpers (`getOrdersByPatient`, `handleOrderReassignment`, `handleOrderUnlink`) to span both collections.

---

## Delivery Strategy

Two distinct streams with different urgencies and engineering shapes:

- **D0 — hotfix on new orders.** Bug fixes against code that already shipped in MLID-2011. May need to land in the current sprint to repair live behavior. Self-contained: reassignment and unlink edge cases on `neworders` only. **Each D0 ticket branches off `develop` directly** and ships independently — they do not wait for the rest of the epic. Jira parenting under MLID-2417 is for organizational tracking only.
- **D1–D4 — new feature for maintenance orders.** Extends existing code paths (the propagation orchestrator, `getOrdersByPatient`, the orders-tracker table column factories, the order-detail layout, the link-orders route) so they cover both `neworders` and `maintenanceorders`. Expect refactoring — e.g. parameterizing collection scope on shared helpers. Branches off the epic branch `epic/MLID-2417-document-notifications-maintenance`.

Note: even where D0 and D4 share a helper name (D0-T2 ↔ D4-T4, D0-T1 ↔ D4-T5), they remain separate tickets and separate PRs because the timelines diverge — D0 ships first as an urgent hotfix; D4 lands later as feature/refactor work.

---

## Glossary

- **DN** — Deliverable number (e.g., D0, D1, D2, D3, D4)
- **TN** — Task number within a deliverable (e.g., T1, T2)
- **FS** — Full-stack work (back-end + UI in one task)
- **BE** — Back-end only
- **UI** — Front-end / UI only

---

## Gaps closed by this epic

The maintenance-orders port plus the D0 retrofits close three gaps in the current (post-MLID-2011) implementation. Each gap currently has **no code path** in production:

- **Order reassignment** — `PUT /api/orders/new` updates `assignedTo` but does not touch existing notifications. `targetUserId` remains the previous assignee. The `isCallerOrderAssignee` fallback in read-side authorization, the relaxed `userReadScope: 'any'` mode on the entity-count routes, and the `orderAssignedTo === userId` clause in the client chip filter all exist to paper over this. **Closed by D0-T1** (neworders) and **D4-T5** (maintenanceorders) — with retarget in place, the three workarounds are removed as part of the same fix.
- **Document unlink** — `PUT /api/documents/[documentId]/link-orders` only fires propagation for the *added* delta. Removed orderIds leave their notifications orphaned in `notifications` (still counted by badge queries, still rendered as "New" in the UI). **Closed by D0-T2** (neworders) and **D4-T4** (maintenanceorders).
- **Maintenance-orders coverage** — the upload path's `getOrdersByPatient({ expandRefs: false })` queries only the `neworders` collection (`ordersTracker.ts:1152-1159`). Maintenance orders linked to the same patient receive no notification on upload. The link-orders route is the only existing code that touches `maintenanceorders` today, and only via a fallback after `neworders` lookup. **Closed by D4-T1, D4-T2, D4-T3** (and the UI surfaces in D1-T1, D2-T1, D3-T1).

---

## Architecture & Design Decisions

### Notification propagation — what it is

For a given `(order, document)` pair, **propagation** writes a durable notification, writes an audit-log entry on the order, and broadcasts a SignalR event. Each step has its own `try/catch`, so a failure in one does not abort the others.

Orchestrator: `propagateDocumentNotificationToOrder` in `apps/web/services/notifications/documentNotification.ts:156`. Steps:

1. **Eligibility gate** — `isDocumentEligibleForOrder(document, order)`. Documents with `category === 'Order'` only propagate when `document.linkedOrderIds` includes this orderId. All other categories pass unconditionally. At upload time, `'Order'`-category docs are implicitly skipped because they aren't linked yet.
2. **Durable notification write** — only when `order.assignedTo` is truthy. Writes a `notifications` row with:
   - `type: 'user'`, `priority: 'normal'`
   - `targetUserId: order.assignedTo` (captured at write time)
   - `title: "New Document: <category>"`, `message: "A new document was added via <Upload|E-Order|Fax>."`
   - `entities: [{ entityType: 'order', entityId }, { entityType: 'document', entityId }]`
   - `createdBy: 'system'`
3. **Audit log write** — `writeOrderDocumentAuditEntry` records `action: 'update'`, `field: 'documents'`, `to: { _id: documentId, category, source }` on the order.
4. **SignalR broadcast** — `signalr.broadcastToAll('documentAdded', { documentId, orderId, source, category })`. Broadcasts to every connected client; client hooks filter locally.

### Public entry points (three fan-out shapes)

| Entry point | Fan-out direction | Used by |
|---|---|---|
| `notifyDocumentCreated(document, { source, actorId, auditSource })` | One document → many orders for the same patient | Upload paths (intakes review, e-order, fax) |
| `notifyOrderOfRecentDocuments(order, { auditSource, actorId })` | One new order → all documents from the last 30 days for that patient | `POST /api/orders/new` backfill |
| `handleOrderLink({ documentId, document, newOrderIds, actorId, auditSource })` | One document → the *delta* set of newly-linked orders | `PUT /api/documents/[documentId]/link-orders` (symmetric pair with `handleOrderUnlink`) |

All three early-return when the `ORDER_DOCUMENT_NOTIFICATIONS` feature flag is off.

### Existing trigger sites (new orders today)

| # | Trigger event | Code location | Entry point used |
|---|---|---|---|
| 1 | Document upload via intakes review (split-pdf path A) | `apps/web/app/api/documents/split-pdf/route.ts:358` | `notifyDocumentCreated` (`source: 'upload'`) |
| 2 | Document upload via intakes review (split-pdf path B) | `apps/web/app/api/documents/split-pdf/route.ts:624` | `notifyDocumentCreated` (`source: 'upload'`) |
| 3 | Document upload via e-order / fax (background job) | `apps/web/services/jobs/definitions/jotFormSubmission/weinfuseUploader.ts:343` | `notifyDocumentCreated` (`source: 'e-order' \| 'fax'`) |
| 4 | New order created — backfill recent docs | `apps/web/app/api/orders/new/route.ts:154` | `notifyOrderOfRecentDocuments` |
| 5 | Document linked to one or more orders (delta only) | `apps/web/app/api/documents/[documentId]/link-orders/route.ts:90` | `handleOrderLink` |

### Data model

- **`notifications`** (`apps/web/models/Notification.ts`): `title`, `message`, `type ('task'|'user')`, `priority`, `targetUserId`, `targetLocationId`, `targetRole`, `entities: [{entityType, entityId}]`, `isRead`, `readBy`, `readAt`, `expiresAt`, `createdBy`. Indexes on `targetUserId`, `targetLocationId`, `targetRole`, and `(entities.entityType, entities.entityId)`.
- **`notificationreads`** (`apps/web/models/NotificationRead.ts`): `notificationId` (ref `Notification`), `userId`, `readAt`. Unique index on `(notificationId, userId)` — duplicate-key on write is idempotent (already-read).

### SignalR events the system emits

| Event name | Emitted from | Payload | Audience |
|---|---|---|---|
| `documentAdded` | propagation step 4 (link-side) | `{ documentId, orderId, source, category }` | All connected clients |
| `documentUnlinked` | `handleOrderUnlink` (MLID-2413) | `{ orderId, documentId }` | All connected clients |
| `notification:read` | `PATCH /api/notifications/:id/read` | `{ notificationId, userId }` | All connected clients |
| `notification:read` | `PATCH /api/notifications/bulk-read` | `{ notificationIds, userId }` | All connected clients |
| `order:reassigned` | `handleOrderReassignment` (MLID-2391) | `{ orderId }` | All connected clients |

Hub: `lihub`. Connection string in Azure Key Vault (secret `li-prod-signalr-primary-connection-string` for prod, `li-stage-signalr-primary-connection-string` otherwise; overridable via `SIGNALR_KV_SECRET_NAME`).

### Feature flag

`FeatureFlag.ORDER_DOCUMENT_NOTIFICATIONS` (`apps/web/utils/featureFlags/featureFlags.ts:138`), default `false`. **Reused for maintenance** — no parallel flag. Gates all 5 notification API routes, all 3 propagation entry points, and all UI surfaces (table badge, tab badge, "New" chip, mark-as-read buttons).

### Shared helpers introduced/widened across this epic

| Helper | First built in | Widened by | Purpose |
|---|---|---|---|
| `handleOrderReassignment(order, previousAssignedTo, newAssignedTo)` | D0-T1 (MLID-2391); initial-assignment branch + MLID-2439 gate added by the MLID-2391 hotfix | D4-T5 (MLID-2454) | Two-branch helper: (1) initial assignment (empty → populated) → backfill via `notifyOrderOfRecentDocuments` (the auto-attach notification write was gated out at creation time, so the backfill writes them now); (2) standard reassignment (populated → populated) → retarget existing notifications (`updateMany` flips `targetUserId`). Broadcasts `order:reassigned` after either path for live refetch. Prior `notificationreads` rows survive for audit but stop participating in the unread join because the predicate keys on the current `targetUserId`. **MLID-2439 follow-up**: `propagateDocumentNotificationToOrder` now also gates audit + SignalR on `assignedTo` (previously only the notification write was gated), so the backfill path produces a single audit row per document instead of a duplicate. |
| `handleOrderLink({ documentId, document, newOrderIds, actorId, auditSource })` | MLID-2086 (pre-epic, renamed in D0-T2/MLID-2413) | — | For each newly-linked orderId: look up the order, hand it to the orchestrator (notification + audit + `documentAdded` broadcast). Single-caller from the link-orders route. Symmetric pair with `handleOrderUnlink`. |
| `handleOrderUnlink(documentId, removedOrderIds, { actorId, documentCategory, documentSource })` | D0-T2 (MLID-2413) | D4-T4 (MLID-2453) | For each removed `(orderId, documentId)` pair: delete matching `notifications` row(s), delete the associated `notificationreads` row(s), write a remove-side `auditLogs` entry on the order, and broadcast `documentUnlinked`. |
| `getOrdersByPatient(..., { collections: ['new', 'maintenance'] })` (extension) | D4-T1 (MLID-2450) | D4-T2 (MLID-2451) — and may simplify D4-T3 | Multi-collection patient → orders lookup for upload-time fan-out |

### General rules across all tasks

- All propagation respects the eligibility rule above: `category === 'Order'` documents only propagate when explicitly linked.
- "Delete notification records for `(order, document)`" or `(order, document, user)` = delete rows in `notifications` where the `entities` array contains both the order and document entity tuples, optionally also filtering on `targetUserId`. **Also delete the matching `notificationreads` rows referencing those notification IDs.** *(PO 2026-05-26 reversed the prior audit-retention exemption — unlink cleanup must be complete.)*
- "Audit-log entry for a document link/unlink" = write an `auditLogs` row on the order with `action: 'update'`, `source: 'user'`, `actorId: <session user>`, `changes: [{ field: 'documents', from, to }]`. On link: `from: null, to: { _id, category, source }`. On unlink: the structural inverse, `from: { _id, category, source }, to: null`. The order-history "Change" column renders **only `category: 'Order'` rows** with the new PO text — `A Document has been added (Document type = Order)` when `from === null`, `A Document has been removed (Document type = Order)` when `to === null`. **Documents of every other category continue to render the legacy `New Document received (Document type = <category>)` format unchanged.** *(Per PO 2026-05-26, the text change is Order-category scoped; non-Order auto-attachment audit rows existed prior and their rendering is unaffected.)*
- Reassignment is a **retarget**, not a delete + recreate: `updateMany` flips `targetUserId` from previous to new. Prior `notificationreads` rows survive in the collection but stop participating in the unread join (the join keys on the notification's current `targetUserId`). The new assignee sees everything as unread automatically; the PO requirement is satisfied by the join semantics, not by an explicit reset step.
- TDD per task: write failing tests first, then implementation, then refactor.
- Every task must verify the existing new-orders behavior is unchanged.

**Area-label note for D0:** D0-T1 and D0-T2 are *scope-equivalent* to D4-T5 and D4-T4 respectively (which are labeled BE), but D0-T1 and D0-T2 are labeled **FS** because the hotfix must be verified against the existing MLID-2011 UI surfaces — the orders-tracker table badge, the Documents tab badge, the per-row "New" chips, and the History log — to confirm the badges/chips clear correctly when the trigger fires. D4-T4 and D4-T5 inherit the verification through the maintenance-orders UI delivered by D1–D3, so they're pure BE.

---

## Summary Table

One-glance view of all sub-tasks across deliverables.

| Deliverable | Task | Area | Ticket | Scope |
|---|---|:--:|---|---|
| **D0** — Retrofit fixes for new orders | T1 | FS | MLID-2391 | New order reassigned: delete previous-user records, propagate to new user |
| **D0** — Retrofit fixes for new orders | T2 | FS | MLID-2413 | Document unlinked from new order: delete `(order, document)` records |
| **D1** — Maintenance orders general table | T1 | FS | MLID-2447 | Unread-count badge per row on `/orders/maintenance` |
| **D2** — Maintenance order detail, Documents tab | T1 | FS | MLID-2448 | Tab badge + per-row "New" chip + per-row "Mark as read" + "Mark all as read" |
| **D3** — Maintenance order detail, History tab | T1 | FS | MLID-2449 | Log entry per associated document (Order-category only when linked) |
| **D4** — Maintenance triggers | T1 | BE | MLID-2450 | Intakes upload → propagate to all maintenance orders for the patient |
| **D4** — Maintenance triggers | T2 | BE | MLID-2451 | E-order/fax upload → propagate to all maintenance orders for the patient |
| **D4** — Maintenance triggers | T3 | BE | MLID-2452 | Document linked to maintenance order → propagate |
| **D4** — Maintenance triggers | T4 | BE | MLID-2453 | Document unlinked from maintenance order → delete notification records |
| **D4** — Maintenance triggers | T5 | BE | MLID-2454 | Maintenance order reassigned → delete previous-user records + propagate to new user |
| **D5** — Link-orders cleanup | T1 | FS | MLID-2628 | Thread order category to `handleOrderLink` → resolve order in one query, remove cross-collection guess |

---

## Deliverable 0: Retrofit fixes for new orders (hotfix scope)

### Tracking

| Task ID | Summary | SP | Status | Branch | Merged to develop | Notes |
|---------|---------|:--:|:------:|--------|:-----------------:|-------|
| MLID-2391 | [D0-T1] Reassignment cascades notifications to new assignee (neworders) | 3 | Done | `feature/MLID-2391-reassignment-cascade-neworders` | ✅ | Original PR merged to develop. |
| MLID-2391 (hotfix) | Initial-assignment notification backfill (empty → populated) | — | Done | `fix/MLID-2391-initial-assignment-notification-backfill` | ✅ | Post-merge follow-up to MLID-2391 — surfaced during MLID-2413 testing when an unassigned order's auto-attached docs never produced notifications after assignment. Also bundled the MLID-2439 fix (see below) since the same `assignedTo` gate change resolves it. PR doc at `docs/agomez/PR/MLID-2391-hotfix.md`. |
| MLID-2413 | [D0-T2] Order notification badge — counter mismatch on document unlink | 4 | Done | `fix/MLID-2413-document-unlink-notification-cleanup` | ✅ | Reported by QA (Olha). PR merged to develop. |
| MLID-2439 (part 1: gate) | Orphan / duplicate audit rows on unassigned orders | — | Done | (bundled into the MLID-2391 hotfix branch) | ✅ | `propagateDocumentNotificationToOrder` was writing audit rows + broadcasting `documentAdded` even when `order.assignedTo` was empty. Combined with the MLID-2391 hotfix, this produced duplicate Order-History rows after assignment. Fixed by gating all three propagator side-effects (notification + audit + broadcast) on `assignedTo` truthy. |
| MLID-2439 (part 2: lookback) | Disable the 30-day backfill lookback window | — | Done | `fix/MLID-2439-disable-backfill-lookback` | ✅ | PO direction 2026-05-27 — surface ALL of the patient's documents to a new order regardless of age, not just the last 30 days. Sentinel pattern: `BACKFILL_LOOKBACK_DAYS: number = -1` disables the filter, the date-filter code path is preserved so re-enabling is one-line. PR doc at `docs/agomez/PR/MLID-2439.md`. Branch also carried one unrelated piggyback commit (`[MLID-000]` turbo.json `ui: tui → stream` — PE pre-approved). PR merged to develop. |

**D0 PRs to develop:** ✅ All D0 work merged — MLID-2391 original, MLID-2391 hotfix (+ MLID-2439 part 1), MLID-2413, and MLID-2439 part 2.

### MLID-2391 — Criteria & Implementation (D0-T1, FS)

**Criteria:**

- When a new order is reassigned to another user, the existing notification records targeting the previous assignee for that order are retargeted to the new assignee — `updateMany` flips `targetUserId` from previous to new for every notification whose `entities` contains the order tuple.
- The new assignee sees every affected document as **unread**, even ones the previous assignee had marked as read (per PO instruction). This is satisfied by the unread predicate joining on the notification's current `targetUserId`, so prior `notificationreads` rows from the previous assignee no longer match.
- The cascade is fire-and-forget — a failure must NOT block the order-update HTTP response.
- Partner orders (`-1` suffix orders propagated by `updateOrder`) must be covered — every `_id` from the `updateOrder` result is included.
- A SignalR `order:reassigned { orderId }` event is broadcast after a successful retarget so both the previous and new assignee's open clients refetch.
- Un-assignment is out of scope (the UI cannot clear `assignedTo` on `PUT /api/orders/new`).
- The three workarounds that exist today to compensate for stale `targetUserId` are removed in the same PR — the read-side is no longer safe to leave in its relaxed form once the retarget runs.

**Implementation:**

- Trigger site: `PUT /api/orders/new` (`apps/web/app/api/orders/new/route.ts`), between the audit await and the response — fire-and-forget, one invocation per item in the `updateOrder` result array.
- Build the `handleOrderReassignment(order, previousAssignedTo, newAssignedTo)` helper here. D4-T5 (MLID-2454) reuses it for `maintenanceorders` without modification — the helper does not branch on collection.
- Coordinated workaround removal (all in the same PR):
  - **Predicate flip on entity-count routes**: `/api/notifications/counts` and `/api/notifications/by-order/:orderId` switch from `userReadScope: 'any'` to a current-target join (`notificationreads.userId === notification.targetUserId`). The `'any'` branch of `buildUserUnreadLookup` becomes dead code and is removed.
  - **Server-side mark-as-read fallback removed**: `isCallerOrderAssignee` deleted; `markUserNotificationAsRead` and `bulkMarkUserNotificationsAsRead` accept only strict `targetUserId === userId`.
  - **Client chip filter relaxed clause removed**: `useDocumentsTabNotifications` drops the `|| orderAssignedTo === userId` clause; the chip and mark-as-read button render only when `targetUserId === userId`. The hook signature loses its `orderAssignedTo` parameter.
- Client hooks (`useDocumentsTabNotifications`, `useOrderNotificationCounts`) add an `order:reassigned` SignalR listener.

**Tests:**

- Unit tests for `reassignUserNotificationsForOrder` (filter shape, return value, error re-throw).
- Unit tests for `handleOrderReassignment` (feature flag, defensive empty/equal early-returns, retarget+broadcast ordering, retarget rejection skips broadcast, SignalR failure is non-fatal).
- Integration tests for the PUT route covering: trigger on assignedTo change (including partner orders), no-trigger conditions, response is 200 even when the orchestrator rejects.
- Regression tests on the read-side: a prior-assignee `notificationreads` row no longer clears the badge; the chip does not appear for an order-assignee who is not the notification's target; mark-as-read rejects callers who are the order's assignee but not the target.
- Hook tests verify the new `order:reassigned` listener triggers refetch.
- Verify on staging: the previous assignee's badges/chips clear and the new assignee sees every document as unread (including ones the previous assignee had marked).

### MLID-2413 — Criteria & Implementation (D0-T2, FS)

> **Spec updated 2026-05-26 after PO conversation.** Two new criteria added (audit-log entries on link/unlink with specific "Change" column text), and the unlink cleanup now also deletes `notificationreads` rows (prior plan kept them as audit artifacts — reversed).

**Criteria:**

- Unlinking a `category: "Order"` document from a new order decreases the badge count on the Documents tab (and on the orders-tracker table row) by exactly the number of unread notifications for that `(order, document)` pair.
- After unlinking, the "New" chip and "Mark as Read" button for that document disappear from the order's Documents tab.
- On unlink, the matching notification record(s) for the `(order, document)` pair are deleted from `notifications`, AND the associated `notificationreads` rows (for any user) are deleted as well. *(Change from prior plan — PO 2026-05-26 reversed the audit-retention exemption.)*
- **Linking** a `category: "Order"` document to a new order writes a new `auditLogs` entry on that order; the order-history "Change" column renders that entry as **`A Document has been added (Document type = Order)`**.
- **Unlinking** a `category: "Order"` document from a new order writes a new `auditLogs` entry on that order; the order-history "Change" column renders that entry as **`A Document has been removed (Document type = Order)`**. *(Today no audit entry is written on unlink — see Root cause.)*
- Re-linking the document to the same order creates fresh notification records AND a fresh `A Document has been added (Document type = Order)` audit entry.
- **Renderer scope**: the new "added"/"removed" text applies **only to `category: "Order"` audit rows**. Documents of every other category (Referral Copy, Other, etc.) continue to render the legacy `New Document received (Document type = <category>)` text unchanged.

**Root cause:**

`PUT /api/documents/[documentId]/link-orders` at `apps/web/app/api/documents/[documentId]/link-orders/route.ts:90` only fires propagation for the **added** delta of order IDs. When the linked-orders set shrinks (an unlink), the removed orderIds leave their notification records orphaned in `notifications` (still counted by the badge query, still shown as "New"), no `notificationreads` cleanup runs, and no `auditLogs` entry is written for the remove side (so order-history shows no record of the unlink).

**Implementation:**

- In the same route, compute `removedOrderIds = previousLinkedOrderIds.filter(id => !sanitizedIds.includes(id))` alongside the existing added-delta computation.
- For each `(orderId, documentId)` pair in the removed set, `handleOrderUnlink` performs four steps in order:
  1. Delete the matching notification record(s) from `notifications` (entities array contains both order and document entity tuples).
  2. Delete the `notificationreads` row(s) that reference those `notificationId`s.
  3. Write a new `auditLogs` entry on the order with `action: 'update'`, `source: 'user'`, `actorId: <session user>`, `changes: [{ field: 'documents', from: { _id, category, source }, to: null }]` — the structural inverse of the add-side write that already exists in propagation step 3.
  4. Broadcast `documentUnlinked { orderId, documentId }` so open clients refresh badges + history.
- Build the `handleOrderUnlink(documentId, removedOrderIds, { actorId })` helper here. D4-T4 (MLID-2453) reuses it for maintenance orders.
- Order-history "Change" column renderer: when `changes[0].field === 'documents'`, output `A Document has been added (Document type = <category>)` if `changes[0].from === null`, or `A Document has been removed (Document type = <category>)` if `changes[0].to === null`. This renderer change affects both the existing add-side rows (already present in `auditLogs`) and the new remove-side rows.
- The link-side `auditLogs` write itself already exists today (propagation step 3 `writeOrderDocumentAuditEntry`); no new write is needed on the add side — only the renderer change to match the PO's text format.

**Tests:**

- Unit tests for `handleOrderUnlink` covering all four side-effects (notifications delete, notificationreads delete, audit entry write, SignalR broadcast).
- Integration tests for `PUT /api/documents/[documentId]/link-orders` covering the removed-delta path (asserting all four side-effects fire with the correct payloads).
- Renderer tests for the order-history "Change" column verifying the add/remove text formats for both Order-category and pre-existing rows.

---

## Deliverable 1: Maintenance orders general table

### Tracking

| Task ID | Summary | SP | Status | Branch | Merged to Epic | Notes |
|---------|---------|:--:|:------:|--------|:--------------:|-------|
| MLID-2447 | [D1-T1] Display unread notification badge on maintenance orders table | 2 | Done | `feature/MLID-2447-maintenance-orders-table-badge` | ✅ | Merged into epic (local, `--no-ff` `230529ce3`) 2026-06-19. Badge verified locally with seeded notifications. |

**D1 PR to develop:** Not yet

### MLID-2447 — Criteria & Implementation (D1-T1, FS)

**Criteria:**

- On the `/orders/maintenance` table, each row shows a notification badge with the count of unread notifications for that order (only visible when count > 0).
- The badge renders the same visual component as the new-orders table (consistent UX between categories).
- Counts refresh live via the existing SignalR `documentAdded` and `notification:read` events (no page reload required).
- When the feature flag `ORDER_DOCUMENT_NOTIFICATIONS` is OFF, no badge is rendered.
- Existing `/orders/new` badge behavior is unchanged.

**Implementation:**

- Reuse `useOrderNotificationCounts(visibleOrderIds)` and `POST /api/notifications/counts` (both already serve the new-orders table).
- In `apps/web/app/orders-tracker/[category]/page.tsx`, extend the `visibleOrderIds` `useMemo` so it also fires when `tab === 'maintenance'` (today it early-returns `[]` unless `tab === 'new'`). The `useOrderNotificationCounts(visibleOrderIds)` call already lives right below it.
- `getMaintenanceOrdersColumns` now lives in its own file `apps/web/app/orders-tracker/Orders/MaintenanceOrders.tsx` (split out of `NewOrders.tsx`). Add a `unreadCountsByOrderId?: Record<string, number>` prop to it, mirroring `getNewOrdersColumns` in `NewOrders.tsx`, which already accepts it.
- Pass `unreadCountsByOrderId` from the `getMaintenanceOrdersColumns({...})` call site in `page.tsx`, gated on `isNotificationsEnabled` exactly like the adjacent `getNewOrdersColumns({...})` call; add it to that `useMemo`'s dependency array.
- Port the badge block: the maintenance `displayId` renderer in `MaintenanceOrders.tsx` mirrors the new-orders `displayId` renderer but omits the badge. Copy the badge JSX from the `displayId` renderer in `NewOrders.tsx` (the `showBadge`/`unreadCount` block) and point its link at `/orders-tracker/maintenance/${row._id}/documents`.
- Extract the badge cell renderer into a shared `OrderNotificationBadge` component used by both column factories. Now that the two factories live in separate files, inlining would duplicate the badge JSX in both.

**Tests:**

- Component tests for `getMaintenanceOrdersColumns` (`MaintenanceOrders.tsx`) rendering the badge when the order has a non-zero unread count.
- Verify existing `/orders/new` badge behavior is unchanged.

---

## Deliverable 2: Maintenance order detail — Documents tab

### Tracking

| Task ID | Summary | SP | Status | Branch | Merged to Epic | Notes |
|---------|---------|:--:|:------:|--------|:--------------:|-------|
| MLID-2448 | [D2-T1] Documents tab notifications on maintenance order detail (badge, "New" chip, mark-as-read, mark-all-as-read) | 3 | Done | `feature/MLID-2448-maintenance-documents-tab-notifications` | ✅ | Merged into epic (local, `--no-ff`) 2026-06-19. Verification + maintenance test coverage only — the shared `[category]` route had no category gate, so no production code change. Verified manually on maintenance order `0yn` across two sessions: assignee sees badge + "New" chips + mark-as-read + mark-all; non-target sees the same badge count but no per-row controls; mark-as-read decremented the badge live (SignalR `notification:read`); feature-flag OFF hid everything. |

**D2 PR to develop:** Not yet

### MLID-2448 — Criteria & Implementation (D2-T1, FS)

**Criteria:**

- The "Documents" tab on `/orders/maintenance/[id]/documents` shows a badge with the total count of unread notifications for that order in the top-right corner of the tab label.
- Each document row that has an unread notification for the current user shows a "New" chip. "Current user" matches **strictly** when `notification.targetUserId === userId`. (The `OR order.assignedTo === userId` fallback was removed in D0-T1/MLID-2391 once retarget kept `targetUserId` always-live; the `isDocumentUnreadForUser` helper enforces the strict rule.)
- Each document row with an unread notification for the current user shows a "Mark as read" button. Clicking it inserts a `notificationreads` record for that `(notification, user)` pair and removes the chip + button immediately.
- A "Mark all as read" button at the top of the documents table marks every notification the current user owns on this order as read (one or more `notificationreads` inserts).
- Counts and chips refresh live via the existing SignalR `documentAdded`, `documentUnlinked`, `notification:read`, and `order:reassigned` events.
- When the feature flag is OFF, no badge / chip / mark-as-read controls render.
- Existing `/orders/new/[id]/documents` behavior is unchanged.

**Implementation:**

> **Scope note:** the order-detail layout (`apps/web/app/orders-tracker/[category]/[orderId]/layout.tsx`) and the documents page (`.../[orderId]/documents/page.tsx`) both live under the shared `[category]` dynamic route, and none of the notification UI is gated to `category === 'new'`. As a result the tab badge, the per-row "New" chip, "Mark as read", and "Mark all as read" already render for maintenance with no code change. This task is therefore mostly **verification + maintenance-specific test coverage** — the affordances only need maintenance notification *data*, which D4 produces. Widen scope only if a category gate is found.

- Reuse `useDocumentsTabNotifications(orderId)` (single argument — the `order.assignedTo` parameter was dropped in D0-T1/MLID-2391) and `GET /api/notifications/by-order/:orderId`. The hook already listens to `documentAdded`, `documentUnlinked`, `notification:read`, and `order:reassigned`.
- Tab badge: rendered by `layout.tsx` from the hook's `unreadCount`, gated only on `isNotificationsEnabled && documentsUnreadCount > 0` — already category-agnostic, verify for maintenance.
- Per-row "New" chip + "Mark as read" live in `documents/page.tsx`; both use `isDocumentUnreadForUser(summary, userId)` (strict `targetUserId === userId`). Per-row mark-as-read → `PATCH /api/notifications/:id/read`. "Mark all as read" → `POST /api/notifications/bulk-read`.
- Known double-fetch (still present): `useDocumentsTabNotifications` is called from both `layout.tsx` (tab badge) and `documents/page.tsx` (per-row state), producing two requests on mount. Optionally lift state to a Context provider as part of this task.

**Tests:**

- Component tests for the maintenance Documents tab covering: tab badge with non-zero count, "New" chip presence by the strict `targetUserId` rule, "Mark as read" inserting a `notificationreads` row, "Mark all as read" inserting multiple.
- Verify existing new-orders Documents tab behavior is unchanged.

---

## Deliverable 3: Maintenance order detail — History tab

### Tracking

| Task ID | Summary | SP | Status | Branch | Merged to Epic | Notes |
|---------|---------|:--:|:------:|--------|:--------------:|-------|
| MLID-2449 | [D3-T1] Show document-upload log entries on maintenance order History tab | 2 | Done | `feature/MLID-2449-maintenance-history-tab-document-log` | ✅ | Merged into epic (`--no-ff` `6278462c9`) 2026-06-22. Verification + maintenance test coverage only — the History tab (`order-history/page.tsx`) reads only `orderId` from `useParams`, never `[category]`, and the audit-log stack + `documents` formatter are category-agnostic, so no production code change. Verified manually on maintenance order `0yn` by seeding 5 `auditLogs` rows: all three Change-text render paths (Order added, Order removed, non-Order) plus actor resolution rendered correctly alongside the order's real audit rows. Branch kept. |

**D3 PR to develop:** Not yet

### MLID-2449 — Criteria & Implementation (D3-T1, FS)

**Criteria:**

- On `/orders/maintenance/[id]/order-history`, the history log shows an entry for each document propagated to this order.
- Documents with `category === 'Order'` appear only when explicitly linked to this order (`document.linkedOrderIds` includes this orderId).
- Documents with any other category appear unconditionally (matches propagation eligibility).
- Each log entry shows at minimum: document name/category, source (Upload / E-Order / Fax), and timestamp.
- Layout, sorting, and pagination match the equivalent new-orders History tab.
- Existing `/orders/new/[id]/order-history` behavior is unchanged.

**Implementation:**

> **Scope note:** the History tab is the shared `apps/web/app/orders-tracker/[category]/[orderId]/order-history/page.tsx`, which renders a generic audit log via `useAuditLog({ entityType: 'order', entityId })` with no order-category gating. The document "Change" text is produced by the shared `documents` formatter in `apps/web/services/mongodb/auditLog.config.ts` (emits `A Document has been added/removed (Document type = Order)` for Order-category rows, `New Document received (Document type = <category>)` otherwise). Nothing here is new-orders-specific, so this task reduces to **verification + maintenance test coverage** once D4 writes audit entries for maintenance orders.

- Log entries are derived from audit records written by `writeOrderDocumentAuditEntry` (step 3 of the propagation orchestrator). Audit entries are only written when the eligibility gate passes, so the "Order-category only when linked" rule is satisfied automatically — no extra read-time filtering required.
- No History-tab parameterization is needed (the page is already category-shared and the formatter is category-agnostic). Verify the maintenance order-history route renders the document entries; widen scope only if a gate is found.

**Tests:**

- Component tests covering the maintenance History tab rendering audit entries.
- Verify existing new-orders History tab behavior is unchanged.

---

## Deliverable 4: Maintenance event triggers (back end)

### Tracking

| Task ID | Summary | SP | Status | Branch | Merged to Epic | Notes |
|---------|---------|:--:|:------:|--------|:--------------:|-------|
| MLID-2450 | [D4-T1] Propagate notifications to maintenance orders on intakes upload | 3 | Done | `feature/MLID-2450-maintenance-intakes-upload-propagation` | ✅ | Merged into epic (`--no-ff` `40e6babb8`) 2026-06-22. Added `collections` option to `getOrdersByPatient` lean path (default `['new']`) + `notifyDocumentCreated` passes `['new','maintenance']`; plus `[Notification Creation]` logging. Verified end-to-end on `0yn` (uploader ≠ assignee, per the MLID-2434 self-upload suppression rule). Branch kept. |
| MLID-2451 | [D4-T2] Propagate notifications to maintenance orders on e-order/fax upload | 1 | Done | `feature/MLID-2451-eorder-fax-maintenance-propagation` | ✅ | Merged into epic (`--no-ff` `64e45c422`) 2026-06-24. Verification + tests only; no production change (background-job path routes through the same `notifyDocumentCreated` → `getOrdersByPatient` chain D4-T1 extended). 9 notify-propagation tests folded into the existing canonical suite `__tests__/weinfuseUploader.test.ts`. Branch kept. |
| MLID-2452 | [D4-T3] Propagate notifications when a document is linked to a maintenance order | 2 | Done | `feature/MLID-2452-maintenance-link-notification` | ✅ | Merged into epic (`--no-ff` `03ddf3410`) 2026-06-25. Verification + tests only; **no production code change** — `handleOrderLink` already resolves linked orders via the `getOrderById(id, 'new') ?? getOrderById(id, 'maintenance')` fallback and hands them to the collection-agnostic `propagateDocumentNotificationToOrder`. Added the "when the order resolves from the maintenance collection" describe block to `documentNotification.test.ts` (full contract — notification targeting `assignedTo`, audit entry, `documentAdded` broadcast — plus the empty-`assignedTo` skip). Manually verified on maintenance order `010w` (`6a3c6976b9900714c97f552c`): all four surfaces confirmed with linker ≠ assignee (MLID-2434 self-action gate correctly OFF). Branch kept. |
| MLID-2453 | [D4-T4] Delete notifications when a document is unlinked from a maintenance order | 2 | Done | `feature/MLID-2453-maintenance-unlink-notification` | ✅ | Merged into epic (`--no-ff` `a0bea60da`) 2026-06-26. Verification + tests only; **no production code change** — the unlink path carries no order lookup and no collection branching: the link-orders route hands `handleOrderUnlink` bare orderId strings, and `deleteNotificationsForOrderDocument` deletes by the `(order, document)` entity tuple regardless of which collection the order lives in, so a maintenance order's cleanup is identical to a new order's. Added a maintenance-framed describe block to both suites: the link-orders route test (a removed maintenance order is delegated to `handleOrderUnlink` with full options + a mixed new/maintenance removed delta flows through as one id set — locks in D0-T2/MLID-2413 behavior) and `handleOrderUnlink.test.ts` (the full four-step cleanup — notifications delete → reads delete → remove-side audit → `documentUnlinked` broadcast — fires keyed on the maintenance orderId). Both blocks carry intent comments so a future `category === 'new'` gate on the unlink branch would break them. Branch kept. |
| MLID-2454 | [D4-T5] Re-target notifications when a maintenance order is reassigned | 3 | Done | `feature/MLID-2454-maintenance-reassignment-cascade` | ✅ | Merged into epic (`--no-ff` `8d522de66`) 2026-06-26. **The one real production change in D4.** Added the reassignment cascade to the maintenance `PUT` route (`apps/web/app/api/orders/maintenance/route.ts`): after the audit await, capture `previousAssignedTo` from the pre-update order, derive `reassignedTo` from `assignedToFields`, and fire-and-forget `handleOrderReassignment(updatedOrder, previousAssignedTo, reassignedTo)` for each order returned by `updateOrder` (partner orders included) when the assignee changed — each `.catch()`-guarded so the response is never delayed/500. Mirrors the D0-T1 wiring on `/api/orders/new`; reuses the collection-agnostic `handleOrderReassignment` helper unmodified (backfill vs retarget branch + `order:reassigned` broadcast). Tests: maintenance route `reassignment cascade` block (7 tests — fires per updated order incl. partners; no-fire on omitted/unchanged assignee + lacking permission; `previousAssignedTo: ''` for missing-field and empty-string initial assignment; 200 on cascade rejection) + one maintenance-framed characterization in `handleOrderReassignment.test.ts`. D0-T1 new-orders behavior unchanged (its suite passes). Coverage on `route.ts` 99.54% lines. Branch kept. |

**D4 PR to develop:** Not yet — **Deliverable 4 complete** (D4-T1 … D4-T5 all merged into the epic and pushed to origin at `8d522de66`, 2026-06-26). No separate D4 PR; folds into the single epic → develop PR opened after D5.

### MLID-2450 — Criteria & Implementation (D4-T1, BE)

**Criteria:**

- On intakes-review document upload (split-pdf path), every maintenance order whose patient matches the uploaded document receives a notification propagation for `(order, order.assignedTo, document)`.
- Maintenance orders whose `assignedTo` is empty are skipped (no notification row, no SignalR broadcast).
- The existing eligibility rule still applies: `category === 'Order'` documents are implicitly skipped at upload time because they aren't linked to any order yet (linking is the D4-T3 trigger).
- When the feature flag is OFF, no propagation runs.
- Existing new-orders propagation behavior on intakes upload is unchanged.

**Implementation:**

- Trigger sites: the two `notifyDocumentCreated` call sites in `apps/web/app/api/documents/split-pdf/route.ts` (paths A and B). Both already fire propagation.
- The fix is narrowly scoped: `getOrdersByPatient` in `apps/web/services/mongodb/ordersTracker.ts` already queries **both** `neworders` and `maintenanceorders` in its enriched (`expandRefs: true`) path — only the **lean (`expandRefs: false`) branch** is still neworders-only (it aggregates `getOrdersCollection('new')` and projects `{ _id, assignedTo }`). Extend that lean branch to also query `getOrdersCollection('maintenance')`, ideally behind a `collections` option so callers control scope.
- `notifyDocumentCreated` calls `getOrdersByPatient(patientId, { expandRefs: false })`, so this single change covers D4-T1 and D4-T2. (Note: `notifyDocumentCreated` now takes an options object — `{ documentId, patientId, category, source, ... }` — not a `(document, opts)` pair.)

**Tests:**

- Unit tests for the multi-collection patient → orders lookup.
- Integration tests for both split-pdf trigger paths for maintenance orders.
- Verify existing new-orders propagation on intakes upload is unchanged.

### MLID-2451 — Criteria & Implementation (D4-T2, BE)

**Criteria:**

- On e-order/fax document upload via the background job, every maintenance order whose patient matches receives a notification propagation for `(order, order.assignedTo, document)`.
- Maintenance orders whose `assignedTo` is empty are skipped.
- The existing eligibility rule still applies (`category === 'Order'` implicitly skipped at upload time).
- When the feature flag is OFF, no propagation runs.
- Existing new-orders propagation on e-order/fax upload is unchanged.

**Implementation:**

- Trigger site: the `notifyDocumentCreated` call in `apps/web/services/jobs/definitions/jotFormSubmission/weinfuseUploader.ts` (with `source` resolved to `'fax'` for spruce, otherwise `'e-order'`).
- **No additional code change** beyond the lean-path `getOrdersByPatient` multi-collection extension done in D4-T1 — this trigger routes through the same `notifyDocumentCreated` → `getOrdersByPatient({ expandRefs: false })` path. This task is primarily test coverage + verification on the background-job path.

**Tests:**

- Integration tests covering the maintenance-orders fan-out on the e-order/fax job.
- Verify existing new-orders propagation on e-order/fax is unchanged.

### MLID-2452 — Criteria & Implementation (D4-T3, BE)

**Criteria:**

- Linking a document to a maintenance order creates a notification record for `(order, order.assignedTo, document)`.
- If the order's `assignedTo` is empty, propagation is skipped (same guard as D4-T1/T2).
- The propagation also writes an audit-log entry and emits the `documentAdded` SignalR event (standard 4-step contract).
- When the feature flag is OFF, no propagation runs.
- Existing new-orders propagation on link is unchanged.

**Implementation:**

- Trigger site: the `handleOrderLink` call in `apps/web/app/api/documents/[documentId]/link-orders/route.ts` (fired for the newly-added orderId delta). `handleOrderLink` (in `apps/web/services/notifications/documentNotification.ts`) already resolves each order via `getOrderById(orderId, 'new') ?? getOrderById(orderId, 'maintenance')`, then hands it to the collection-agnostic `propagateDocumentNotificationToOrder`.
- The maintenance fallback is already in place, so this task is **verification + test coverage**: confirm propagation fires correctly when the resolved order is in `maintenanceorders` (notification write gated on `assignedTo`, audit entry, `documentAdded` broadcast). Widen the lookup only if a gap is found.

**Tests:**

- Integration tests covering the link path resolving to a maintenance order.

### MLID-2453 — Criteria & Implementation (D4-T4, BE)

**Criteria:**

- Unlinking a document from a maintenance order deletes the matching notification record(s) for that `(order, document)` pair from the `notifications` collection.
- The maintenance order's Documents-tab badge and per-row "New" chip update accordingly (live via SignalR).
- When the feature flag is OFF, no deletion runs.
- The corresponding new-orders unlink fix (D0-T2 / MLID-2413) continues to behave correctly (shared code path).

**Implementation:**

- Same route as D4-T3: the `removedOrderIds` branch of `apps/web/app/api/documents/[documentId]/link-orders/route.ts`, which already computes the removed delta and calls `handleOrderUnlink`.
- The `handleOrderUnlink(documentId, removedOrderIds, options)` helper built in D0-T2 (MLID-2413) now lives in its own file `apps/web/services/notifications/handleOrderUnlink.ts` (options: `{ actorId, documentCategory, documentSource }`). It deletes notifications via `deleteNotificationsForOrderDocument(orderId, documentId)`, deletes the associated reads, writes a remove-side audit entry (`writeOrderDocumentUnlinkAuditEntry`), and broadcasts `documentUnlinked` — all keyed on the `(orderId, documentId)` tuple, so it is collection-agnostic and works for `maintenanceorders` without modification (its own docstring calls out this task).
- This task is therefore **verification + test coverage** on the maintenance unlink path.

**Tests:**

- Integration tests covering the unlink path resolving to a maintenance order.
- Verify D0-T2's neworders unlink behavior continues to work.

### MLID-2454 — Criteria & Implementation (D4-T5, BE)

**Criteria:**

- Reassigning a maintenance order to a different user removes the previous user's badges, "New" chips, and "Mark as Read" controls on the Documents tab.
- The new assignee sees the badges, "New" chips, and "Mark as Read" controls without page reload (SignalR refetch).
- Un-assignment (`assignedTo` cleared) removes the previous user's records and creates nothing new.
- The cascade is fire-and-forget — failure does NOT block the order-update HTTP response.
- Partner orders (`-1` suffix orders propagated by `updateOrder`) are covered.
- When the feature flag is OFF, no cascade runs.
- The corresponding new-orders reassignment fix (D0-T1 / MLID-2391) continues to behave correctly (shared helper).

**Implementation:**

- Trigger site: the `PUT` handler in `apps/web/app/api/orders/maintenance/route.ts`. It already updates `assignedTo` (`assignedToFields`) and calls `updateOrder` (partner-order aware), but has **no** reassignment cascade today — this is the one real code change in D4-T5.
- Mirror the D0-T1 wiring from `apps/web/app/api/orders/new/route.ts`: capture `previousAssignedTo` from the pre-update order, compute the new assignee, and after `updateOrder` returns its array, fire-and-forget `handleOrderReassignment(updatedOrder, previousAssignedTo, reassignedTo)` for each updated order when `reassignedTo !== previousAssignedTo` (each call `.catch()`-guarded so the route response is never delayed). Confirm the handler reads the current order before the update so `previousAssignedTo` is available.
- Reuse the `handleOrderReassignment(order, previousAssignedTo, newAssignedTo)` helper (own file `apps/web/services/notifications/handleOrderReassignment.ts`, built in D0-T1/MLID-2391). It branches between backfill (empty → populated, via `notifyOrderOfRecentDocuments`) and retarget (populated → populated, via `reassignUserNotificationsForOrder`), then broadcasts `order:reassigned`. Collection-agnostic — works for `maintenanceorders` without modification (its own docstring calls out this task).

**Tests:**

- Unit tests for the shared helper covering both collections.
- Integration tests for the maintenance-order reassignment route.
- Verify D0-T1's neworders reassignment behavior continues to work.

---

## Deliverable 5: Link-orders category pass-through (cleanup)

### Tracking

| Task ID | Summary | SP | Status | Branch | Merged to Epic | Notes |
|---------|---------|:--:|:------:|--------|:--------------:|-------|
| MLID-2628 | [D5-T1] Link-orders category pass-through — remove cross-collection guessing in `handleOrderLink` | 3 | Done | `feature/MLID-2628-link-orders-category-passthrough` | ✅ | Merged into epic (`--no-ff` `0b22bbf41`) 2026-06-29 and pushed to origin. **Collapsed the planned 3-stage rollout into one pass** — no backward-compatible dual-shape window is needed because the route handler and its React senders ship in the same atomic Next.js deploy and the epic is not yet deployed. Real production change: `handleOrderLink` takes `newOrders: { id, category }[]` and resolves each order in a single `getOrderById(id, category)` call (the `'new' ?? 'maintenance'` guess is deleted); `PUT /api/documents/[documentId]/link-orders` accepts only `{ linkedOrders: [{ id, category }] }`; both senders (`LinkOrdersModal`, intake submission via `useSubmissionProcessor`) send the category, with `UpdateOrderForm` persisting `selectedExistingOrderCategory` on existing-order selection. Document `linkedOrderIds` storage and the `/api/intakes/[id]/link-orders` route are unchanged. Manually verified end-to-end (both senders) on maintenance order `010w` — payload `{ linkedOrders: [{ id: "6a3c6976b9900714c97f552c", category: "maintenance" }] }` confirmed; testing notes in `docs/agomez/testing/MLID-2628.md`. Branch kept. Full plan: `docs/agomez/plans/MLID-2628-D5-T1-link-orders-category-passthrough.md`. |

**D5 PR to develop:** Folds into the single epic → develop PR (now feature-complete — see Current State).

### MLID-2628 — Criteria & Implementation (D5-T1, FS)

**Context:** `handleOrderLink` (in `apps/web/services/notifications/documentNotification.ts`) resolves each linked order with a cross-collection guess — `(await getOrderById(id, 'new')) ?? (await getOrderById(id, 'maintenance'))`. Because `getOrderById` runs the enriched aggregation pipeline (not a cheap `findOne`), a maintenance order costs **two** full aggregations. The order category is already known at every UI sender and discarded at the request boundary. This task threads the category end-to-end so the order resolves in a single `getOrderById(id, category)` call and the guess is deleted. It is forward-compatible with the future "merge `neworders` + `maintenanceorders`" direction — the helper still benefits from being handed the category.

**Criteria:**

- Linking a maintenance order resolves with a single `getOrderById(id, 'maintenance')` call (no `neworders` probe first); a new order resolves with a single `getOrderById(id, 'new')` call.
- The `'new' ?? 'maintenance'` fallback in `handleOrderLink` is removed; an order not found for the supplied category logs the existing "order not found" warning and skips (no cross-collection retry).
- `PUT /api/documents/[documentId]/link-orders` request body becomes `{ linkedOrders: Array<{ id, category: 'new' | 'maintenance' }> }`; `category` is validated server-side; ids keep the 24-char hex validation; malformed entries return 400.
- Both senders (patient Documents tab `LinkOrdersModal`, document-intake submission) send the category.
- Document `linkedOrderIds` storage stays a bare `string[]` (unchanged).
- The unlink path (`handleOrderUnlink`) and `/api/intakes/[id]/link-orders` are untouched.
- Existing new-orders link + propagation behavior is unchanged.

**Implementation:** Staged 1 → 2 → 3 (server accepts category, backward compatible → migrate both senders → remove legacy shape + delete the guess). Full step-by-step plan, files affected, and tests in `docs/agomez/plans/MLID-2628-D5-T1-link-orders-category-passthrough.md`.

**Open question:** confirm a single intake submission cannot create a mix of new + maintenance orders; if it can, source each created order's category from the creation result rather than the intake's `selectedType`.

---

## Task List

### Deliverable 0: Retrofit fixes for new orders (7 SP — ships off `develop`)

- MLID-2391 — [D0-T1] Reassignment cascades notifications to new assignee (neworders) (3 SP)
- MLID-2413 — [D0-T2] Order notification badge — counter mismatch on document unlink (4 SP)

### Deliverable 1: Maintenance orders general table (2 SP)

- MLID-2447 — [D1-T1] Display unread notification badge on maintenance orders table (2 SP)

### Deliverable 2: Maintenance order detail — Documents tab (3 SP)

- MLID-2448 — [D2-T1] Documents tab notifications on maintenance order detail (3 SP)

### Deliverable 3: Maintenance order detail — History tab (2 SP)

- MLID-2449 — [D3-T1] Show document-upload log entries on maintenance order History tab (2 SP)

### Deliverable 4: Maintenance event triggers (11 SP)

- MLID-2450 — [D4-T1] Propagate notifications to maintenance orders on intakes upload (3 SP)
- MLID-2451 — [D4-T2] Propagate notifications to maintenance orders on e-order/fax upload (1 SP)
- MLID-2452 — [D4-T3] Propagate notifications when a document is linked to a maintenance order (2 SP)
- MLID-2453 — [D4-T4] Delete notifications when a document is unlinked from a maintenance order (2 SP)
- MLID-2454 — [D4-T5] Re-target notifications when a maintenance order is reassigned (3 SP)

### Deliverable 5: Link-orders category pass-through (3 SP)

- MLID-2628 — [D5-T1] Link-orders category pass-through — remove cross-collection guessing in `handleOrderLink` (3 SP)

---

## Epic Reference

- **Jira**: [MLID-2417](https://localinfusion.atlassian.net/browse/MLID-2417) — "Orders: Document Notifications V2 - Maintenance & Order Linking"
- **Total Story Points**: 28 (D0: 7 · D1: 2 · D2: 3 · D3: 2 · D4: 11 · D5: 3)
- **Epic Branch**: `epic/MLID-2417-document-notifications-maintenance` (for D1–D5; D0 branches off `develop` directly)
- **Deliverables**: 6 (D0–D5)
- **Feature flag**: `FeatureFlag.ORDER_DOCUMENT_NOTIFICATIONS` (reused — no parallel flag)

---

## Non-Code Tasks

| Task ID | Summary | Status |
|---------|---------|:------:|
| — | None at present. | — |
