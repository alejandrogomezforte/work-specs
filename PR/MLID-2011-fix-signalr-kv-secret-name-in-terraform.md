# [MLID-2011] Fix: Resolve SignalR KV secret name from APP_ENV with safe stage default

## Summary

- `apps/web/services/signalr/index.ts` defaulted `SIGNALR_SECRET_NAME` to the literal `'signalr-primary-connection-string'` — a value that does **not** match any real Key Vault secret in stage or prod (both are env-prefixed: `li-stage-signalr-primary-connection-string` / `li-prod-signalr-primary-connection-string`). Production therefore depended on `SIGNALR_KV_SECRET_NAME` being set on the Container App, which had only ever been added manually via the Azure Portal during the 2026-04-29 notifications outage. The next `terraform apply` would have silently dropped that override and re-introduced the bug. Full investigation: [`docs/agomez/postmortem/order-document-notifications-not-firing-on-stage.md`](../postmortem/order-document-notifications-not-firing-on-stage.md).
- This PR closes the gap from **two sides**:
  1. **Code-side** — the fallback default is now derived from `APP_ENV` and matches the real secret names in stage and prod. The `SIGNALR_KV_SECRET_NAME` env var is now optional (escape hatch for renames), not load-bearing.
  2. **Terraform-side** — `SIGNALR_KV_SECRET_NAME` is codified on both Container Apps (web + worker) so the manually-set Portal value is now backed by IaC and won't disappear on the next apply.

**Branch:** `fix/MLID-2011-signalr-kv-secret-name-in-terraform` → `develop`

**Supersedes:** an earlier abandoned attempt ([`MLID-2011-hotfix-codify-signalr-env-var-in-terraform.md`](./MLID-2011-hotfix-codify-signalr-env-var-in-terraform.md)) that tried to refactor the whole secret-injection pipeline to the project's standard `data.azurerm_key_vault_secret` + `secret {}` pattern. That approach was too invasive for the value it delivered, and the branch was deleted. This PR keeps the existing app-runtime KV fetch and only fixes the resolution logic.

