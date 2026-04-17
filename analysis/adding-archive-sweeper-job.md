# MLID-2167 Hotfix — Spruce Archive Sweeper

Context for the follow-up hotfix on top of PR #1126. PR #1126 closes ~99% of the archive race by saving `spruce.requestId` immediately after each Spruce send. This document covers the belt-and-suspenders changes we're adding on top to close the remaining gap, and — because I had to ask — a concrete map of how a new Pulse job reaches staging and production without an infra touch.

---

## 1. The problem we're still trying to solve

PR #1126 fixes the two-phase design in `sendMessageJob.ts` by persisting `spruce.requestId` inside the `Promise.allSettled` callback instead of deferring it to the post-batch save loop. That removes the multi-second window where the Spruce outbound webhook could land before the DB had the `requestId`.

What it does **not** fix:

- **The microtask gap.** Between `sendSpruceSms` resolving (Spruce has the `requestId`) and the new `await message.save()` completing, Node can still yield. Under load or GC pressure the webhook could still beat the save.
- **Webhook delivery failures.** If Spruce's webhook never arrives, or arrives when Mongo is briefly unavailable, the conversation stays unarchived forever. Nothing in the current system retries.
- **Process crashes.** If the worker dies between the Spruce 200 and the save, the message is sent but has no `requestId` in Mongo — the archive flow can never match it.

All three paths produce the same symptom Olha reported: "Sent" status in LISA with no ↗ link, and an unarchived conversation in Spruce.

Priority for this hotfix is **guarantee every sent message gets archived**, not "make it real-time." A 5-minute worst case is acceptable; a permanent gap is not.

---

## 2. The two fixes

### Fix A — Safety-net sweeper job (the actual guarantee)

A recurring Pulse job that periodically finds sent messages whose Spruce conversation hasn't been archived yet and archives them. This is the only change that makes "no gap" provable, because it doesn't care *why* the webhook missed — it just reconciles reality against Mongo.

**What it does, roughly:**

