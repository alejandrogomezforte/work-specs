# [MLID-2366] chore(terraform): scope Spruce SMS whitelist to per-environment, sync stage

## Summary

- The terraform variable `spruce_appointment_reminder_target_criteria` (which wires `APPOINTMENT_REMINDER_TARGET_CRITERIA` into the web and worker Container Apps) was set up with a default value that contained three staging-leaning test phone numbers. Production overrode this default to `"*"` (allow all) via `terraform/environments/prod.tfvars`, but staging had no override — so terraform's source-of-truth for staging was the three-number default, even though the actual staging Azure Portal had been manually updated to include additional QA whitelist numbers.
- This PR closes the gap from **two sides**:
  1. **Per-env scoping** — `terraform/variables.tf` default is now `""` instead of the three staging-leaning numbers. The variable is now strictly per-environment: prod opts in via `"*"` in `prod.tfvars`, stage opts in via the new override below. No environment inherits an implicit fallback that leaks test data.
  2. **Stage reconciliation** — `terraform/environments/stage.tfvars` now declaratively binds the variable to the 13-number CSV that QA needs for SMS testing in staging. This matches what's already running in Azure (manually applied during Stage 1) so the next `terraform apply` is a no-op for this env var rather than a destructive reset.

**Branch:** `feature/MLID-2366-terraform-stage-whitelist` → `develop`

**Background:** The ticket asked for QA SMS/fax numbers to be whitelisted in stage. Stage 1 was a direct Azure Portal update on 2026-05-13 to unblock Olha's testing. Stage 2 — this PR — backs that change with IaC so it survives the next deploy.

---

## Changes

### Infrastructure

| File | Change |
|------|--------|
| `terraform/variables.tf` | `spruce_appointment_reminder_target_criteria` default changed from `"7078817918,9176755175,2014030962"` (three staging-leaning numbers) to `""`. Description expanded to document the per-env contract: prod sets `"*"`, stage sets a CSV of QA numbers. The three original baseline numbers were preserved — they now live in `stage.tfvars` alongside the new QA numbers, where they semantically belong. |
| `terraform/environments/stage.tfvars` | New `spruce_appointment_reminder_target_criteria` override inserted between `intake_api_url` (line 43) and the `feature_flags` block (line 45), grouped with other external-service config. Value is the full 13-number CSV: original 3 baseline + 6 QA SMS numbers (already live on Azure stage portal since 2026-05-13) + 4 QA fax numbers. |

### NOT touched

- `terraform/main.tf` — env-block wiring already correct on both `azurerm_container_app.app` (web, line 277) and `azurerm_container_app.worker` (line 715). No structural change needed.
- `terraform/environments/prod.tfvars` — production stays `"*"` (allow all). Untouched.
- `spruce_appointment_reminder_allow_fnf` — defaults to `"false"`, which is the correct staging value. No override needed.
- Application code — none of the SMS code paths (`appointmentReminder`, `welcomeTextService`, `reverificationService`, `skyriziBaseService`, `jotFormSubmission`) needed changes. They already read `APPOINTMENT_REMINDER_TARGET_CRITERIA` at runtime from the env var; this PR only changes the value flowing in.
- Fax-sending code (`sendWeeklyProviderReport.ts` → `sendSpruceFax`) — does not consult this env var. The 4 fax numbers in the whitelist are documentation-only no-ops today and exist for future-proofing if a fax allowlist gate is ever added (out of scope for this ticket).

---

## Why this shape

### Why empty the default instead of just adding a stage override

Either alone would prevent the immediate problem (staging losing its QA numbers on the next apply). But the original default was structurally wrong: it contained content that only made sense for staging, sitting in a "shared default" position that any new environment (a hypothetical `dev.tfvars`, a future preview env) would inherit. That's a leaky abstraction. Making the default `""` forces every environment to opt in explicitly, mirroring the discipline that `prod.tfvars` already demonstrates. Behavioral impact on the two existing environments is zero because both now override.

### Why a "block all" default and not "allow all"

If a future environment gets added and forgets to override, an empty whitelist will block all outbound SMS in that environment. The application code's `isAllowedToSendByTargetCriteria` paths treat `""` as the safe default (block) — confirmed in `services/messages/welcomeTextService.ts`, `services/messages/reverificationService.ts:120`, `services/messages/skyrizi/skyriziBaseService.ts:61`, and the appointment-reminder + JotForm job handlers. The inverse — defaulting to `"*"` — would silently allow real outbound SMS in any new environment that forgot to override, which is the failure mode HIPAA-adjacent code shouldn't bias toward.