**Companion to:** [`MLID-2011-hotfix-decouple-signalr-from-notifications`](./MLID-2011-hotfix-decouple-signalr-from-notifications.md) (already merged) — code-side hardening of the trigger so a SignalR misconfig can't silently swallow notification writes. With both landed, the notification path is defended at three layers: trigger logic (decoupling), runtime resolution (this PR's code change), and infrastructure (this PR's terraform change).

---

## Changes

### Code

| File | Change |
|------|--------|
| `apps/web/services/signalr/index.ts` | Extract resolution into a pure exported `resolveSignalRSecretName()`. Order: (1) `SIGNALR_KV_SECRET_NAME` env var if set (escape hatch); (2) `APP_ENV === 'prod'` → `'li-prod-signalr-primary-connection-string'`; (3) otherwise → `'li-stage-signalr-primary-connection-string'`. Module-load constant `SIGNALR_SECRET_NAME` now calls the helper instead of using the broken literal default. |

### Tests

| File | Change |
|------|--------|
| `apps/web/services/signalr/index.test.ts` | 4 new unit tests on `resolveSignalRSecretName`: env-var override (regardless of APP_ENV), `APP_ENV='prod'` branch, `APP_ENV='stage'` branch, both vars unset (local dev). Original 17 tests preserved. |

### Infrastructure

| File | Change |
|------|--------|
| `terraform/main.tf` | Add `SIGNALR_KV_SECRET_NAME` env var to both `azurerm_container_app.app` (web) and `azurerm_container_app.worker`. Value is interpolated as `"${var.prefix}-${var.environment}-signalr-primary-connection-string"`, matching the secret-naming convention used by `li-terraform`'s `signalr` module. |

### Documentation

| File | Change |
|------|--------|
| `docs/signalr/signalr-integration.md` | Rewrite the env-var section to document the new resolution order and the APP_ENV vs NODE_ENV rationale. Replace the prior "you must set `.env.local` to work locally" instruction with the new behavior (stage default applies if unset; the env var is now optional). |

### NOT touched

- `services/signalr/index.ts` public surface, `getSignalRService`, `parseConnectionString`, `generateAccessToken`, `broadcastToAll` — unchanged.
- Trigger code in `services/notifications/documentNotification.ts` — already hardened by the companion hotfix.
- Other Container Apps (`mcp`, `li-public-api`) — they don't import the SignalR service, no env var added.
- `apps/web/services/apiKeys/generateKey.ts:9` — has a related but separate APP_ENV bug (`=== 'production'` vs the actual deployed `'prod'`). Filed as **MLID-2331** for separate handling.

---

## Why this shape

### Why APP_ENV and not NODE_ENV

`NODE_ENV` is hardcoded by Terraform to `"production"` for **every** deployed Container App — both stage and prod. A `NODE_ENV`-based check would treat the two environments identically, which is exactly the discrimination problem we're trying to solve. `APP_ENV` is sourced from `var.environment` in the per-environment tfvars files (`stage` / `prod`), making it the only env-var signal in this codebase that actually distinguishes stage from prod. There's prior art for this pattern in `services/apiKeys/generateKey.ts` (though that file has a value-mismatch bug — see MLID-2331).

### Why default-to-stage when APP_ENV is unset

A wrong secret name in production fails **loud** at startup (Key Vault returns 404 on the missing secret, and the app surfaces it as a clear error in App Insights). The inverse failure mode — silently using prod credentials in a non-prod environment — would be much harder to detect and could leak production data into a stage deployment. Defaulting to the stage secret is the safer bias.

### Why both code-side fix AND terraform env var

Either alone would work. Together they form belt-and-suspenders:

- **Code-side alone:** prod comes up correctly even with no infra change. But the manually-set Portal value still drifts from IaC, and a future reader checking terraform sees no signal that this env var matters.
- **Terraform-side alone:** the env var is preserved across applies, but the code's literal default (`'signalr-primary-connection-string'`) remains a footgun for anyone setting up a new environment without remembering to wire the env var first.
- **Both:** intent is documented in IaC, prod can boot from code-defaults alone, and the env var becomes a documented escape hatch for future secret renames rather than a hidden requirement.

### Why a pure helper function instead of inline ternary

The new logic is non-trivial enough (three branches, env-var lookup) that testing it as part of `getSignalRService()`'s setup would require module-reload gymnastics. Extracting `resolveSignalRSecretName()` lets the four new unit tests exercise every branch directly without re-importing the module per test. The module-load constant `SIGNALR_SECRET_NAME` still resolves once at import time — same runtime semantics as before.

---

## Verification

| Check | Result |
|-------|--------|
| `services/signalr/index.test.ts` | 21/21 passed (4 new + 17 existing) |
| TypeScript (`tsc --noEmit` from `apps/web/`) | Clean |
| ESLint (changed `.ts` files) | Clean |
| Prettier (changed `.ts` and `.md` files) | Clean |
| `terraform fmt -check main.tf` | Clean |
| Coverage on `services/signalr/index.ts` | 99.41% statements / 100% branches / 88.88% functions |
| Backend / infra-only diff — no UI behavior change | n/a |
| Stage live behavior | Manually-set Portal `SIGNALR_KV_SECRET_NAME` already in place; this PR's terraform change is functionally a no-op for the running revision (same value), and the code change is exercised whenever `getSignalRService()` is first invoked. |

---

## Test Plan

- [ ] **Stage terraform plan dry-run:** run `scripts/apply-stage.sh` to the plan stage (do *not* apply), confirm the diff for `azurerm_container_app.app` and `azurerm_container_app.worker` shows `SIGNALR_KV_SECRET_NAME` as either no-op (if state already has it) or a `+` add (if state did not capture the manual Portal change). Either is acceptable; both result in the same running value.
- [ ] **Stage apply:** apply terraform. Container App revisions rotate. Confirm `az containerapp revision show --name li-stage-local-infusion-app` and `--name li-stage-local-infusion-worker` both show `SIGNALR_KV_SECRET_NAME=li-stage-signalr-primary-connection-string` on the new revision.
- [ ] **Stage smoke:** trigger a document upload that fans out a SignalR broadcast (split-pdf or weinfuseUploader path). Confirm a `SignalR broadcast sent` info log fires in App Insights and the Documents tab badge updates without a refresh in an open browser.
- [ ] **Prod terraform plan dry-run:** run `scripts/apply-prod.sh` plan stage. Expect a `+` diff adding `SIGNALR_KV_SECRET_NAME=li-prod-signalr-primary-connection-string` on both prod Container Apps (the manual Portal change was never applied to prod). Review carefully before approving.
- [ ] **Prod apply:** apply when ready, then repeat the smoke test above against a prod-eligible test account.
- [ ] **Code-side regression (no env var):** in a scratch local env, unset `SIGNALR_KV_SECRET_NAME` and `APP_ENV`, run `npm run dev:web`, hit any page that initializes the SignalR service, confirm it attempts to fetch `li-stage-signalr-primary-connection-string` (visible in the `Error retrieving secret` line if local KV access is not configured — the *secret name* in the error is the assertion).
