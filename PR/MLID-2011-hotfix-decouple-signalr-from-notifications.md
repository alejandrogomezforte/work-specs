# [MLID-2011] Hotfix: Decouple SignalR init from notification creation

## Summary

- `notifyDocumentCreated` has two responsibilities: (1) writing a row to `notifications` (durable, drives badge counts and unread state across reloads) and (2) emitting a `documentAdded` SignalR broadcast (a courtesy real-time push). Until now, `getSignalRService()` was awaited at the **top of the function, before** the per-order loop. Any throw there — e.g. a Key Vault lookup failing — was caught by the outer `try/catch` and silently aborted the entire function: zero rows in `notifications`, zero broadcasts.
- Stage hit exactly that on **2026-04-29**: `SIGNALR_KV_SECRET_NAME` was not set on the Container Apps' runtime config, so the SignalR service factory fell back to the default secret name (`signalr-primary-connection-string`), which doesn't exist in stage's Key Vault. Three documents were uploaded; three `Document notification: Unexpected error in notifyDocumentCreated` traces fired; **zero notifications were written**. Full investigation: `docs/agomez/postmortem/order-document-notifications-not-firing-on-stage.md`.
- The infra side (`SIGNALR_KV_SECRET_NAME=li-stage-signalr-primary-connection-string` set on the two stage Container Apps) is already deployed and verified.
- This PR is the **code-side hardening** so that any *future* SignalR misconfiguration can never silently break the badge feature again.

**Branch:** `hotfix/MLID-2011-decouple-signalr-from-notifications` → `develop`

**Source:** Production-shaped near-miss on stage; structural fragility in the trigger surfaced during the postmortem.

---

## Changes

### Trigger logic

| File | Change |
|------|--------|
| `apps/web/services/notifications/documentNotification.ts` | Reorder per-iteration work so `createNotification` runs **before** any SignalR I/O. SignalR init is now lazy (deferred until after the first notification has been attempted) and cached via a local `signalrInitAttempted` flag, so `getSignalRService()` runs **at most once per call** regardless of order count. The broadcast block is guarded on `signalr !== null`; on init failure, we log a single explicit `SignalR unavailable — broadcasts will be skipped, notifications were still written` line and continue writing notifications for every order. |

### Tests

| File | Change |
|------|--------|
| `apps/web/services/notifications/documentNotification.test.ts` | 5 new cases. **Ordering contract:** `calls createNotification before getSignalRService on the first iteration` (uses `mock.invocationCallOrder` — guards against future regression of upfront init); `initializes SignalR at most once per call regardless of order count`. **SignalR-down failure mode:** `still creates the notification` (single order); `creates a notification for every order` (N=3); `logs the SignalR-unavailable error exactly once (not per order)`; `never calls broadcastToAll for any order`. All 25 existing tests preserved. |

### NOT touched

- Call sites (`split-pdf/route.ts` branches A/B, `weinfuseUploader.ts:uploadPDFToWeInfuse`) — they still call `notifyDocumentCreated` the same way.
- Broadcast payload shape, event names, client listeners — unchanged.
- `notifications` schema, `createNotification`, `getOrdersByPatient` — unchanged.
- The SignalR module's public surface — local type uses `Awaited<ReturnType<typeof getSignalRService>>` rather than importing the internal `SignalRService` interface, to keep the hotfix surgical.
- No env-var or infra changes (Fix 1 already covered the runtime config gap).

---

## Why this shape

The notification row is **durable** — it persists across reload, drives badge counts, and is the source of truth for unread state. The SignalR broadcast is a **courtesy** live-push that lets already-open browsers refetch without a manual reload. If a user can recover state by hitting F5, the broadcast was a UX nicety; if no notification ever got written, the user has no way to recover.

So the per-iteration order should match that priority: **insert first, broadcast second**, and a broadcast-side failure must never block the insert. That's now structurally enforced.

### Why lazy + cached, instead of upfront init

Upfront `await getSignalRService()` (the previous shape) means the secondary I/O (KV roundtrip) blocks the primary work even on the happy path. With this change, the very first `createNotification` runs first; only then do we touch the SignalR service factory. The `signalrInitAttempted` guard (combined with the singleton inside `getSignalRService` itself) ensures we never pay more than one KV roundtrip per call, success or failure. With N=3 broken orders this means **1 retry + 1 error log**, not 3 + 3.

### Why a new dedicated log line

The previous failure mode produced an ambiguous `Document notification: Unexpected error in notifyDocumentCreated` from the outer `try/catch` — which could mean *anything*. The new explicit `SignalR unavailable — broadcasts will be skipped, notifications were still written` is unambiguous: dashboards and alerts can distinguish "real-time push is degraded but writes are healthy" from "the trigger blew up entirely." Past tense (`were still written`) is accurate because the log fires after at least one `createNotification` has already run.

---

## Verification

| Check | Result |
|-------|--------|
| `documentNotification.test.ts` | 30/30 passed (5 new + 25 existing) |
| TypeScript (`tsc --noEmit`) | Clean |
| Coverage on changed file | 96% statements, 95% branches, 100% functions |
| Backend-only diff — no UI behavior change | n/a |
| Stage live-test (Fix 1 already deployed) | QA verified end-to-end: order created, document uploaded, notification appeared, badge updated without page refresh |

---

## Test Plan

- [ ] **Happy path (regression):** upload a document for a patient with at least one active assigned order. Confirm a `notifications` row appears with `targetUserId` = order's assignee, `entities` containing both order and document, and a `SignalR broadcast sent { eventName: 'documentAdded' }` trace fires in App Insights. Documents tab badge updates without page refresh.
- [ ] **Multiple orders:** patient with N≥2 active assigned orders, upload one document. Confirm N notification rows are written and N broadcasts fire (one per order).
- [ ] **SignalR down (chaos test):** in a non-prod scratch env, temporarily clear `SIGNALR_KV_SECRET_NAME` on a test container, upload a document, confirm:
  - Exactly one `SignalR unavailable — broadcasts will be skipped, notifications were still written` log per upload (not per order).
  - All expected rows still appear in `notifications`.
  - No `documentAdded` broadcast fires.
  - Restore the env var afterward.
- [ ] **Order with no `assignedTo`:** existing skip behavior preserved — no notification, no broadcast.
- [ ] **`createNotification` throws on one order in a multi-order set:** that order's per-iteration `Failed to create document notification` log fires, but the next order still gets its notification (and broadcast if SignalR is healthy).
