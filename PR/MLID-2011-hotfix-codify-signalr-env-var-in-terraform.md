# [MLID-2011] Hotfix: Inject SignalR connection string from KV-backed Container App secret

## Summary

- The trigger `notifyDocumentCreated` needs the SignalR primary connection string at runtime. Previously it pulled it from Key Vault inside app code at startup, using `SIGNALR_KV_SECRET_NAME` (the Key Vault secret *name*) as input. The actual secret in KV carries an env prefix (`li-{env}-signalr-primary-connection-string`) that no other secret in the vault uses, so the env var was needed to bridge the naming gap.
- On **2026-04-29** stage hit a misconfiguration where this env var was unset on the two Container Apps. Three documents uploaded → zero notifications written. Full investigation: [`docs/agomez/postmortem/order-document-notifications-not-firing-on-stage.md`](../postmortem/order-document-notifications-not-firing-on-stage.md).
- **Fix 1** (manual) added the env var via the Azure Portal to stage's two Container Apps. That change lived only at the running-revision level — the next IaC re-apply would have silently dropped it. Prod was never patched.
- **This PR is the IaC-side hardening:** every future apply preserves the SignalR config on stage, and additionally **delivers Fix 1 to prod** (which the manual change never reached).

**Branch:** `hotfix/MLID-2011-codify-signalr-env-var-in-terraform` → `develop`