### Why bundle the variables.tf cleanup into the stage-sync commit

The two changes are the same intent: align terraform's model of "what value belongs in each environment" with reality. Splitting them into two commits would create an intermediate state where the default is empty and stage still has no override — i.e., an unintended block-all on stage. Bundling avoids that transient state in the git history.

---

## Verification

| Check | Result |
|-------|--------|
| `terraform init -reconfigure` (stage backend) | Success. Backend connected, all providers cached, no downloads. |
| `terraform validate` (local) | Skipped. Terraform 1.15 partial-backend wart — fails with "Missing required argument" on `container_name` / `storage_account_name` locally despite init succeeding. CI workflow runs validate in an environment that doesn't hit the bug; plan exercises the same parser, so plan succeeding is sufficient. |
| `terraform plan -var-file=environments/stage.tfvars` | **0 env-block changes** for `APPOINTMENT_REMINDER_TARGET_CRITERIA` on both web and worker. This is the correct outcome: Azure portal already has the 13-number string (manually applied during Stage 1), so this PR is a declarative reconciliation, not a behavior change. Pre-existing unrelated drift on `azurerm_container_app.worker` (`startup_probe.initial_delay: 1 → 0`) is acceptable and stays unaddressed. The plan ends with an unrelated Cloudflare provider auth error (no `CLOUDFLARE_API_TOKEN` set locally) — that's a tooling gap, not a config issue. |
| Live Azure stage state (`az containerapp show ... --query "...[?name=='APPOINTMENT_REMINDER_TARGET_CRITERIA']"`) | Confirmed: both `li-stage-local-infusion-app` (web) and `li-stage-local-infusion-worker` already have the full 13-number string. The PR's stage.tfvars value matches exactly. |
| TypeScript / Jest / ESLint / Prettier | n/a — no application code touched. |
| `terraform fmt -check` | Not configured in CI for this repo (no fmt step in `terraform-plan-apply.yml`). Manual review of the diff confirms HCL formatting matches surrounding lines. |
| CI gate | The repo's automated terraform validation/plan in `azure-deploy.yml` is currently commented out. There is no automated terraform check on this PR — local verification above is the only gate. |

---

## Test Plan

- [ ] **Read the diff** — exactly two files changed, both under `terraform/`. No application code. Confirm the 13-number string in `stage.tfvars` matches what's already on Azure stage portal.
- [ ] **Stage terraform plan (whoever owns deploys)** — run a stage plan from a workstation with `CLOUDFLARE_API_TOKEN` available. Confirm:
  - Zero changes for `APPOINTMENT_REMINDER_TARGET_CRITERIA` on `azurerm_container_app.app` and `azurerm_container_app.worker` (i.e., terraform sees no env-var diff — this proves the new tfvars value already matches Azure).
  - The pre-existing `worker.startup_probe.initial_delay` drift may show up; that's unrelated to this PR and was already there before.
- [ ] **Stage terraform apply** — when convenient, apply against stage to lock in the codified value. Confirm `az containerapp revision show` on the new revisions still shows the 13-number string.
- [ ] **Prod terraform plan dry-run** — run a prod plan. Expected: zero changes for `APPOINTMENT_REMINDER_TARGET_CRITERIA` (prod uses `"*"` via `prod.tfvars`, which this PR does not touch). Anything else for that env var would indicate a problem.
- [ ] **Stage smoke** — after stage apply, trigger an SMS-sending flow (e.g., an appointment reminder for a whitelisted patient phone) and confirm the message goes out. The application semantics should be unchanged because Azure was already running this value.

---

## Notes

- **No Jira transition.** Per team workflow, QA moves this to QA-ready after the code lands on `develop`.
- **A small mystery worth noting for the reviewer:** the Jira comment from 2026-05-13 says only the 6 QA SMS numbers were added to Azure stage portal. But the live state today shows all 13 numbers (including the 4 fax numbers). Either someone (likely the ticket assignee in a separate session) added the fax numbers later without commenting, or another contributor did. Doesn't change the correctness of this PR — the reconciliation covers both paths — but flagging in case anyone wants to audit the Azure activity log.
- **Why no separate fax allowlist:** the 4 fax numbers are listed in stage.tfvars for documentation/consistency only. Today's outbound fax sender (`services/jobs/definitions/sendWeeklyProviderReport.ts`, via `sendSpruceFax`) does not consult this env var; it sends to whatever fax numbers are stored on `ProviderOffice.faxNumbers` in the staging DB. If a real fax allowlist gate is ever needed, that's a separate ticket (new env var, new code-side gate). Out of scope here.
