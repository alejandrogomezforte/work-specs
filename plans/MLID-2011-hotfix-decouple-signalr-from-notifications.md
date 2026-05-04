# MLID-2011 Hotfix — Decouple SignalR Init from Notification Writes

## Context

Postmortem: `docs/agomez/postmortem/order-document-notifications-not-firing-on-stage.md`.

A stage misconfiguration (env var `SIGNALR_KV_SECRET_NAME` not set on the Container Apps) caused `getSignalRService()` to throw on every invocation. In `apps/web/services/notifications/documentNotification.ts`, the call to `getSignalRService()` happens **before** the per-order loop (L58). A throw there is swallowed by the outer `try/catch` (L105) and **the loop never runs**, so `createNotification` is never called.

**Symptom:** zero rows written to `notifications` for three documents uploaded today, despite the trigger being invoked correctly.

The infra fix (Fix 1) has been applied and verified — SignalR is now operational on stage. This plan covers Fix 2: a structural code hotfix that prevents *any* future SignalR misconfiguration from silently breaking the primary notification-write path.

## Goal

Make the notification-creation path independent of SignalR availability. A SignalR failure should at most lose the real-time push (`documentAdded` broadcast) — it must never block the `createNotification` insert.

The badge / unread-count contract is the *primary* feature obligation; SignalR is a secondary delivery optimization. The code should reflect that priority.

## Non-goals

- No change to call sites (`split-pdf/route.ts`, `weinfuseUploader.ts`). They still call `notifyDocumentCreated` the same way.
- No change to broadcast payloads, event names, or client listeners.
- No change to the `notifications` schema or `createNotification` function.
- No env-var changes, no infra changes (Fix 1 already covered the runtime config gap).

## Approach

**Per-iteration: notification first, SignalR second. Lazy-init SignalR once and cache across orders.**

Reasoning (per user feedback during implementation): the `notifications` row is durable and load-bearing — it drives badge counts, lets users reload to see unread state, and persists across SignalR outages. The SignalR broadcast is a courtesy live-push: nice to have, but not the primary obligation. So within each iteration, `createNotification` runs **before** SignalR is even touched. SignalR init is lazy (deferred until after the first notification has been attempted) and cached (a single attempt covers all orders in the call).

```ts
const sourceLabel = SOURCE_LABELS[source];

let signalr: Awaited<ReturnType<typeof getSignalRService>> | null = null;
let signalrInitAttempted = false;

for (const order of orders) {
  const { assignedTo } = order;
  if (!assignedTo) continue;

  const orderId = String(order._id);

  // 1) Primary: write the notification.
  try {
    await createNotification({ ... });
  } catch (error) {
    logger.error('Document notification: Failed to create document notification', { orderId, documentId, error });
  }

  // 2) Secondary: live push via SignalR. Init lazily, at most once per call.
  if (!signalrInitAttempted) {
    signalrInitAttempted = true;
    try {
      signalr = await getSignalRService();
    } catch (error) {
      logger.error(
        'Document notification: SignalR unavailable — broadcasts will be skipped, notifications were still written',
        { documentId, error }
      );
    }
  }

  if (signalr) {
    try {
      await signalr.broadcastToAll('documentAdded', { documentId, orderId, source, category });
    } catch (error) {
      logger.error('Document notification: Failed to broadcast documentAdded event', { orderId, documentId, error });
    }
  }
}
```

### Why this shape

- **Notification insert is unconditionally first.** Even on the very first order, `createNotification` runs before any SignalR I/O. The primary contract is honored before any "courtesy" work begins.
- **One log line per upload on SignalR init failure**, not N (the `signalrInitAttempted` guard ensures init runs at most once even if it fails). No log spam if SignalR is broken.
- **Past-tense log message** (`notifications were still written`) is accurate because at least one createNotification has already been attempted before this log fires.
- **Broadcasts still fire** when SignalR is healthy — same cardinality as today (one per order × document).
- **No new public type surface.** Uses `Awaited<ReturnType<typeof getSignalRService>>` to type the local variable — avoids exporting the internal `SignalRService` interface from the SignalR module.

### Alternatives considered (rejected)

- **Init SignalR upfront, before the loop.** Conceptually mixes the order: the secondary I/O (KV roundtrip) blocks the primary work. If init takes 200ms, every notification is delayed even on the happy path. Discarded after user feedback during implementation.
- **Lazy-load SignalR inside each iteration's broadcast `try` (no caching).** N retries per upload if SignalR is broken (one per order × KV roundtrip). Wasted I/O and log noise. Discarded.
- **Move SignalR call entirely after the loop and aggregate broadcasts.** Changes broadcast cardinality (today: one per order, PHI minimized; aggregating would either send fewer events or change payload shape). Out of scope for a hotfix.

