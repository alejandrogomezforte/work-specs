# Spruce SMS Messages — Volume Analytics (Production)

Purpose: measure the real production volume of outbound SMS messages so we can
evaluate whether the `spruceArchiveSweeper` cadence (every 5 min) is
appropriately sized.

All data was pulled from the **Production** environment only.

---

## Environment & data sources

- **MongoDB (source of truth for message counts):** Atlas cluster
  `local-infusion-prod` → database `local-infusion-db` → collection `messages`.
  Accessed read-only via Atlas Data Explorer (Documents + Aggregations tabs).
- **Azure Application Insights (job execution logs):** resource `li-prod-appins`
  in `li-prod-monitoring-rg`, subscription `Production`
  (`04cb1c68-e911-423a-a794-5df89d268ffe`), AppId
  `b1af6c79-2b5c-4eb2-a342-c3c55c80eedb`. Queried via `az monitor app-insights
  query` (KQL, read-only).

Raw log dumps referenced below are saved under `docs/agomez/logs/`.

### Key data shapes

- `messages.status`: lowercase enum (e.g. `"sent"`, `"declined"`, `"pending"`).
- `messages.source`: lowercase enum. Archivable sources (in scope for the
  sweeper) are defined in
  `apps/web/services/webhooks/handlers/spruce_conversation.ts:115-122`:
  - `reminder`
  - `symptom`
  - `what-to-expect`
  - `skyrizi-obi-training`
  - `skyrizi-complete-promotion`
- `messages.sentDate`: UTC timestamp set when the Spruce API call succeeds
  (`sendMessageJob.ts:335,349,370`).
- `messages.spruce.conversationId`: present only when the post-send Spruce
  webhook delivered — this is the sweeper's input surface.
- `messages.spruce.archivedAt`: present once the conversation has been archived
  (either by the webhook or by the sweeper).

---

## Last 24 hours (2026-04-16 21:00 UTC → 2026-04-17 ~21:00 UTC)

### MongoDB — messages with `status: "sent"`

Atlas aggregation used:

```js
[
  { $match: {
      status: "sent",
      sentDate: { $gte: ISODate("2026-04-16T21:00:00.000Z") }
  } },
  { $group: { _id: "$source", count: { $sum: 1 } } },
  { $sort: { count: -1 } }
]
```

| Source | Count | Archivable? |
|---|---:|:---:|
| `symptom` | 206 | ✅ |
| `what-to-expect` | 60 | ✅ |
| `skyrizi-complete-promotion` | 4 | ✅ |
| `skyrizi-obi-training` | 2 | ✅ |
| `reminder` | 1 | ✅ |
| **Total** | **273** | **100% in-scope** |

- Daily volume: **273 SENT messages / 24h** ≈ **11.4 / hour average**.
- Every sent message in this window was for an archivable source — the sweeper
  has no "non-archivable noise" to filter out.
- Skew: `symptom` alone is **75%** of all sent traffic.

### Application Insights — `sendMessageJob` handler activity

Raw dump: `docs/agomez/logs/sendMessageJob-prod-last24h_2026-04-17.json`.

**Caveat — the data window for this dump is partial.** Although the query asked
for the last 24h, `li-prod-appins` only has `sendMessageJob` traces starting at
**~19:30 UTC on 2026-04-17** (~1h of data), not the full 24h. The earlier part
of the day either wasn't logged or went to a different destination; this needs
follow-up before we can correlate logs and Mongo 1:1 across the full 24h.

Breakdown of the 75 log rows that were available:

| Count | Log line |
|---:|---|
| 30 | `sendMessageJob: start sending messages (start: …, end: …)` |
| 24 | `sendMessageJob: nothing to send. Done.` |
|  6 | `sendMessageJob: N messages will be send` |
|  6 | `sendMessageJob: job completed successfully` |
|  6 | `sendMessageJob: message source '…' sent (<id>)` |
|  3 | `sendMessageJob: message source '…' failed (<id>)` |

