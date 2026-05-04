# Postmortem: Why SignalR Was Failing to Connect

**Status**: All three bugs fixed and verified end-to-end on the epic branch. Infra rename for the Key Vault secret name still pending (PE).
**Dates**: 2026-04-23 (Bug #1 surfaced), 2026-04-24 (Bugs #2 and #3 surfaced and resolved)
**Epic**: MLID-2011 (Order Document Notifications)
**Related**:

- `docs/agomez/analysis/how-signalR-implementation-works.md`
- `docs/agomez/plans/MLID-2011-plan-progress.md`
- `apps/web/services/signalr/index.ts`
- `apps/web/services/keyVault/index.ts`
- `apps/web/app/api/signalr/negotiate/route.ts`

---

## Three bugs, three failure points

SignalR end-to-end real-time delivery has three independent moving parts, each with its own auth contract:

1. **Hop 1** — Browser → **our Next.js API** `POST /api/signalr/negotiate` (NextAuth-gated; reads connection string from Key Vault, mints client JWT).
2. **Hop 2** — Browser → **Azure SignalR client endpoint** `POST {endpoint}/client/negotiate?hub={hubName}` (uses the token from hop 1; subject to Azure's hub-name validation).
3. **Server-side broadcast** — Next.js server → **Azure SignalR REST API** `POST {endpoint}/api/hubs/{hubName}/:send?api-version=2022-06-01` (uses a separately-minted server JWT).

Each had a distinct bug. Each surfaced only after the previous one was fixed. Each was invisible to unit tests because the tests mocked the boundary that would have caught it.

---

# Bug #1 — Hop 1 returning 500 (Key Vault secret name)

## Symptom

In the browser console while running `npm run dev:web`:

```
Failed to load resource: the server responded with a status of 500 (Internal Server Error)
http://localhost:8080/api/signalr/negotiate
```

Consequence: the `SignalRProvider` at `apps/web/utils/context/signalr-context.tsx` never gets a `{ url, accessToken }`, `HubConnection.start()` is never called, and no SignalR events (`documentAdded`, `notification:read`) ever reach the browser. Real-time badge updates silently do not work — page refresh still shows correct state because the listeners only drive refetches.

---

## Root Cause

**Key Vault secret-name mismatch between code and infrastructure.**

The negotiate endpoint (`apps/web/app/api/signalr/negotiate/route.ts`) calls `getSignalRService()` → `getSecret(SIGNALR_SECRET_NAME)`. `SIGNALR_SECRET_NAME` is defined in `apps/web/services/signalr/index.ts:14-15`:

```ts
const SIGNALR_SECRET_NAME =
  process.env.SIGNALR_KV_SECRET_NAME || 'signalr-primary-connection-string';
```

Local `.env.local` had `AZURE_KEY_VAULT_URL=https://li-stage-kv.vault.azure.net/` and no `DEV_SECRET_signalr_primary_connection_string` fallback, so `getSecret()` in `apps/web/services/keyVault/index.ts` hit the stage Key Vault looking for the default-named secret:

```
az keyvault secret show --vault-name li-stage-kv --name signalr-primary-connection-string
→ SecretNotFound
```

The actual secret provisioned in `li-stage-kv` is:

```
li-stage-signalr-primary-connection-string
```

i.e. the Terraform that provisioned SignalR baked the environment into the secret name. Because no environment was setting `SIGNALR_KV_SECRET_NAME`, the lookup fell through to the default, missed, and `getSecret()` threw — caught by the `try/catch` in the negotiate route and re-emitted as a 500.

### Why this naming is an outlier

Every other secret in `li-stage-kv` uses a canonical, env-agnostic name — e.g. `abbvie-sftp-host`, `applicationinsights-connection-string`, `azure-openai-api-key`, `aws-access-key-id`. Because `li-stage-kv` and `li-prod-kv` are separate vaults, the environment is **already encoded by which vault you point `AZURE_KEY_VAULT_URL` at**. No prefix is needed.

The SignalR secrets break this convention. As a result, they are the only secrets in the codebase that require a per-env override (`SIGNALR_KV_SECRET_NAME`) to resolve.

---

## Why It Worked in Unit Tests but Not Locally

Unit tests in `apps/web/services/signalr/index.test.ts` mock `getSecret` and return a synthetic connection string, so the Key Vault name never matters in CI. The mismatch only surfaces against a real Key Vault.

---

## Fix Applied (Option B — Local Unblock)

Added one line to `apps/web/.env.local`:

```
SIGNALR_KV_SECRET_NAME=li-stage-signalr-primary-connection-string
```

After restarting `npm run dev:web`, the negotiate endpoint now:

1. Reads `SIGNALR_KV_SECRET_NAME` → `li-stage-signalr-primary-connection-string`.
2. `getSecret()` returns the stage connection string.
3. `parseConnectionString()` extracts `endpoint` + `accessKey`.
4. `negotiate(userId)` returns `{ url, accessToken }`.
5. The client `HubConnectionBuilder` upgrades to WebSocket against `wss://li-stage-signalr.service.signalr.net/...`.

This fix is **local-only**. It lives in `.env.local`, which is gitignored. It does not fix the underlying naming inconsistency for deployed environments.

---

## Recommended Follow-Up (Option A — Infra Fix, Not Done)

**Principal-engineer territory.** Rename (or duplicate) the Key Vault secrets in both `li-stage-kv` and `li-prod-kv` to drop the environment prefix:

| Current name                              | Proposed name                       |
| ----------------------------------------- | ----------------------------------- |
| `li-stage-signalr-primary-connection-string` | `signalr-primary-connection-string` |
| `li-prod-signalr-primary-connection-string`  | `signalr-primary-connection-string` |
| `li-stage-signalr-hostname`                  | `signalr-hostname`                  |
| `li-prod-signalr-hostname`                   | `signalr-hostname`                  |

Then:

- Remove the `SIGNALR_KV_SECRET_NAME` env override entirely from `apps/web/services/signalr/index.ts` (keep the constant, drop the `process.env.SIGNALR_KV_SECRET_NAME ||` side).
- Remove the per-env comment block at lines 11-13.
- Remove `SIGNALR_KV_SECRET_NAME` from every deployed env config and from this dev's `.env.local`.

This brings SignalR in line with every other secret in the vault and removes one class of misconfiguration.

---

## Detection Gap

Neither of the following would have caught this before a developer tried to run the app:

- CI / unit tests — mock the Key Vault entirely.
- Types / lint — `SIGNALR_KV_SECRET_NAME` is an optional string.
- Build-time check — `getSecret()` short-circuits to `''` during `NEXT_PHASE=phase-production-build`.

If we end up keeping the env-prefixed naming, consider a startup smoke test that calls `getSignalRService()` once during server boot and logs a loud error if it throws.

---

# Bug #2 — Hop 2 returning 400 (invalid hub name)

## Symptom

Once hop 1 started returning 200, the `@microsoft/signalr` SDK used the returned `{ url, accessToken }` to call hop 2 and got:

```
POST https://li-stage-signalr.service.signalr.net/client/negotiate?hub=li-hub&negotiateVersion=1
→ 400 Bad Request
Error: Failed to complete negotiation with the server: Error: Invalid value of 'hub'.
```

The client never reaches the WebSocket upgrade; `SignalRProvider.isConnected` stays `false`; no `documentAdded` / `notification:read` events ever arrive in the browser.

## Root Cause

**Azure SignalR requires hub names to be valid C# identifiers** — must start with a letter or underscore; only letters, digits, and underscores allowed. **Hyphens are not permitted.** The PE's default hub name `li-hub` violates this rule, and Azure rejects it at the hop-2 negotiate.

The name was hardcoded / defaulted in four places:

- `apps/web/services/signalr/index.ts:6` — default constant `'li-hub'`
- `apps/web/.env.local:64` — `SIGNALR_HUB_NAME=li-hub`
- `apps/web/services/signalr/index.test.ts` — 4 test-fixture uses of `'li-hub'`
- `apps/web/app/api/signalr/negotiate/route.test.ts` — 2 URL-string assertions including `?hub=li-hub`
- `docs/notifications-data-model.md:285` — sample broadcast URL

## Why Unit Tests Didn't Catch It

The tests mock `@/services/keyVault` and never make a real call to Azure SignalR. Azure's hub-name validation only runs on real traffic. Every test passes with `'li-hub'` because the mocks happily echo the string back.

## Why Deployed Environments Are Also Broken

The same hardcoded default is shipped in the build. If stage / prod App Service configs also set `SIGNALR_HUB_NAME=li-hub` (or don't set it at all and fall through to the default), **SignalR is broken everywhere, not just on a developer laptop**. The failure mode is silent: the UI continues to work because every listener hook re-fetches on mount; only real-time push is lost. Supervisors just see "notifications don't update until I refresh."

## Fix Applied (Option A — Repo-Wide)

Branch: `fix/MLID-2011-signalr-hub-name` (cut from `epic/MLID-2011-order-document-notifications`).

Changes:

1. `apps/web/services/signalr/index.ts:6` — default `'li-hub'` → `'lihub'`, with a comment documenting Azure's C# identifier rule.
2. `apps/web/services/signalr/index.test.ts` — all test-fixture uses of `'li-hub'` → `'lihub'`; added a new regression test asserting `service.getHubName()` matches `/^[A-Za-z_][A-Za-z0-9_]*$/` so no future change can regress the default back to a hyphenated name.
3. `apps/web/app/api/signalr/negotiate/route.test.ts` — URL-string assertions updated to `?hub=lihub`.
4. `docs/notifications-data-model.md:285` — sample broadcast URL updated.
5. This postmortem.

`.env.local` (gitignored) also has `SIGNALR_HUB_NAME=lihub` locally, which is what unblocked real-time events in the browser.

## Infra Follow-Up (PE / Ops)

The `lihub` default only wins in environments that either:

- Do **not** set `SIGNALR_HUB_NAME` (default applies), or
- Set `SIGNALR_HUB_NAME=lihub` explicitly.

Anywhere in App Service / Kubernetes / Terraform-managed env config that currently sets `SIGNALR_HUB_NAME=li-hub`, that override must be removed or changed to `lihub`. Otherwise the deployed env keeps failing hop 2 even after this code change lands. **Principal-engineer action required on all deployed environments.**

## Detection Gap

Identical gap to Bug #1:

- Unit tests mock Azure — never exercised the validator.
- Lint / types are silent on string-constant values.
- Build has no connectivity check.

Same recommendation: a boot-time smoke test that issues a single no-op hop-2 call against Azure SignalR and fails the pod readiness probe on a 4xx.

---

# Bug #3 — Server-side broadcast returning 401 (audience mismatch)

## Symptom

After Bugs #1 and #2 were fixed, the WebSocket connection from the browser to Azure SignalR was fully established (hop 1 + hop 2 working). But when a user clicked **Mark as Read** on a document notification:

```
PATCH /api/notifications/<id>/read → 500 Internal Server Error
{
  "error": "Failed to mark notification as read",
  "details": "SignalR broadcast failed: 401 Unauthorized"
}
```

Server logs:

```
[ERROR] SignalR broadcast failed {
  eventName: 'notification:read',
  hubName: 'lihub',
  status: 401,
  statusText: 'Unauthorized'
}
```

The notification was correctly written to MongoDB, but the side-effect broadcast call to Azure SignalR's REST `:send` endpoint was rejected with 401, causing the route handler to throw. End users saw a 500 even though the read receipt was already persisted.

## Root Cause

**The JWT `aud` (audience) claim used for server-to-Azure-SignalR REST calls did not match what `api-version=2022-06-01` requires.**

In `apps/web/services/signalr/index.ts`, the helper that mints server tokens was using only the hub-base path as audience:

```ts
function generateServerTokenForHub(hubName: string): string {
  const audience = `${endpoint}/api/hubs/${hubName}`;  // ← missing /:send
  return jwt.sign({ aud: audience }, accessKey, { expiresIn: TOKEN_EXPIRY });
}
```

While the actual fetch URL was:

```
{endpoint}/api/hubs/lihub/:send?api-version=2022-06-01
```

Azure SignalR's REST API for this version validates that the JWT's audience equals the **full request URL path without query string**. Older API conventions accepted just the hub-base URL; `2022-06-01+` is stricter. The `/:send` suffix had to be part of the audience.

This is the same authoring author as MLID-2080 (`broadcastToAll`); the audience pattern was carried over from an older Azure SignalR example without re-checking the version-specific requirement.

### Why client-side connections still worked

Hop 2 (browser ↔ Azure SignalR) uses a **different** JWT — minted by `generateAccessToken()` with `aud = {endpoint}/client/?hub={hubName}`. That audience matched the client-endpoint URL exactly, so client connections succeeded. The 401 was specific to **server-to-Azure** REST calls only. From the browser's point of view, "the WebSocket is open" — but no event ever arrived because the server side was being rejected before the broadcast left the box.

## Why Unit Tests Didn't Catch It

`apps/web/services/signalr/index.test.ts` mocks `global.fetch`. The fetch mock returns whatever the test wants — Azure's real audience validator never ran. The existing audience assertion was even encoding the *wrong* expectation:

```ts
expect(decoded.aud).toBe(`${endpoint}/api/hubs/${hubName}`); // wrong
```

So the test was passing because the implementation and the assertion were consistent with each other — but neither matched Azure's contract.

This is the third instance in this saga where unit-test mocks let a contract bug ship: a reminder that mocks encode an *assumption about the boundary*, not the boundary itself. Real integration with the boundary is the only thing that catches contract drift.

## Fix Applied (Repo-Wide)

Branch: `fix/MLID-2011-signalr-broadcast-audience` (from `epic/MLID-2011-order-document-notifications`).

Changes:

1. `apps/web/services/signalr/index.ts`:
   - Build `broadcastPath = ${endpoint}/api/hubs/${hubName}/:send` once.
   - Use it as **both** the fetch URL base and the JWT `aud` claim.
   - Rename the helper `generateServerTokenForHub(hubName)` → `generateRestApiToken(audience)` so its purpose (audience-driven token generation for any REST API URL) is explicit.
   - Capture Azure's response body (`response.text()`) into `bodyText` on the error log so any future broadcast failure shows Azure's actual rejection reason without a code round-trip.
   - Remove the unused public `generateServerToken()` from the `SignalRService` interface (no production caller existed; only `broadcastToAll` used the internal helper).

2. `apps/web/services/signalr/index.test.ts`:
   - **RED → GREEN**: updated the audience assertion to expect the full `${endpoint}/api/hubs/${hubName}/:send` path. The previous assertion was encoding the buggy contract; the new one matches Azure's actual contract.
   - New regression test asserting `bodyText` from Azure's response is included in the `logger.error` call on failure.
   - Updated the older "no info log on failure" mock to include `text: jest.fn()...` so the new `await response.text()` doesn't throw on the simpler error tests.

3. `apps/web/app/api/signalr/negotiate/route.test.ts`:
   - Dropped the `generateServerToken: jest.fn()` line from the `getSignalRService` mock object (no longer on the interface).

## End-to-End Verification (2026-04-24)

After the fix landed on the epic branch and the dev server was restarted:

| Checkpoint | Result |
| --- | --- |
| Browser: `PATCH /api/notifications/<id>/read` | **200** |
| Server log: `[INFO] SignalR broadcast sent { eventName: 'notification:read', hubName: 'lihub', status: 202, payload: { ... } }` | ✅ |
| Browser DevTools → Network → `wss://...` Messages tab | Inbound frame `{"type":1,"target":"notification:read","arguments":[{ ... }]}` ✅ |
| UI on the same tab | "New" chip removed, "Mark as Read" button hidden, Documents tab badge decremented (6 → 5) ✅ |

The broadcast plumbing exercised here (`broadcastToAll`) is the same path used by `notifyDocumentCreated` for `documentAdded`. Validating one validates both.

## Detection Gap

Same shape as Bugs #1 and #2:

- Unit tests mock fetch — Azure's audience validator never ran.
- The audience assertion in the test was encoding the same buggy expectation as the code.
- No integration / smoke test against real Azure exists.

Recommendation: a single boot-time / health-check broadcast that issues a no-op `:send` against Azure SignalR. If it returns anything other than 202, fail readiness. Catches all three bug classes at once.

---

## Timeline

| When       | What                                                                                                                                |
| ---------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| Earlier    | PE delivers SignalR service + negotiate route + `SignalRProvider` in a separate PR; unit tests pass.                                |
| Earlier    | Terraform provisions SignalR Key Vault secrets as `li-<env>-signalr-primary-connection-string`.                                     |
| 2026-04-23 | Alejandro starts the dev server to exercise the broadcast flow for MLID-2011. Browser reports `POST /api/signalr/negotiate` → 500. |
| 2026-04-23 | Traced through negotiate route → `getSignalRService()` → `getSecret()`. Confirmed via `az keyvault secret show` that the default-named secret does not exist in `li-stage-kv` and the actual secret carries the `li-stage-` prefix. |
| 2026-04-23 | Applied Option B for Bug #1: added `SIGNALR_KV_SECRET_NAME=li-stage-signalr-primary-connection-string` to `apps/web/.env.local`. Hop 1 now returns 200. |
| 2026-04-24 | Bug #2 surfaces: hop 2 returns `400 Invalid value of 'hub'`. Traced to Azure SignalR's C# identifier rule; `li-hub` contains a hyphen and is rejected. |
| 2026-04-24 | Local unblock: set `SIGNALR_HUB_NAME=lihub` in `apps/web/.env.local`. Real-time events start flowing.                                                     |
| 2026-04-24 | Cut `fix/MLID-2011-signalr-hub-name` from epic branch. Changed default to `lihub`, added regression test, updated tests + docs. Merged into epic with `--no-ff`. Pushed.                                                       |
| 2026-04-24 | Decided not to surface SignalR config knobs as env vars in deploy infra (don't yet know how to wire them in Azure). Cut `refactor/MLID-2011-signalr-inline-constants`, removed env overrides for `SIGNALR_HUB_NAME` / `SIGNALR_TOKEN_EXPIRY` / `SIGNALR_REST_API_VERSION`. Kept `SIGNALR_KV_SECRET_NAME` (Anatoliy's, MLID-2079). Merged into epic. |
| 2026-04-24 | Cut `refactor/MLID-2011-signalr-broadcast-logging` to add an `[INFO] SignalR broadcast sent` log inside `broadcastToAll`'s success path — single source of truth for "a broadcast left the server", covering all dispatchers. Merged into epic. |
| 2026-04-24 | Bug #3 surfaces during live QA: hop 1 + hop 2 working, but `Mark as Read` returns 500 because the server-side broadcast hits Azure's REST API and gets `401 Unauthorized`.                                                    |
| 2026-04-24 | Diagnosed audience mismatch — JWT `aud` claim missing the `/:send` suffix required by Azure SignalR REST API `2022-06-01+`. Cut `fix/MLID-2011-signalr-broadcast-audience`. Fixed audience, captured Azure's response body in error log, removed unused `generateServerToken`. Updated tests. Merged into epic. Pushed. |
| 2026-04-24 | End-to-end verified: Mark as Read on a `type: 'user'` notification fires the broadcast, Azure relays via WebSocket, listener refetches, UI updates (chip removed, badge 6 → 5). All three bugs closed.                       |
| TBD        | Infra follow-ups: (a) rename Key Vault secrets to drop the `li-<env>-` prefix; (b) (optional) re-introduce env overrides in deploy config if/when a coordinated change with deployed env is planned; (c) (optional) add a boot-time SignalR smoke test that catches all three bug classes pre-deploy. |
