# MLID-2167 — Relax `spruceArchiveSweeper` cadence from 5 min to 15 min

**Branch:** `fix/MLID-2167-spruce-sweeper-cadence-15min`
**Base:** `develop`
**Create PR:** https://github.com/LocalInfusion/li-call-processor/pull/new/fix/MLID-2167-spruce-sweeper-cadence-15min

---

## Summary

The `spruceArchiveSweeper` Pulse job (introduced in MLID-2167 as the safety-net for Spruce webhook misses) runs every 5 minutes. Production volume analysis (30 days) shows the cadence is heavily over-spec'd: there's ~4× headroom inside the sweeper's 60-minute lookback window. This PR relaxes the cron to every 15 minutes.

No behavioral change outside the tick interval — `LOOKBACK_MS`, `RECENT_SKIP_MS`, the candidate filter, and the atomic claim-then-archive flow are unchanged.

## Why

The sweeper is a safety net, not a user-facing latency path. The post-send Spruce webhook remains the primary archival path and completes within milliseconds on the happy path. The sweeper only acts on the residual cases (process crashes, transient Mongo errors, webhook delivery failure).

Production volume analysis (see `docs/agomez/analysis/spruce-messages-volume-analytics.md`):

| Window | Total SMS (archivable) | Per day | Per hour |
|---|---:|---:|---:|
| 30 days | 7,742 | 258 | 10.75 |
| 7 days (heavy week) | 2,369 | 338 | 14 |

Cadence trade-off:

| Cadence | Worst-case archival delay | Lookback headroom | Ticks/hr |
|---|---|---|---:|
| **5 min (before)** | 7 min | 12× | 12 |
| **15 min (this PR)** | 17 min | 4× | 4 |
| 30 min | 32 min | 2× | 2 |

Why 15 min is the sweet spot:

- **Still 4× headroom** inside the 60-min lookback (45 min safety margin) — no miss will ever age out of the window.
- **Worst-case archival delay of 17 min** is acceptable for a team-internal "Processed by LISA" note that only appears on webhook-miss residuals (not the happy-path 99%+).
- **Cuts Pulse scheduler overhead on this job by ~66%** (12 ticks/hr → 4) — all real savings come from eliminating empty scans that find nothing to do.
- **Per-tick workload stays trivial** — candidate ceiling at 7d peak rate is <15 messages, `BATCH_SIZE` is 100, expected candidates per tick are 0–1 given a healthy webhook.

5 minutes was a reasonable conservative default when the job shipped. With a month of production data in hand we can size it more intentionally.

## Changes

**`apps/web/services/jobs/scheduler.ts`** — two lines:

- Cron: `'*/5 * * * *'` → `'*/15 * * * *'`
- Scheduler log: `'Spruce archive sweeper scheduled every 5 minutes'` → `'... every 15 minutes'`

No code touched inside `spruceArchiveSweeper.ts` itself. `LOOKBACK_MS` (60 min) and `RECENT_SKIP_MS` (2 min) are unchanged — the query window is still "last hour, skip the last 2 min".

## Tests

- `apps/web/services/jobs/scheduler.test.ts` — 15 tests pass unchanged (no existing assertion on the cron string, only on the mocked job name).
- Manual verification of the diff — 2 lines, string-only.

TDD note per CLAUDE.md: a cron-string config change with zero conditional logic is a legitimate TDD exception (see CLAUDE.md §"Legitimate TDD Exceptions" → "Configuration Changes"). No new test added.

## Rollout / monitoring

After deploy, a one-time worker restart will re-register the recurring job with the new cron. The existing `removeDuplicateRecurringJobs` invocations in `scheduler.ts` handle any stale 5-minute entry automatically.

Things to watch in the first hour:

- App Insights should continue to show zero `spruceArchiveSweeper:` handler log lines on empty ticks (silent no-op when there are no candidates — `spruceArchiveSweeper.ts:84-86`).
- On a tick with real work, expect `spruceArchiveSweeper: candidates found` followed by `spruceArchiveSweeper: run complete {scanned, claimed, archived, released}`.
- If `candidates found` suddenly fires on every tick with high counts, the webhook is degraded — revert this cadence change and investigate the webhook first.

## Rollback

Single-commit revert. Nothing stateful changes.
