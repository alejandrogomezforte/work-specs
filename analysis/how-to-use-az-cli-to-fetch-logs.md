# How to fetch Pulse job logs from Azure via `az` CLI

This guide documents the end-to-end process I used to pull logs for the
`sendMessageProcessing` Pulse job from the staging environment. The same
steps work for any other Pulse job — just swap the job name / time window.

---

## Why this works

- The web app and the worker run as **Azure Container Apps** in
  `li-stage-app-rg` (staging) / `li-prod-app-rg` (production).
- Both containers send `logger.info/warn/error` output to
  **Application Insights**, which stores it in Log Analytics under the
  `traces` table.
- Pulse job handlers prefix every log line with the job name
  (e.g. `sendMessageJob: start sending messages ...`), so we can filter
  the `traces` table by message content.

> Heads-up: the handler for `sendMessageProcessing` is the same function
> as `sendMessageJob` (see `apps/web/services/jobs/definitions/sendMessageJob/sendMessageJob.ts:445-465`),
> so its log lines are prefixed with `sendMessageJob:`. For other jobs,
> check the handler file to see the real prefix being logged.

---

## Prerequisites

1. Azure CLI installed (`az --version`).
2. Logged in: `az login`.
3. The `application-insights` extension:
   ```bash
   az extension add --name application-insights --yes
   ```
4. Access to the right subscription:
   - **Staging** lives in the `Non-Production` subscription (default on my machine).
   - **Production** lives in a separate subscription — switch with
     `az account set --subscription <prod-sub-id>`.

---

## Step 1 — Confirm the subscription you are in

```bash
az account show
```

Look at `name` in the output. It should be `Non-Production` for staging
and the prod subscription for production. If it is wrong:

```bash
az account list --output table
az account set --subscription "<name-or-id>"
```

---

## Step 2 — Find the Container Apps (optional, just to orient yourself)

This is useful if you want to know which app a job runs in. Pulse jobs
run inside the **worker** container app.

```bash
az containerapp list --output table
```

Example output (staging):

```
Name                            ResourceGroup
------------------------------  ---------------
li-stage-local-infusion-app     li-stage-app-rg   # Next.js web app
li-stage-local-infusion-worker  li-stage-app-rg   # Pulse worker (jobs run here)
li-stage-li-public-api          li-stage-app-rg
li-stage-li-mcp-server          li-stage-app-rg
...
```

Production mirrors this with `li-prod-*` names in `li-prod-app-rg`.

---

## Step 3 — Find the Application Insights resource

Logs from the worker go to the **main** Application Insights instance,
which is in the `monitoring` resource group — not the app RG.

```bash
az monitor app-insights component show --output table
```

Staging result:

```
Name                      ResourceGroup           AppId
------------------------   ----------------------  ------------------------------------
li-stage-appins           li-stage-monitoring-rg  b0855cae-c060-4be6-baff-4e36f9fa7f52
li-stage-vig-appinsights  li-stage-app-rg         400f2b20-a0fc-4173-a071-93fa87383bb3  # VIG only, not us
```

The value we care about is `AppId` of `li-stage-appins`:
`b0855cae-c060-4be6-baff-4e36f9fa7f52`. That is what
`az monitor app-insights query` uses with the `--app` flag.

For production the resource is named `li-prod-appins` in
`li-prod-monitoring-rg`. Run the same command after switching
subscriptions to get its `AppId`.

---

## Step 4 — Query the `traces` table with KQL

The query is written in **KQL** (Kusto Query Language). The structure is:

```
traces
| where timestamp > ago(<window>)
| where message contains '<job log prefix>'
| project timestamp, severityLevel, message
| order by timestamp desc
| take <N>
```

- `traces` — the Log Analytics table that holds every `logger.*` line.
- `timestamp > ago(24h)` — time window. `24h`, `2h`, `30m`, `7d` all work.
- `message contains '...'` — filter by substring. Use the log prefix
  from the job handler. For our example it's `sendMessageJob`.
- `severityLevel` — `1 = Info`, `2 = Warning`, `3 = Error`.

Full command for our example — **last 24 hours of `sendMessageProcessing`
(= `sendMessageJob` prefix) on staging**:

