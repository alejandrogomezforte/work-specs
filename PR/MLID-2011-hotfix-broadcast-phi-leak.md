# [MLID-2011] Hotfix: Drop `patientId` from `documentAdded` SignalR broadcast

## Summary

- The `documentAdded` SignalR event was being fanned out via `signalr.broadcastToAll`, which delivers to **every authenticated browser session** — not just the assigned user of the order. The payload included `patientId`, leaking a PHI-adjacent identifier far beyond the per-user authz scope enforced on the REST side.
- Fix: remove `patientId` from the broadcast payload. Existing client consumers (`useOrderNotificationCounts`, `useDocumentsTabNotifications`) ignore the payload and re-fetch via REST on event, where per-user authz already applies — so trimming the payload is a no-op for the client and closes the leak.
- Added a regression-guard test that asserts `patientId` is never present in the broadcast payload.

**Branch:** `hotfix/MLID-2011-broadcast-phi-leak` → `epic/MLID-2011-order-document-notifications`

**Source:** Code-review bot finding ("PHI leak in SignalR broadcast payload").

---

## Changes

| File | Change |
|------|--------|
| `apps/web/services/notifications/documentNotification.ts` | Removed `patientId` from the `documentAdded` broadcast payload + comment explaining why. `patientId` is still used internally to look up orders. |
| `apps/web/services/notifications/documentNotification.test.ts` | Updated payload assertion + added explicit `expect(payload).not.toHaveProperty('patientId')` regression guard. |
| `docs/signalr/order-documents-signalr-notifications.md` | Updated SignalR doc (impl example + events table) to reflect PHI-minimized payload. |

### NOT touched

- Client SignalR handlers — they already ignore the payload and trigger a REST re-fetch, so no client change was needed.
- The `NotifyDocumentCreatedParams` interface — `patientId` is still required as **input** to `notifyDocumentCreated` (used for order lookup); it just no longer appears in the **outbound** broadcast.
- The `notification:read` broadcast — `userId` in that payload is the session user's id, not a target id; no PHI concern, left untouched to keep this hotfix minimal.

---

## Verification

| Check | Result |
|-------|--------|
| `documentNotification.test.ts` | 22/22 passed (1 updated + 1 new regression test) |
| TypeScript (`tsc --noEmit`) | Clean |
| Backend-only diff — no UI to verify | n/a |

---

## Test Plan

- [ ] Upload a document for a patient with active assigned orders → verify the badge count updates in real time on the assigned user's browser (functional regression check).
- [ ] Open browser DevTools on a non-assigned user's session and inspect the SignalR `documentAdded` frame → confirm payload is `{ documentId, orderId, source, category }` (no `patientId`).
- [ ] Confirm the recipient's documents tab still refetches and shows the new notification badge after upload (the re-fetch path is unchanged).
