# How the `ordersTrackerCacheJob` Works

## What it is

A **long-running materialized-view cache** for the Orders Tracker queries. Instead of recomputing the expensive multi-`$lookup` orders aggregation (with eligibility) every time the UI/API asks for orders, this job pre-builds a flattened cache collection and keeps it in sync with MongoDB **change streams**.

- Job name constant: `ORDERS_TRACKER_CACHE_JOB_NAME = 'ordersTrackerCacheJob'`
- Cache collection: `orderstrackercache` (`COLLECTION.OrdersTrackerCache` in `apps/web/utils/constants/collections.ts:15`)
- Resume token: stored in `ResumeTokens` with `_id: 'ordersTrackerCache'`

## Files

| File | Purpose |
|---|---|
| `apps/web/services/jobs/definitions/orders-tracker-cache/ordersTrackerCache.ts` | Job definition + handler (the brains) |
| `apps/web/services/jobs/definitions/orders-tracker-cache/ordersTrackerCache.test.ts` | Job-level tests |
| `apps/web/utils/api/orders/ordersTrackerCache.ts` | Helpers: `extractRefs`, `extractSources`, `mergeRefs`, `extractDocValuesWithPaths` |
| `apps/web/utils/api/orders/ordersTrackerCache.test.ts` | Helper unit tests |
| `apps/web/services/jobs/definitions/index.ts:78,138` | Registered via `defineOrdersTrackerCacheJob()` in `defineAllJobs()` |
| `apps/web/services/jobs/scheduler.ts:843-851` | Scheduling logic at worker startup |

## How it works

### 1. Startup (`jobHandler` in `ordersTrackerCache.ts:203`)

1. Sets up a `setInterval` that calls `job.touch()` every 2.5 min to renew the Pulse lock (`lockLifetime: 5 min`, `concurrency: 1`).
2. Loads the previous change-stream resume token from `ResumeTokens`.
3. Cleans up orphaned `orderstrackercache_tmp_*` collections from prior crashed runs.
4. **Full rebuild** (`fullRebuild`, line 114):
   - Calls `getOrders({ category, filterDeleted: false, returnQuery: true, includeEligibility: true })` for each of `new`, `maintenance`, `lead` orders.
   - Writes results into a temp collection `orderstrackercache_tmp_<instanceId>`.
   - Creates indexes (`_refs`, unique `(_originalId, category)`, plus copies of any indexes from the existing cache).
   - **Atomically swaps** the temp into `orderstrackercache` via `rename(..., { dropTarget: true })`.
5. Each cached doc carries `_originalId`, `_lastCacheUpdated`, `_refs`, plus the original projected fields.
6. From the captured pipeline queries, `extractSources` walks `$lookup` / `$unionWith` / `$graphLookup` to discover **all** collections referenced (e.g. `hospitalSystems`, `patient_insurances`, `insurance_plans`, etc.) and adds them to the watch list.

### 2. Change tracking (`openChangeStream`, line 184)

Opens a single MongoDB change stream over **all source collections** (base + lookups) filtered to `insert/update/replace/delete`, resuming from the saved token if present.

For each change (`handleChange`, line 400):

- **Base collection change** (`neworders` / `maintenanceorders` / `leadorders`):
  - Schedule a **targeted rebuild** of that one order via `rebuildSingleOrder` (which re-runs the same aggregation but with `$match: { _id: documentKey }` appended, line 73).
  - Also fan out to any *other* cached orders that reference this `_id` in their `_refs`.
  - On `delete`: removes the cache row immediately.
- **Lookup collection change** (e.g. a `patient_insurances` doc updates):
  - `findAffectedOrdersByDocValues` reads the changed doc, extracts every ObjectId/string value, and queries the cache for `_refs: { $in: [...] }` — that's how it knows which orders need to be rebuilt.
  - Plus `findAffectedOrders` matches by the doc's own `_id` against `_refs`.
- All targeted rebuilds are **debounced 100ms** and deduped per `(category, _id)`.

### 3. Resilience

- If the change stream errors with `NonResumableChangeStreamError` / `ChangeStreamHistoryLost` / code 280/286, it clears the resume token, runs another `fullRebuild`, then resumes watching.
- During a full rebuild (`fullRunning`), incoming changes are queued (`pendingLookupChanges`) and rediscovered once the rebuild finishes.
- `SIGTERM` / `SIGINT` close the stream and clear the lock interval.

The handler returns `new Promise<void>(() => {})` — the job **never resolves**, it runs forever like a daemon.

## How it's triggered

In `scheduler.ts:843-851`, on every worker startup:

```ts
const pulse = await getPulse();
await pulse.cancel({ name: ORDERS_TRACKER_CACHE_JOB_NAME });
await scheduleImmediateJob(
  ORDERS_TRACKER_CACHE_JOB_NAME,
  {},
  { priority: JobPriority.Critical }
);
```

Any prior instance is cancelled and a new one is enqueued immediately at Critical priority. So **starting the worker = starting the job**.

You can also trigger manually via the `/admin/jobs` UI.

## Testing locally

**Prerequisite:** MongoDB **must be a replica set** — `db.watch()` change streams don't work on standalone mongod. If you're using Atlas, you're fine. Local mongod needs `--replSet rs0` and `rs.initiate()`.

1. Start the worker from the repo root:
   ```bash
   npm run worker
   ```
2. Watch the logs. You should see:
   - `OrdersTrackerCache: Full rebuild triggered`
   - `OrdersTrackerCache: Initial materialization complete.`
   - `OrdersTrackerCache: Watching collections: [...]`
   - `OrdersTrackerCache: Listening for changes...`
3. Verify the cache exists:
   ```js
   db.orderstrackercache.countDocuments()
   db.orderstrackercache.findOne()   // should have _originalId, _refs, _lastCacheUpdated, category
   ```
4. **Test incremental updates:** modify a doc in `neworders` / `maintenanceorders` / `leadorders`, or in any lookup collection (e.g. `patient_insurances`, `hospitalSystems`). You should see a log like:
   ```
   OrdersTrackerCache: Change detected in neworders:<id> (update)
   OrdersTrackerCache: Targeted rebuild of N order(s) in Xms
   ```
   And `_lastCacheUpdated` on the affected cache doc should bump.
5. **Test full-rebuild path:** stop the worker, delete the resume token (`db.ResumeTokens.deleteOne({_id: 'ordersTrackerCache'})`), restart — it'll do a full rebuild from scratch.
6. Run unit tests:
   ```bash
   npm run test -- ordersTrackerCache
   ```
   This covers both the helpers (`utils/api/orders/ordersTrackerCache.test.ts`) and the job (`definitions/orders-tracker-cache/ordersTrackerCache.test.ts`).

## Gotchas

- The temp collection name embeds a per-process `INSTANCE_ID = new ObjectId().toString()` (line 27), so each worker restart leaves a unique temp namespace. The orphan-cleanup step (lines 587-600) only sweeps temps that don't match the current instance — so if you crash mid-rebuild the leftovers are tidied on the next start.
- The job runs **forever** (`return new Promise<void>(() => {})`). It is not a periodic batch job — Pulse keeps the lock alive via `job.touch()` while the change stream is open.
- Field projections are not applied by `getOrders` here; the whole shape from the aggregation is cached, including `_refs` produced by the pipeline merged with `_refs` extracted from the doc itself.