- Scan `Message` where:
  - `source` is in the archivable set (REMINDER, SYMPTOM, WHAT_TO_EXPECT, SKYRIZI_OBI_TRAINING, SKYRIZI_COMPLETE_PROMOTION)
  - `spruce.conversationId` exists (the webhook at least linked the conversation)
  - `spruce.archivedAt` does not exist
  - `sentDate` is inside a bounded window (e.g., between 2 min and 1 h ago — give the webhook a chance, don't rescan ancient history)
- For each hit, run the same `noteAndArchiveConversation` flow the webhook runs.
- Use a `findOneAndUpdate` with `archivedAt: { $exists: false }` in the filter to atomically claim the row before calling Spruce — prevents two sweeper runs (or the webhook and the sweeper) from double-processing the same message.

**Bounded lookback is important** — on first deploy the sweeper must not try to archive every historical prod message. Limiting `sentDate` to the last hour means only new traffic is in scope.

**Cadence:** every 5 min. Low priority. Cheap query thanks to the existing index on `spruce.requestId` (`Message.ts:97`) and the status/source compound indexes.

### Fix B — Webhook sets `archivedAt` on success

The sweeper needs a way to know "this one was already handled by the webhook." Schema addition: `spruce.archivedAt: Date`.

In `spruce_conversation.ts`, right after `archiveSpruceConversation(conversationId)` succeeds, atomically flip the flag:

```ts
await Message.updateOne(
  { 'spruce.requestId': requestId, 'spruce.archivedAt': { $exists: false } },
  { $set: { 'spruce.archivedAt': new Date() } }
);
```

Two things fall out of this:

- The sweeper filter excludes anything the webhook already archived.
- Webhook re-delivery (Spruce retries) won't post "Processed by LISA" twice — the second webhook's lookup finds `archivedAt` already set.

### What's explicitly deferred

- Moving save into `sendSpruceSms` / atomic `findOneAndUpdate` in the send path — architectural, touches a working hot path.
- Returning 5xx from the webhook on miss to force Spruce retry — the sweeper covers the same failure mode with less coupling.
- Outbox pattern for true send-then-persist atomicity — big refactor.
- Cleanup of `throw item as Error` and the redundant Phase-2 save in `sendMessageJob.ts` — backlog hygiene.

The hotfix contract is: **every sent message gets archived, eventually, period.** Everything else waits.

---

## 3. How a new Pulse job actually ships to prod

I had to chase this down because I didn't know the infra story. The short version: **nothing in Terraform changes, nothing in Azure changes, the pipeline handles everything.** The worker container app already exists, already runs `npx tsx worker.ts`, and already calls `setupRecurringJobs()` on boot. Adding a job is pure application code.

### Where each piece lives

**Infra (untouched by this hotfix):**

- `terraform/main.tf:570-1002` — `azurerm_container_app.worker` resource. Runs full-time, pulls the same image as the web container (`var.container_image`), with `WORKER_PROCESS=true` (line 622) and command `npx tsx worker.ts` (line 595).
- `terraform/main.tf:998-1000` — `ignore_changes = [template.0.container.0.image]` on the worker. This is why `terraform apply` is not part of deploys — image updates come from the pipeline, not Terraform.
- Worker scale, CPU/memory, env vars, Key Vault secrets all live in `terraform/variables.tf` + `main.tf`. Adding a sweeper touches none of them.

**Application wiring (this is where you work):**

- `apps/web/services/jobs/scheduler.ts` — one file with `setupRecurringJobs()` (line 331). Every recurring cron lives here. Called on worker boot. Safety-net pattern precedent: `abbvieAiSubstatusProcessor` at lines 396-409.
- `apps/web/services/jobs/definitions/` — one file per job. Each exports a `defineXxxJob` function that calls `defineJob(name, handler, options)` to register the handler with the Pulse instance.
- `apps/web/services/jobs/definitions/index.ts` — imports and invokes every `defineXxxJob` on boot. Missing this step means the handler exists but Pulse doesn't know how to run it.
- `apps/web/worker.ts` — the process entry point the worker container runs.

### The three-file change

1. **Create the handler** — `apps/web/services/jobs/definitions/spruceArchiveSweeper.ts`:
   ```ts
   export const SPRUCE_ARCHIVE_SWEEPER_JOB_NAME = 'spruceArchiveSweeper';

   export const spruceArchiveSweeperJob: JobHandler = async (_job, done) => {
     try {
       // Find candidates, findOneAndUpdate-claim each, archive.
       done();
     } catch (err) {
       done(err as Error);
     }
   };

   export const defineSpruceArchiveSweeperJob = async () => {
     await defineJob(SPRUCE_ARCHIVE_SWEEPER_JOB_NAME, spruceArchiveSweeperJob, {
       attempts: 1,
       lockLifetime: 5 * 60 * 1000,
       priority: 'low',
     });
   };
   ```

2. **Register it** — add the import + call to `apps/web/services/jobs/definitions/index.ts` alongside the other `defineXxxJob` entries.

3. **Schedule it** — one block in `scheduler.ts` (near the existing safety-net crons):
   ```ts
   await scheduleRecurringJob(
     SPRUCE_ARCHIVE_SWEEPER_JOB_NAME,
     '*/5 * * * *',
     {},
     { timezone: 'America/New_York', skipImmediate: true }
   );
   logger.info('Spruce archive sweeper scheduled every 5 min');
   ```

### Deploy path

Standard pipeline:

- Merge PR → `develop` → `buildserver/cloudbuild.yaml.staging` runs → new image published to ACR → staging worker container restarts → on boot, `setupRecurringJobs()` upserts the cron entry into MongoDB `pulseJobs` → Pulse picks it up on the next tick → sweeper starts running in staging.
- Merge to `main` → production follows the same flow via `cloudbuild.yaml.production`.

No `terraform apply`. No `az containerapp update`. No new Key Vault secret. No new env var. No SRE ticket.

### What we explicitly can't do from code

For completeness, these need infra help (not required for this hotfix):

- Worker replicas / CPU / memory — `terraform/variables.tf`
- New env vars on the worker — `main.tf` worker block
- New Key Vault secret — `main.tf` + Azure
- Scaling rules — Container App scale config

---

## 4. Rollout considerations

- Both changes are additive. The webhook still does everything it did before; it just writes an extra `archivedAt` field on success. The sweeper is a new file; zero impact on the existing flow if it misbehaves.
- The sweeper's first production run will see messages sent in the last hour that have `conversationId` but no `archivedAt`. If the webhook was working for those, they'll already be archived in Spruce — but our DB doesn't know. The `noteAndArchiveConversation` call is idempotent for the archive PATCH, but **it would double-post "Processed by LISA" notes** on every pre-fix message during the first hour after deploy. Two options:
  - **Backfill:** before turning on the sweeper, run a one-off script that sets `archivedAt` on all recent SENT messages with a `conversationId`. Accepts some false positives (messages that weren't actually archived), but those will get caught the next time the message flow runs normally — and duplicate "Processed by LISA" notes are worse than a handful of messages staying unarchived for one cycle.
  - **Feature-flag the sweeper** off on first deploy, confirm the webhook is setting `archivedAt` on new traffic, then enable the sweeper after the 1 h lookback window has fully turned over. Safer.
- Consider feature-flagging both changes via the existing `FeatureFlagService` so we can disable without a redeploy if the sweeper goes sideways in prod.
