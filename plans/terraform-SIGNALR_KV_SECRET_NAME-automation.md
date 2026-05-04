# Plan: Codify `SIGNALR_KV_SECRET_NAME` env var in Terraform

**Status:** Plan drafted — awaiting answers to open questions before editing
**Date:** 2026-04-29
**Owner:** Alejandro Gomez
**Related:** [order-document-notifications-not-firing-on-stage.md](../postmortem/order-document-notifications-not-firing-on-stage.md) — "Open follow-ups" item #1
**PE direction:** "Fix it yourself." Pointed at:
- `https://github.com/LocalInfusion/li-terraform`
- `https://github.com/LocalInfusion/li-call-processor/blob/develop/terraform/main.tf`
- `https://github.com/LocalInfusion/li-call-processor/blob/develop/scripts/apply-stage.sh`
- Requires stage VPN + Cloudflare token (PE sent Cloudflare invite, needs 2FA enabled).

## Why we're doing this

Fix 1 in the postmortem was applied manually via the Azure Portal: env var `SIGNALR_KV_SECRET_NAME=li-stage-signalr-primary-connection-string` added to both `li-stage-local-infusion-app` (web, revision `--0001297`) and `li-stage-local-infusion-worker` (worker, revision `--0000668`). That change exists **only at the running-revision level** — not in IaC. Any future Terraform re-apply will silently remove it and re-introduce the original bug (SignalR init throws → notifications loop never runs → zero rows in `notifications`).

We need to codify the env var in `li-call-processor/terraform` so the next `terraform apply` is a no-op for stage and a normal addition for prod.

## Repo split (important context)

| Repo | Responsibility |
|---|---|
| `li-terraform` (sibling folder `C:/Users/alejandro.gomez/Dev/li-terraform/`) | Owns the **Azure SignalR Service** + the KV secret. Module `modules/signalr/main.tf` writes the secret as `${var.name}-primary-connection-string` with `var.name = "li-{env}-signalr"` → stage produces `li-stage-signalr-primary-connection-string`, prod produces `li-prod-signalr-primary-connection-string`. `signalr_enabled = true` in both `stage.tfvars` and `prod.tfvars`. |
| `li-call-processor/terraform` (this repo) | Owns the two Container Apps that consume SignalR: `azurerm_container_app.app` (web) and `azurerm_container_app.worker`. **Currently has zero SignalR references.** This is the gap we're closing. |

## Why env var instead of renaming the KV secret

The code default in `apps/web/services/signalr/index.ts:16` is `'signalr-primary-connection-string'` (no env prefix). The Terraform-provisioned secret carries an env prefix (`li-{env}-`). Until the secret is renamed (separate PE follow-up — would obsolete this env var entirely), every environment needs the override. So this PR is the tactical fix; the rename is the long-term cleanup.

## Proposed changes (call-processor terraform only)

**Scope:** all changes inside `li-call-processor/terraform/`. No `li-terraform` change required for the tactical fix.

### 1. `terraform/variables.tf` — declare new variable

```hcl
variable "signalr_kv_secret_name" {
  description = "Key Vault secret name holding the SignalR primary connection string (provisioned by li-terraform as li-{env}-signalr-primary-connection-string)"
  type        = string
}
```

**No default on purpose** — force each `*.tfvars` to set it explicitly. Defaults silently mask config drift; that is literally what caused the original bug.

### 2. `terraform/main.tf` — add `env` block to both container templates

Insert into `azurerm_container_app.app` (around line 101, inside `template { container { … } }`):

```hcl
env {
  name  = "SIGNALR_KV_SECRET_NAME"
  value = var.signalr_kv_secret_name
}
```

Insert the same block into `azurerm_container_app.worker` (around line 591, inside `template { container { … } }`).

Plain `value` — this is the *name* of a KV secret, not a secret itself, so it does not need a `secret_name` reference.

### 3. `terraform/environments/stage.tfvars` — set value

```hcl
signalr_kv_secret_name = "li-stage-signalr-primary-connection-string"
```

### 4. `terraform/environments/prod.tfvars` — set value

```hcl
signalr_kv_secret_name = "li-prod-signalr-primary-connection-string"
```