**Companion to:** [`hotfix/MLID-2011-decouple-signalr-from-notifications`](./MLID-2011-hotfix-decouple-signalr-from-notifications.md) (commit `20ff9502`) — code-side defense. With both landed, notification writes are protected at both the code layer (cannot be gated by SignalR misconfig) and the infra layer (config won't drift from source of truth).

---

## Approach (after PR review iteration)

The first commit (`81bfbe87`) wired `SIGNALR_KV_SECRET_NAME` as a plain env var, leaving the runtime KV fetch in app code. PR review (Ruslan) flagged that this is the only secret in the project that resolves at app-runtime instead of at terraform-apply time — every other secret in `terraform/main.tf` (mongodb-uri, openai-api-key, slack-bot-token, …) uses the project's standard pattern: read the value from KV via a `data.azurerm_key_vault_secret`, expose it as a Container App `secret { }`, and inject it into the container as an env var.

Commit `a16294c5` refactors to that standard pattern. The variable `signalr_kv_secret_name` is kept (Terraform still needs to know *which* KV secret to read), but the env var the application sees is now `SIGNALR_CONNECTION_STRING` — the full connection string, not a lookup key.

### Before vs after

```text
Before                                   After
──────                                   ─────
Terraform                                Terraform
  env { SIGNALR_KV_SECRET_NAME            data.azurerm_key_vault_secret.signalr_connection_string
        = "li-stage-signalr-..." }        secret { name = "signalr-connection-string"
                                                   value = data... }
                                          env { SIGNALR_CONNECTION_STRING
                                                secret_name = "signalr-connection-string" }

App code                                 App code
  getSecret(                              process.env.SIGNALR_CONNECTION_STRING
    process.env.SIGNALR_KV_SECRET_NAME)   (one read, no SDK call, no managed-identity hop)
  // round-trip to KV at startup
```

---

## Changes

### Terraform — data source + container-app secret + env reference

| File | Change |
|------|--------|
| `terraform/main.tf` | New `data "azurerm_key_vault_secret" "signalr_connection_string"` reading the per-env KV secret. New `secret { name = "signalr-connection-string", value = data... }` block on **both** `azurerm_container_app.app` (web) and `azurerm_container_app.worker`. The previous `env { SIGNALR_KV_SECRET_NAME = var... }` blocks are replaced with `env { SIGNALR_CONNECTION_STRING, secret_name = "signalr-connection-string" }` on both apps. Worker is in scope because the JotForm intake path (`apps/web/services/jobs/definitions/jotFormSubmission/weinfuseUploader.ts:344`) also calls `notifyDocumentCreated`. |
| `terraform/variables.tf` | `variable "signalr_kv_secret_name"` retained; description updated to reflect that the value is now read at terraform-apply time via the data source rather than passed through to the container as an env value. **No default** is intentional — both `stage.tfvars` and `prod.tfvars` must set the value explicitly so a missing override in one environment cannot silently inherit the other's secret name. |

### Per-environment values

| File | Change |
|------|--------|
| `terraform/environments/stage.tfvars` | `signalr_kv_secret_name = "li-stage-signalr-primary-connection-string"` |
| `terraform/environments/prod.tfvars` | `signalr_kv_secret_name = "li-prod-signalr-primary-connection-string"` |

### Application — read full connection string from env

| File | Change |
|------|--------|
| `apps/web/services/signalr/index.ts` | Drop the `getSecret` import and the `SIGNALR_SECRET_NAME` constant. `getSignalRService()` now reads `process.env.SIGNALR_CONNECTION_STRING` and throws a descriptive error if it is unset (fail loudly at first use rather than silently mint invalid tokens later). |

### Tests

| File | Change |
|------|--------|
| `apps/web/services/signalr/index.test.ts` | Drop the `@/services/keyVault` mock; set `SIGNALR_CONNECTION_STRING` in `beforeAll`. New test uses `jest.isolateModulesAsync` to verify the missing-env-var failure mode. |
| `apps/web/services/notifications/documentNotification.test.ts` | Refresh the mocked rejection error message to match the new "env var not set" failure. The test still asserts the original property: notification writes survive a SignalR factory failure. |

**Result:** 48 tests pass; coverage `signalr/index.ts` 99.37%, `notifications/documentNotification.ts` 96.32%.

### Docs / dev env

| File | Change |
|------|--------|
| `docs/signalr/signalr-integration.md` | Rewrite the env vars section and the failure-mode table row to describe `SIGNALR_CONNECTION_STRING` and the KV → data-source → Container-App-secret → env-var chain. |
| `apps/web/.env.local.example` | Add a `SIGNALR_CONNECTION_STRING` entry with format guidance. |

### NOT touched

- `li-terraform`. The SignalR module + secret naming convention there are correct as-is. Verified end-to-end (see "Mapping verification" below).
- KV secret naming. The long-term cleanup (renaming the KV secret to drop the env prefix) remains a separate PE-owned task. With this PR, that cleanup becomes a single tfvars change instead of a code+infra change.

---

## Files changed

| File | Lines |
|------|:-----:|
| `terraform/main.tf` | +27 -11 |
| `terraform/variables.tf` | +9 -0 (then -1 / +1 description) |
| `terraform/environments/stage.tfvars` | +4 |
| `terraform/environments/prod.tfvars` | +4 |
| `apps/web/services/signalr/index.ts` | +9 -8 |
| `apps/web/services/signalr/index.test.ts` | +25 -8 |
| `apps/web/services/notifications/documentNotification.test.ts` | +4 -4 |
| `apps/web/.env.local.example` | +5 |
| `docs/signalr/signalr-integration.md` | +7 -7 |

---

## Mapping verification

The KV secret name is produced by string concatenation across both repos. Traced end-to-end to confirm the values we set are correct:

```
li-terraform/modules/signalr/main.tf:41
  azurerm_key_vault_secret.signalr_primary_connection_string.name
  = "${var.name}-primary-connection-string"
                ▲
li-terraform/main.tf:672                          ← var.name comes from here
  module.signalr.name = "${local.prefix}-${var.environment}-signalr"
                              ▲                    ▲
li-terraform/locals.tf:2                           │
  local.prefix = "li" (default + explicit)         │
                                                   │
stage.tfvars: environment = "stage" ───────────────┘
prod.tfvars:  environment = "prod"  ───────────────┘
```

**Resolves to:**
- Stage: `li-stage-signalr-primary-connection-string`
- Prod: `li-prod-signalr-primary-connection-string`

**Live verification (stage, 2026-04-30):**
```
$ az keyvault secret list --vault-name li-stage-kv \
    --query "[?contains(name,'signalr')].name" -o tsv
li-stage-signalr-hostname
li-stage-signalr-primary-connection-string
```

---

## Apply impact

### Stage

The previous `SIGNALR_KV_SECRET_NAME` env var was set on stage via the Portal (Fix 1). With this PR that env var goes away and is replaced by `SIGNALR_CONNECTION_STRING` sourced from a new Container App secret. The next `terraform plan` will show:
- `+ secret { name = "signalr-connection-string" }` on web and worker
- `+ env { name = "SIGNALR_CONNECTION_STRING", secret_name = ... }` on web and worker
- `- env { name = "SIGNALR_KV_SECRET_NAME", value = ... }` removal (or noop if state file never captured the Portal change)

Apply rotates a revision on each Container App; the new revision boots with the connection string already in env, so SignalR initializes synchronously without a KV round-trip.

### Prod

Prod was never manually patched. Next `terraform apply` will *add* the secret + env for the first time. New revisions roll out, web + worker pick up the connection string on restart. **This effectively delivers Fix 1 to prod as part of this PR.**

### Prerequisites for apply (operator)

- Stage VPN.
- Cloudflare API token (the provider re-evaluates DNS resources on every apply).
- `az login` to the right subscription (`61090de5-…` for stage; `04cb1c68-…` for prod).
- Terraform CLI.

---

## Test Plan

- [ ] **Stage apply (deferred to operator):** `terraform plan -var-file=environments/stage.tfvars` shows the secret + env additions on the two Container Apps and the removal of the prior `SIGNALR_KV_SECRET_NAME` env block.
- [ ] **Stage post-apply verification:** in Azure Portal, both `li-stage-local-infusion-app` and `li-stage-local-infusion-worker` show `SIGNALR_CONNECTION_STRING` referenced from secret `signalr-connection-string`. App Insights continues to show **0** `Document notification: Unexpected error in notifyDocumentCreated` traces. Trigger a stage document upload and confirm the SignalR broadcast trace fires.
- [ ] **Prod apply (deferred to operator):** `terraform plan -var-file=environments/prod.tfvars` shows the same set of additions on prod's web + worker.
- [ ] **Prod post-apply verification:** trigger a document upload (via QA flow), confirm `notifications` rows are written for the matching active orders, confirm `SignalR broadcast sent { eventName: 'documentAdded' }` traces fire in App Insights.

---

## Risk assessment

- **Low blast radius.** Additive secret + env on two Container Apps; no resource recreation, no dependency reshuffling, no actual KV secret rotation.
- **Idempotent.** Re-applying after merge is safe.
- **Reversible.** Removing the secret + env via terraform restores the previous state; the prior code path is no longer present, so a rollback also requires reverting the application code change.
- **App-code change is gated by the env var being present.** If the secret somehow isn't injected, the app fails fast at first SignalR call with a descriptive error, and the notification-write path stays operational thanks to the companion code-side defense (`hotfix/MLID-2011-decouple-signalr-from-notifications`).