- Scheduler cadence observed: **every 2 minutes** (30 run-starts in ~60 min).
- In this 1h slice: 9 send attempts (6 sent, 3 failed), all `what-to-expect`.
- The Pulse job name on the scheduler is `sendMessageProcessing`
  (`scheduler.ts:729-740`), but the handler always logs with the
  `sendMessageJob:` prefix (`sendMessageJob.ts:471-485`). They are the same
  thing.

---

## Last 7 days (2026-04-10 21:00 UTC → 2026-04-17 ~21:00 UTC)

Atlas aggregation used (same as 24h, only the cutoff changed):

```js
[
  { $match: {
      status: "sent",
      sentDate: { $gte: ISODate("2026-04-10T21:00:00.000Z") }
  } },
  { $group: { _id: "$source", count: { $sum: 1 } } },
  { $sort: { count: -1 } }
]
```

| Source | 7d count | 7d share | Archivable? |
|---|---:|---:|:---:|
| `reminder` | 1,097 | 46.3% | ✅ |
| `symptom` | 1,040 | 43.9% | ✅ |
| `what-to-expect` | 208 | 8.8% | ✅ |
| `skyrizi-obi-training` | 13 | 0.5% | ✅ |
| `skyrizi-complete-promotion` | 11 | 0.5% | ✅ |
| **Total** | **2,369** | **100%** | **100% in-scope** |

- Weekly volume: **2,369 SENT messages / 7d** ≈ **338 / day average** ≈ **14 / hour average**.
- All 5 sources are archivable — sweeper scope stays at 100% of SENT traffic.
- **Source mix differs sharply from the 24h sample.** The 24h window (Fri
  2026-04-17) had only 1 `reminder` (0.4%) vs. the 7-day avg of ~157/day
  (46% of daily volume). Likely cause: the 24h window we sampled does include
  ~9 AM EDT (13:00 UTC) of 2026-04-17, but that day's reminder cohort was
  unusually small — worth confirming with a per-day-of-week breakdown when
  we pull the 30-day window.
- `symptom` is steadier in ratio: 75% in 24h vs. 44% in 7d. That's still a
  real shift — could be weekend appointment volume, could be a reminder-job
  hiccup. Flag for the 30-day pass.

### Comparison: 24h vs 7-day-average-day

| Source | 24h count | 7d avg/day | Δ |
|---|---:|---:|---:|
| `reminder` | 1 | 156.7 | −99.4% |
| `symptom` | 206 | 148.6 | +38.6% |
| `what-to-expect` | 60 | 29.7 | +102% |
| `skyrizi-obi-training` | 2 | 1.9 | +7.5% |
| `skyrizi-complete-promotion` | 4 | 1.6 | +154% |
| **Total** | **273** | **338.4** | **−19%** |

---

## Last 30 days (2026-03-18 21:00 UTC → 2026-04-17 ~21:00 UTC)

Atlas aggregation used (Aggregations tab — filter input in the Documents tab
is a cramped single-line field, the pipeline editor is much easier to work
with for iterative queries):

```js
[
  { $match: {
      status: "sent",
      sentDate: { $gte: ISODate("2026-03-18T21:00:00.000Z") }
  } },
  { $group: { _id: "$source", count: { $sum: 1 } } },
  { $sort: { count: -1 } }
]
```

| Source | 30d count | 30d share | Archivable? |
|---|---:|---:|:---:|
| `reminder` | 4,444 | 57.4% | ✅ |
| `symptom` | 2,402 | 31.0% | ✅ |
| `what-to-expect` | 867 | 11.2% | ✅ |
| `skyrizi-obi-training` | 16 | 0.2% | ✅ |
| `skyrizi-complete-promotion` | 13 | 0.2% | ✅ |
| **Total** | **7,742** | **100%** | **100% in-scope** |

