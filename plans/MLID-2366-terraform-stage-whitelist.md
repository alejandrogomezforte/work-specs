# MLID-2366 — Whitelist numbers for sending messages from staging env

## Task Reference

- **Jira**: [MLID-2366](https://localinfusion.atlassian.net/browse/MLID-2366)
- **Story Points**: 1
- **Branch**: `feature/MLID-2366-terraform-stage-whitelist`
- **Base Branch**: `develop`
- **Status**: In Progress
- **Type**: Standalone task (no parent epic)

---

## Summary

QA (Olha) needs 6 SMS phone numbers and 4 fax numbers whitelisted in the staging environment so she can run SMS flows safely without accidentally sending messages to real patients. Stage 1 (the manual Azure portal update to `APPOINTMENT_REMINDER_TARGET_CRITERIA`) is already done. This task is Stage 2: sync terraform with that manual change so the next infra deploy does not overwrite it. All 13 numbers (3 baseline + 6 SMS + 4 fax) are written into `stage.tfvars` for long-term consistency and traceability, even though the 4 fax numbers are no-ops in application code today.

---

## Scope

**In scope:**
- Add `spruce_appointment_reminder_target_criteria` to `terraform/environments/stage.tfvars` with the full 13-number string
- Update `terraform/variables.tf` to make the variable's default an empty string (`""`) instead of the legacy 3-number baseline, and expand the description to document the per-environment contract. **Rationale:** the legacy default contained staging-leaning test numbers sitting in a "shared default" position, which blurred the prod/stage boundary. Each environment now opts in explicitly — prod via `"*"` in `prod.tfvars`, stage via the 13-number CSV in `stage.tfvars`. Behavioral impact is zero because both real environments have explicit overrides.

**Explicitly out of scope:**
- No application code changes
- No changes to `terraform/main.tf` or `terraform/environments/prod.tfvars`
- No changes to `spruce_appointment_reminder_allow_fnf` (defaults to `"false"`, correct for staging)
- No Jest tests, no TypeScript type checks, no linting
- No Jira ticket transitions
- No fax allowlist gate in application code (future follow-up if needed)

---

## Codebase Analysis

- The variable `spruce_appointment_reminder_target_criteria` is declared in `terraform/variables.tf:200` and was previously defaulted to `"7078817918,9176755175,2014030962"` (3-number staging-leaning baseline). That default is being changed to `""` so the variable is per-environment-only.
- `terraform/environments/stage.tfvars` previously had no override for this variable, so the legacy 3-number default was what terraform applied. Adding the explicit override here is the correct pattern for environment-specific values.
- `terraform/environments/prod.tfvars` sets this variable to `"*"` (allow all). That file must not be touched.
- The insertion point in `stage.tfvars` is after line 43 (`intake_api_url = ...`) and before line 45 (`feature_flags = {`), grouping it with other external-service configuration values. This mirrors the style of MLID-2011 commits: blank line + comment + assignment.
- Terraform CI validation is commented out in `azure-deploy.yml`. There is no automated terraform gate on PRs. Local verification is the only gate for this change.

**Number breakdown (for traceability):**

| Numbers | Count | Source |
|---------|-------|--------|
| `7078817918,9176755175,2014030962` | 3 | Original baseline (variables.tf default) |
| `2252542523,8022843792,9707840653,3802603245,9707840507,2018577757` | 6 | QA SMS numbers — already live in Azure stage portal |
| `7047041494,9199848698,9049134304,6463987792` | 4 | QA fax numbers — documentation-only, no-ops in code today |

---

## Implementation Steps

### Step 1 — Create feature branch from develop

Pull the latest `develop` locally, then branch off it. Do not rebase.

```bash
git checkout develop
git pull origin develop
git checkout -b feature/MLID-2366-terraform-stage-whitelist
```

No files changed. This step produces no commit.

---

### Step 2 — Edit stage.tfvars

**File:** `terraform/environments/stage.tfvars`

Insert the following block after line 43 (`intake_api_url = ...`). The existing blank line at line 44 stays; add the new content after it so it sits between `intake_api_url` and `feature_flags`.

**Diff context:**

```
  Line 43:  intake_api_url = "https://li-stage-intake.agreeableflower-6a3cfa05.eastus2.azurecontainerapps.io"
  Line 44:  (blank)
+ Line 45:  (blank)
+ Line 46:  # Spruce — whitelist QA phone/fax numbers for staging SMS testing (see MLID-2366)
+ Line 47:  spruce_appointment_reminder_target_criteria = "7078817918,9176755175,2014030962,2252542523,8022843792,9707840653,3802603245,9707840507,2018577757,7047041494,9199848698,9049134304,6463987792"
  Line 48:  (blank — was line 44 before: separates from feature_flags block)
  Line 49:  feature_flags = {
```

**Exact text to insert** (place between the existing blank line after `intake_api_url` and the `feature_flags` block):

```hcl

# Spruce — whitelist QA phone/fax numbers for staging SMS testing (see MLID-2366)
spruce_appointment_reminder_target_criteria = "7078817918,9176755175,2014030962,2252542523,8022843792,9707840653,3802603245,9707840507,2018577757,7047041494,9199848698,9049134304,6463987792"
```

---

### Step 3 — Local verification (USER runs these manually; AI guides)

**Execution model:** The AI must NOT run any `terraform` commands automatically. Terraform writes to remote backend state and consumes infra resources; the user executes them in their own shell and pastes output back for AI to interpret.

The AI's role in this step:
1. Present each command one at a time with a one-sentence explanation of what it does and what to look for in the output.
2. Wait for the user to run it and paste back the output (or confirm success).
3. Interpret the output, flag anything unexpected, and only then move to the next command.
4. After `terraform plan`, the AI must read the plan output carefully and confirm the expected outcome below before proceeding to Step 4.

Commands the user will run (from the `terraform/` directory):

```bash
cd terraform

terraform init \
  -backend-config="storage_account_name=litfstatestage" \
  -backend-config="container_name=tfstate" \
  -backend-config="resource_group_name=li-tfstate-stage-rg" \
  -reconfigure

terraform validate -no-color

terraform plan -var-file=environments/stage.tfvars -out=tfplan -no-color
```

**Expected `terraform plan` output:** exactly two `env` block updates — one on the web container app and one on the worker container app — both changing `APPOINTMENT_REMINDER_TARGET_CRITERIA` from the 3-number baseline string to the new 13-number string. No other resources should show as changed.

If the plan shows any resource changes beyond those two env block updates, stop and investigate before committing.

---

### Step 4 — Commit

Stage both terraform files and commit with a detailed message.

```bash
git add terraform/variables.tf terraform/environments/stage.tfvars

git commit -m "[MLID-2366] - chore(terraform): scope Spruce SMS whitelist to per-environment, sync stage

Stage 2 of MLID-2366. Stage 1 (manual Azure portal updates to
APPOINTMENT_REMINDER_TARGET_CRITERIA on the web and worker container
apps) was already applied to keep QA unblocked. This commit syncs
terraform so the next infra deploy does not overwrite the manual
change, and tightens the variable's scope so prod and stage stop
sharing a stage-leaning default.

variables.tf:
- Default changed from the legacy 3-number staging-leaning string
  (7078817918,9176755175,2014030962) to '' so the variable is
  per-environment-only. The 3 baseline numbers were not lost — they
  live in stage.tfvars now alongside the new QA numbers.
- Description expanded to document the per-env contract.

stage.tfvars:
- New override binds the variable to the 13-number string that
  staging requires for QA testing:
  - 7078817918, 9176755175, 2014030962 — original 3-number baseline
  - 2252542523, 8022843792, 9707840653, 3802603245, 9707840507,
    2018577757 — 6 QA SMS numbers requested by Olha
  - 7047041494, 9199848698, 9049134304, 6463987792 — 4 QA fax
    numbers; no fax allowlist gate exists in application code today
    so these are documentation-only no-ops until a future ticket
    adds that gate

No application code changed. prod.tfvars untouched (remains '*').
terraform plan verified locally against stage: 0 env-block changes
(Azure already aligns with the new tfvars value, so this commit is
a declarative reconciliation rather than a behavior change)."
```

---

### Step 5 — Push branch and write PR draft

Push the feature branch:

```bash
git push -u origin feature/MLID-2366-terraform-stage-whitelist
```

Then write the PR draft to `docs/agomez/PR/MLID-2366-terraform-stage-whitelist.md` for the user to open manually from the GitHub UI. The PR draft should include:

- **Title:** `[MLID-2366] chore(terraform): add Spruce SMS/fax whitelist to stage.tfvars`
- **Base branch:** `develop`
- **Body sections:**
  - Summary (Stage 2, syncs terraform with manual Azure change)
  - What changed (one file, one variable added)
  - Number breakdown table (same as the traceability table above)
  - Verification checklist (`terraform validate` passed, `terraform plan` showed exactly 2 env-block changes)
  - Notes: CI terraform gate is disabled; no application code changed; fax numbers are no-ops today

Do not run `gh pr create`. Do not transition the Jira ticket.

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `terraform/environments/stage.tfvars` | Modify | Add `spruce_appointment_reminder_target_criteria` override with all 13 whitelisted numbers |
| `terraform/variables.tf` | Modify | Change default from the legacy 3-number staging-leaning string to `""`, expand description to document the per-environment contract |
| `docs/agomez/PR/MLID-2366-terraform-stage-whitelist.md` | Create | PR draft for user to open manually from GitHub UI |

---

## Testing Strategy

This is a terraform-config-only change. The TDD red-green-refactor cycle does not apply. Verification is:

1. **`terraform init -reconfigure`** with the staging backend-config flags — must end with "Terraform has been successfully initialized!".
2. **`terraform validate -no-color`** — skipped locally. Known terraform-version wart with partial backend configs makes this fail locally with "Missing required argument" on `container_name` / `storage_account_name`, despite init having stored those values in `.terraform/`. CI runs this in a workflow environment that does not hit the bug; plan exercises the same parser, so plan succeeding is sufficient.
3. **`terraform plan -var-file=environments/stage.tfvars -out=tfplan -no-color`** — verified locally. Expected outcome: **0 env-block changes** for `APPOINTMENT_REMINDER_TARGET_CRITERIA` on both the web and worker container apps. The reason zero changes appear is that Azure portal was already manually updated to the 13-number string during Stage 1; this commit reconciles the tfvars source-of-truth with that live state so a future `terraform apply` does not revert it. Pre-existing drift unrelated to this change (worker `startup_probe.initial_delay` and container app revision numbers) is acceptable and stays unaddressed. The plan may end with an unrelated Cloudflare provider auth error if `CLOUDFLARE_API_TOKEN` is not set locally — that's a tooling/auth gap, not a configuration problem, and does not block the implementation.
4. **No Jest, no type checks, no lint** — no application code was touched.

Note: there is no automated terraform gate on PRs. The CI terraform block is commented out in `azure-deploy.yml`. Local verification (steps above) is the only gate before merging.

---

## Security Considerations

- **PHI handling:** phone numbers in this variable are QA test numbers, not patient PHI. No patient data is introduced.
- **Prod safety:** `prod.tfvars` is not touched. Production remains `"*"` (allow all). Staging becomes restricted to the 13-number list, which is the intended direction.
- **No secrets:** phone numbers are not credentials. Committing them to terraform config is consistent with how the baseline numbers are stored in `variables.tf`.

---

## Open Questions

None. Olha confirmed fax numbers as no-ops are acceptable. Scope is fully defined.

---

## Out of Scope / Future Work

If QA ever needs the 4 fax numbers to actually gate fax sends, a follow-up ticket would need to:
1. Add a separate `spruce_fax_target_criteria` (or equivalent) variable to the application code and terraform
2. Wire the new variable into the fax-sending service path (analogous to how `APPOINTMENT_REMINDER_TARGET_CRITERIA` gates SMS)

That work is out of scope for MLID-2366.

---

## Risks and Notes

- **Disabled CI terraform gate:** `azure-deploy.yml` has the terraform validation/plan step commented out. The PR will merge without any automated terraform check. Local verification in Step 3 is the only safety net — do not skip it.
- **Fax numbers are no-ops today:** the 4 fax numbers (`7047041494`, `9199848698`, `9049134304`, `6463987792`) are included for documentation and future-proofing, but the application has no fax allowlist code path that reads this variable. Adding them to terraform does not enable any behavior; it is purely a tracking decision.
- **Stage 1 already live:** the Azure stage portal already has the 6 SMS numbers set manually. This terraform change should produce a plan that matches what is already running — the plan output confirms alignment rather than introducing a net-new change to the running environment for those 6 numbers.