```bash
az monitor app-insights query \
  --app b0855cae-c060-4be6-baff-4e36f9fa7f52 \
  --analytics-query "traces
    | where timestamp > ago(24h)
    | where message contains 'sendMessageJob'
    | project timestamp, severityLevel, message
    | order by timestamp desc
    | take 50"
```

Sample rows we got back:

```
2026-04-17T15:32:00Z  Error  sendMessageJob: message source 'what-to-expect' failed (69e25237d9b6e7f5785b49bb)
2026-04-17T15:32:00Z  Info   sendMessageJob: job completed successfully
2026-04-17T15:32:00Z  Info   sendMessageJob: 1 messages will be send
2026-04-17T15:32:00Z  Info   sendMessageJob: start sending messages (start: ..., end: ...)
2026-04-17T15:30:00Z  Info   sendMessageJob: nothing to send. Done.
...
```

---

## Useful query variations

### Only errors for the job

```bash
az monitor app-insights query \
  --app b0855cae-c060-4be6-baff-4e36f9fa7f52 \
  --analytics-query "traces
    | where timestamp > ago(24h)
    | where message contains 'sendMessageJob'
    | where severityLevel >= 3
    | project timestamp, message
    | order by timestamp desc"
```

### All log lines around a specific entity id

Useful when the job log points at a specific `messageId`, `patientId`,
or `appointmentId`. Replace the id with the one you saw in the trace.

```bash
az monitor app-insights query \
  --app b0855cae-c060-4be6-baff-4e36f9fa7f52 \
  --analytics-query "traces
    | where timestamp > ago(24h)
    | where message contains '69e25237d9b6e7f5785b49bb'
    | project timestamp, severityLevel, message
    | order by timestamp asc"
```

### Exceptions table (full stack trace)

`logger.error(..., err)` sends the exception to the `exceptions` table
as well, which contains the stack. Cross-reference with a timestamp
from `traces`:

```bash
az monitor app-insights query \
  --app b0855cae-c060-4be6-baff-4e36f9fa7f52 \
  --analytics-query "exceptions
    | where timestamp > ago(24h)
    | where outerMessage contains 'sendMessage' or innermostMessage contains 'sendMessage'
    | project timestamp, type, outerMessage, innermostMessage, details
    | order by timestamp desc
    | take 20"
```

### Output formats

- Default is JSON — good for piping into `jq`.
- Add `--output table` for a quick terminal view (but long messages get truncated).
- Add `--output tsv` if you want to grep or pipe.

---

## Recap: the exact commands I ran, in order

```bash
# 1. Verify CLI + subscription
az --version
az account show

# 2. Install the App Insights extension (one-time)
az extension add --name application-insights --yes

# 3. Orient myself (optional)
az containerapp list --output table

# 4. Find the App Insights AppId
az monitor app-insights component show --output table

# 5. Query traces for the job in the last 24h
az monitor app-insights query \
  --app b0855cae-c060-4be6-baff-4e36f9fa7f52 \
  --analytics-query "traces
    | where timestamp > ago(24h)
    | where message contains 'sendMessageJob'
    | project timestamp, severityLevel, message
    | order by timestamp desc
    | take 50"
```

---

## Doing the same thing on production

1. `az account set --subscription "<Prod subscription name or id>"`
2. `az monitor app-insights component show --output table`
   → grab the `AppId` of `li-prod-appins`.
3. Re-run the query in Step 5 with that `AppId`.

Everything else (KQL, log prefix, filters) is identical.

---

## Troubleshooting

- **`The command requires the extension application-insights`** — run
  `az extension add --name application-insights --yes` and retry.
- **Empty result set** — widen the time window (`ago(7d)`) or confirm
  the job actually ran recently. You can cross-check the Pulse
  collection in MongoDB (`db.pulse.find({ name: "sendMessageProcessing" })`).
- **Wrong subscription** — `az account show` and
  `az account set --subscription ...`.
- **Truncated messages in table output** — drop `--output table` and
  read the raw JSON, or pipe to `jq '.tables[0].rows[] | .[2]'`.