- Monthly volume: **7,742 SENT messages / 30d** ≈ **258 / day average** ≈
  **10.75 / hour average**.
- All 5 sources remain archivable — sweeper scope is still 100% of SENT
  traffic.
- `reminder` is the dominant source at the monthly scale (57%), confirming
  it's the workhorse flow and the 24h sample's single-reminder count was a
  real outlier, not a misread.

### Comparison across windows

| Source | 30d avg/day | 7d avg/day | 24h | 7d vs 30d | 24h vs 30d |
|---|---:|---:|---:|---:|---:|
| `reminder` | 148.1 | 156.7 | 1 | +5.8% | −99.3% |
| `symptom` | 80.1 | 148.6 | 206 | +85.5% | +157% |
| `what-to-expect` | 28.9 | 29.7 | 60 | +2.7% | +108% |
| `skyrizi-obi-training` | 0.5 | 1.9 | 2 | +247% | +274% |
| `skyrizi-complete-promotion` | 0.4 | 1.6 | 4 | +263% | +823% |
| **Total** | **258.1** | **338.4** | **273** | **+31.1%** | **+5.8%** |

Observations:

- Last 7 days ran **31% above** the 30-day baseline — last week was a heavy
  week, not a light one. Driven mainly by `symptom` (+85% vs. baseline) and
  the skyrizi sources (small absolute numbers but proportionally large).
- Today's 24h sits close to the 30-day baseline (+5.8% total), but the
  source mix is skewed: `symptom` spiked (+157%), `what-to-expect` nearly
  doubled (+108%), while `reminder` collapsed to 1. That's a single-day
  anomaly worth explaining before we use today's data for anything.
- Hourly peak pressure on the sweeper is still bounded by the hourly average
  (~11/hr at 30d baseline, ~14/hr at 7d baseline). Even at the heavy 7d
  baseline, the sweeper's per-tick upper bound stays under ~15 candidates.

---

## Volume implications for `spruceArchiveSweeper`

The sweeper's input surface (messages from the last hour with
`spruce.conversationId` set but no `spruce.archivedAt`) is bounded by:

1. the **per-hour** SENT volume (not per-day), because the sweeper's lookback
   window is 1h (`spruceArchiveSweeper.ts:32`, `LOOKBACK_MS = 60 * 60 * 1000`);
2. the **webhook miss rate** — the webhook archives within milliseconds on the
   happy path, so the sweeper only processes the residual.

From the 7-day data (more representative than the 24h sample), the upper
bound for sweeper candidates per tick is **~14 messages** (one full hour of
SENT traffic at the 7-day hourly average), and only if the webhook missed
100% of them. Realistic candidate count per tick is expected to be **0–1**
given a healthy webhook.

The next step (still TODO) is to quantify webhook miss rate directly:
count how many messages in the last 24h have `spruce.conversationId` set
but no `spruce.archivedAt`. That is the true sweeper workload.

---

## TODO — follow-up windows

- [x] Last 7 days — total + source/status breakdown.
- [x] Last 30 days — monthly totals and source mix.
- [ ] Explain the 24h `reminder` anomaly (1 sent in 24h vs. 30d avg
  ~148/day) — likely candidates: appointment-reminder job failure/skip,
  empty `dateToSend` cohort for today, or a feature-flag flip. Correlate
  with App Insights traces for `appointmentReminder` and `sendMessageJob:
  today is a holiday…` lines once the log gap is resolved.
- [ ] Per-day-of-week distribution across the 30-day window to identify
  real peak vs. off days (vs. one-off anomalies).
- [ ] Webhook miss rate: for each window, count messages where
  `spruce.conversationId` exists but `spruce.archivedAt` does not (split by
  still-within-lookback vs. past-lookback). This is the sweeper's actual
  workload.
- [ ] Resolve the App Insights log gap (worker logs only visible from ~19:30 UTC
  on 2026-04-17) so execution logs and Mongo counts can be reconciled.