(Inferred from the module template — see Open Question #1 below.)

## Apply path

`scripts/apply-stage.sh` is the deploy mechanism. Flow:

1. `az login` against subscription `61090de5-5e3b-4a4e-8b48-4a4f9a2339c5` (li-stage).
2. `terraform init` with remote state at `litfstatestage / tfstate / li-local-infusion-app.tfstate` (in resource group `li-tfstate-stage-rg`).
3. `terraform plan -var-file=environments/stage.tfvars -out=tfplan.stage`.
4. **Review plan output carefully** (see Open Question #2 below).
5. Type `yes` → `terraform apply tfplan.stage`.

**Prereqs:**
- Stage VPN connected.
- Cloudflare token (PE sent invite — needs 2FA enabled to obtain). Required because the Terraform config has Cloudflare DNS records (`cloudflare_record.app_cname` etc.) that re-evaluate even if no DNS changes are intended.
- Azure CLI installed & authenticated.
- Terraform CLI installed.

For prod: identical process via `scripts/apply-prod.sh` (subscription `04cb1c68-e911-423a-a794-5df89d268ffe`).

## Open questions — need answers before applying

### Q1. Verify the prod KV secret name

I inferred `li-prod-signalr-primary-connection-string` from the `li-terraform/modules/signalr/main.tf` template. Read-only verification (allowed per `feedback_az_cli_prod_readonly.md`):

```bash
az keyvault secret list --vault-name li-prod-kv \
  --query "[?contains(name,'signalr')].name" -o tsv
```

**Decision needed:** OK to run this myself read-only, or do you want to ask the PE first?

### Q2. Stage drift reconciliation

Stage's Container Apps already have `SIGNALR_KV_SECRET_NAME` set via the Portal (Fix 1 → revisions `--0001297` web, `--0000668` worker).

Two possible plan-output shapes when we run `terraform plan`:

- **Best case:** Terraform sees the value already on the live revision and shows **no diff** for env. Apply is a true no-op for envs.
- **Likely case:** Terraform's *state file* doesn't include the portal-set env var (because state was last refreshed before the portal change). Plan will show `+ env { name = "SIGNALR_KV_SECRET_NAME"; ... }` as if it were new. Apply will create a new revision that supersedes the portal one — same value, just owned by Terraform now. Functionally identical, just a revision rotation.

Either is benign. **Decision needed:** plan to scrutinize the diff together before typing `yes` — do not auto-apply.

### Q3. PR scope

Changes are fully within `li-call-processor`. No `li-terraform` PR needed for this tactical fix. The rename of the KV secret (which would obsolete `SIGNALR_KV_SECRET_NAME` everywhere) is a separate, riskier PE-owned task. **Decision needed:** confirm we're shipping just the call-processor PR for now.

### Q4. Variable shape — required vs default

I picked **no default** so each tfvars must set it explicitly. The repo's existing convention has defaults for most `*_key` vars, but those secrets share a common (non-prefixed) name across all envs. SignalR is the *exception* — the prefix differs per env. A default that "works for stage" silently breaks prod. **Decision needed:** confirm the no-default approach.

### Q5. Stage VPN + Cloudflare token

PE said:
- Stage VPN required to apply.
- Cloudflare invite sent — token will be needed (likely as `CLOUDFLARE_API_TOKEN` env var picked up by the cloudflare provider).
- 2FA must be enabled on Cloudflare.

**Decision needed:** confirm VPN is connected and Cloudflare 2FA + token are set up before the apply step. Won't be able to `terraform apply` without these.

### Q6. Worker container — confirm scope

Postmortem confirms the worker also runs the trigger (via `weinfuseUploader.ts:344`) and PE manually added the env var to the worker too. So worker definitely needs the same `env` block. Just flagging here so it doesn't get missed.

## Caveats / things not in scope

- **Renaming the KV secret to drop the env prefix** — separate cleanup, PE-owned. Would obsolete `SIGNALR_KV_SECRET_NAME` entirely, but requires migrating the secret value and coordinating across all consumers.
- **Hotfix code change** (`hotfix/MLID-2011-decouple-signalr-from-notifications`, commit `20ff9502`) — already merged-pending. Independent of this Terraform work. Even after this codifies the env var, the hotfix is still valuable: a future SignalR misconfiguration must never gate notification writes.
- **`li-terraform` changes** — not needed.

## Files to be modified (summary)

| File | Type of change |
|---|---|
| `terraform/variables.tf` | + new variable block |
| `terraform/main.tf` | + 2 `env` blocks (one in `app`, one in `worker`) |
| `terraform/environments/stage.tfvars` | + 1 line |
| `terraform/environments/prod.tfvars` | + 1 line |

## Resume checklist (for picking back up after a break)

1. Re-read this plan.
2. Answer Q1–Q5 above (decide / ask PE / verify state).
3. Make the 4 file edits.
4. Run `npm run format` (per repo dev guideline) — though Terraform isn't covered by Prettier, run `terraform fmt -recursive` from `terraform/` if available.
5. Connect to stage VPN, ensure `az login` is good, ensure Cloudflare token is available.
6. `cd terraform/ && bash ../scripts/apply-stage.sh` (or run from repo root — script is robust to either).
7. **Review the plan output. Don't `yes` blindly.**
8. After stage applies clean, repeat for prod with `scripts/apply-prod.sh`.
9. Verify in Azure Portal that the new revision on each container app has `SIGNALR_KV_SECRET_NAME` set.
10. Open PR per `feedback_pr_doc_not_gh_create.md` — write the PR draft to `docs/agomez/PR/`, don't `gh pr create` directly.