## File changes (single file)

| File | Change |
|---|---|
| `apps/web/services/notifications/documentNotification.ts` | Replace direct `await getSignalRService()` at L58 with a `try/catch`-wrapped init that sets `signalr` to either the service or `null`. Guard the per-iteration broadcast on `signalr !== null`. |

No other production files change. No type changes. No new dependencies.

## TDD plan

Test file: `apps/web/services/notifications/documentNotification.test.ts` (already exists — covers the trigger).

### Red — new failing tests to add first

1. **`when getSignalRService initialization throws … with one active order … still creates the notification`** — `getSignalRService` mocked to throw; assert `createNotification` is still called for the single order.
2. **`… with multiple active orders … creates a notification for every order`** — N=3 orders, init throws; assert `createNotification` called N times.
3. **`… logs the SignalR-unavailable error exactly once (not per order)`** — N=3 orders, init throws; assert exactly one error log mentioning "SignalR unavailable".
4. **`ordering contract … calls createNotification before getSignalRService on the first iteration`** — using `mock.invocationCallOrder`, assert createNotification's call order is less than getSignalRService's.
5. **`ordering contract … initializes SignalR at most once per call regardless of order count`** — N=3 orders, healthy SignalR; assert `getSignalRService` called exactly once.
6. **Existing per-broadcast error tests** — kept; verify they still pass after the structural change (regression guard).

### Green — implementation

Apply the code change described under "Approach". Run the new tests; they pass.

### Refactor

- Confirm the existing happy-path test (SignalR succeeds, broadcasts fire) still passes.
- Confirm the existing FF-off and no-orders early returns still pass.
- Run `npm run lint:fix` and `npm run types:check` from `apps/web/`.

## Logging contract change

| Scenario | Before | After |
|---|---|---|
| SignalR init throws | 1× outer-catch `Unexpected error in notifyDocumentCreated`, **0** notifications written | 1× explicit `SignalR unavailable — broadcasts will be skipped, notifications will still be written`, **N** notifications written |
| SignalR init succeeds, `broadcastToAll` throws on order X | 1× per-broadcast catch | unchanged |
| `createNotification` throws on order X | 1× per-create catch | unchanged |
| Happy path | `SignalR broadcast sent` per order | unchanged |

The new log message is intentionally specific so dashboards/alerts can distinguish "SignalR is broken but notifications work" from the old, ambiguous "outer catch caught something."

## Branching & rollout

- Branch off `origin/develop` (per memory: never rebase, never merge `origin/develop` directly into a feature branch — checkout-pull local develop first).
- Branch name: `hotfix/MLID-2011-decouple-signalr-from-notifications` (matches naming of recent MLID-2011 hotfixes: `hotfix/MLID-2011-notification-read-authz`, `hotfix/MLID-2011-broadcast-phi-leak`).
- Commit format: `[MLID-2011] - fix(notifications): decouple SignalR init from notification creation`. Layered detailed body per memory.
- Per memory: no `gh pr create`. Write a PR draft to `docs/agomez/PR/MLID-2011-hotfix-decouple-signalr-from-notifications.md` and let the user open the PR manually.

## Verification after merge

Once the hotfix is on `develop` and deployed to stage:

1. Confirm existing happy-path: trigger an upload, see the `notifications` row + `SignalR broadcast sent` trace.
2. Simulate the failure path **only in a non-prod scratch environment** (do not break stage SignalR): temporarily unset `SIGNALR_KV_SECRET_NAME` on a test container, upload a document, confirm:
   - 1× `SignalR unavailable …` trace per upload (not per order, not outer-catch).
   - Row(s) appear in `notifications`.
   - No `documentAdded` broadcast.
3. Restore the env var.

## Out of scope

- Restoring SignalR after a confirmed outage (still infra-owned).
- Renaming the stage KV secret (long-term cleanup, PE-owned).
- Doc fix in `docs/signalr/order-documents-signalr-notifications.md` §4.2 (`findOrdersForPatient` → `getOrdersByPatient` rename).

## Open questions before implementation

- **Jira ticket:** the recent MLID-2011 hotfixes appear to have been merged without dedicated sub-tasks (PR titles reference the epic ID directly). Confirm whether this hotfix needs its own Jira issue or piggybacks the epic.
- **Test setup:** the existing `documentNotification.test.ts` already mocks `getSignalRService` and `createNotification`. The new tests should reuse the same mocking strategy for consistency. Will confirm during the red phase.
