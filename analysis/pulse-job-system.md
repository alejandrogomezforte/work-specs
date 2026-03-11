# Pulse Job System — Architecture Reference

## What Is Pulse?

Pulse is a fork of [Agenda](https://github.com/agenda/agenda), a lightweight job scheduling library for Node.js that uses **MongoDB as its backing store**. In this codebase it's imported from `@pulsecron/pulse`.

It provides:
- Immediate, delayed, and recurring (cron) job scheduling
- MongoDB-backed persistence (jobs survive restarts)
- Lock-based coordination (prevents duplicate execution)
- Retry mechanisms and concurrency control

---

## Two-Process Architecture

The system runs as **two separate processes** that communicate through a shared MongoDB collection (`pulseJobs`):

```
┌──────────────────────────┐         ┌──────────────────────────┐
│    WEB APP (Next.js)     │         │   WORKER (worker.ts)     │
│    npm run dev:web       │         │   npm run worker         │
│                          │         │                          │
│  Schedules jobs only     │         │  Executes jobs only      │
│  (writes to MongoDB)     │         │  (reads from MongoDB)    │
└───────────┬──────────────┘         └───────────┬──────────────┘
            │                                    │
            └──────────┐    ┌────────────────────┘
                       ▼    ▼
              ┌──────────────────────┐
              │   MongoDB collection │
              │     "pulseJobs"      │
              │   (the job queue)    │
              └──────────────────────┘
```

### Web App (Schedule Only)

The Next.js app initializes Pulse in **web mode** — it loads job definitions (so it knows what job names exist) but does **not** start processing. API routes and webhook handlers use scheduling functions to enqueue jobs.

### Worker Process (Execute Only)

`apps/web/worker.ts` is a **standalone Node.js process** that initializes Pulse in **worker mode** — it loads job definitions, sets up recurring schedules, cleans stale locks, and **starts Pulse** to poll the queue every 3 seconds.

**Script:** `npm run worker` → `ts-node -r tsconfig-paths/register worker.ts`

---

## Pulse Configuration

**File:** `apps/web/services/jobs/pulse.ts`

```typescript
pulseInstance = new Pulse({
  db: {
    address: mongoConnectionString,
    collection: process.env.PULSE_COLLECTION || 'pulseJobs',
  },
  processEvery: '3 seconds',    // Worker polls every 3 seconds
  maxConcurrency: 20,           // Max 20 jobs running at once
  defaultLockLifetime: 3600000, // 1 hour safety net for stuck jobs
});
```

| Setting | Value | Purpose |
|---------|-------|---------|
| `processEvery` | `3 seconds` | How often the worker checks for new jobs |
| `maxConcurrency` | `20` | Maximum simultaneous jobs across all types |
| `defaultLockLifetime` | `1 hour` | Auto-release lock if job doesn't call `done()` |
| `collection` | `pulseJobs` | MongoDB collection name (configurable via env) |

The Pulse instance is a **singleton** — `getPulse()` creates it once and returns the same instance on subsequent calls.

---

## Dual-Mode Initialization

**File:** `apps/web/services/jobs/initialize-jobs.ts`

```typescript
// Web mode: Define job types only (enable scheduling, no processing)
export const initializeJobs = async (): Promise<void> => {
  await defineAllJobs();
  await getPulse();
  initialized = true;
};

// Worker mode: Define jobs + set up recurring + clean locks + start processing
export const initializeAndStartPulse = async (): Promise<void> => {
  await initializeJobs();

  if (process.env.WORKER_PROCESS === 'true') {
    await setupRecurringJobs();
    await cleanupStaleLocks();
    const pulse = await getPulse();
    await pulse.start();  // ← Only the worker calls this
  }
};
```

The environment variable `WORKER_PROCESS=true` determines which mode to use.

---

## Job Lifecycle

```
1. SCHEDULE          2. STORE               3. EXECUTE              4. COMPLETE
(web app)            (MongoDB)              (worker)                (MongoDB)

API route calls      pulseJobs doc:         Worker picks up job:    Success or failure:
scheduleImmediateJob { name, data,          - Acquires lock         - lastFinishedAt set
       │               nextRunAt: now,      - Runs handler          - lockedAt cleared
       │               lockedAt: null }     - Calls done()          - Retry if failed
       ▼                    ▼                     ▼                       ▼
  pulse.create()     Saved to MongoDB      handler(job, done)      done() / done(error)
  job.save()
```

### MongoDB Document Shape (`pulseJobs` collection)

```javascript
{
  _id: ObjectId,
  name: "appointmentReminder",        // Job type name
  data: { /* job-specific payload */ },
  priority: 0,                         // -10 (low) to 20 (critical)

  // Scheduling
  nextRunAt: ISODate("2026-03-02T14:00:00Z"),
  lockedAt: null,                      // Set during execution
  lastRunAt: ISODate("2026-03-01T14:00:00Z"),
  lastFinishedAt: ISODate("2026-03-01T14:00:05Z"),

  // Retry
  attemptsMade: 0,
  failCount: 0,
  failedAt: null,

  // Recurring
  repeatInterval: "0 9 * * *",        // Cron expression (null for one-time)
  repeatTimezone: "America/New_York",
}
```

---

## Scheduling Functions

**File:** `apps/web/services/jobs/scheduler.ts`

All scheduling functions call `ensureJobsInitialized()` first, which lazy-loads job definitions on the web app side. During tests and build time, this is a no-op.

### `scheduleImmediateJob(name, data, options?)`

Runs as soon as the worker picks it up (within ~3 seconds).

```typescript
export const scheduleImmediateJob = async <T extends JobAttributesData>(
  name: string,
  data: T = {} as T,
  options: { priority?: JobPriority } = {}
): Promise<Job<T>> => {
  await ensureJobsInitialized();
  const pulse = await getPulse();
  const job = pulse.create(name, data);
  if (options.priority) {
    job.priority(String(options.priority));
  }
  await job.save();
  return job;
};
```

**Use case:** Event-driven jobs — "something changed, recalculate now."

### `scheduleJobForLater(name, when, data, options?)`

Runs at a specific future time.

```typescript
export const scheduleJobForLater = async <T extends JobAttributesData>(
  name: string,
  when: Date | string,
  data: T = {} as T,
  options: { priority?: JobPriority } = {}
): Promise<Job<T>> => {
  await ensureJobsInitialized();
  const pulse = await getPulse();
  const job = pulse.create(name, data);
  job.schedule(when);
  if (options.priority) {
    job.priority(String(options.priority));
  }
  await job.save();
  return job;
};
```

**Use case:** Delayed jobs — "send a reminder in 2 hours."

### `scheduleRecurringJob(name, interval, data, options?)`

Runs on a cron schedule indefinitely. Prevents duplicates by checking for existing recurring jobs with the same name.

```typescript
export const scheduleRecurringJob = async <T extends JobAttributesData>(
  name: string,
  interval: string,
  data: T = {} as T,
  options: {
    skipImmediate?: boolean;
    priority?: JobPriority;
    timezone?: string;
  } = {}
): Promise<Job<T>> => {
  await ensureJobsInitialized();
  const pulse = await getPulse();

  // Prevent duplicates
  const existingJobs = await pulse.jobs({
    name,
    repeatInterval: { $exists: true },
  });
  if (existingJobs.length > 0) {
    return existingJobs[0];
  }

  const job = pulse.create(name, data);
  job.repeatEvery(interval, {
    skipImmediate: options.skipImmediate ?? true,
    timezone: options.timezone,
  });
  if (options.priority) {
    job.priority(String(options.priority));
  }
  await job.save();
  return job;
};
```

**Use case:** Cron jobs — "run daily at 9 AM."

### Example Recurring Schedules

| Job | Cron | Schedule |
|-----|------|----------|
| `appointmentReminder` | `0 9 * * *` | 9 AM EST daily |
| `downloadRxPreferredClaims` | `0 3 13 * *` | 3 AM EST on the 13th |
| `ordersTrackerExportNewOrders` | `*/30 * * * *` | Every 30 minutes |
| `skyriziObiTraining` | `0 18 * * 1-5` | 6 PM ET weekdays |

---

## Job Types & Handler Signature

**File:** `apps/web/services/jobs/types.ts`

### Priority Enum

```typescript
export enum JobPriority {
  Low = -10,
  Normal = 0,
  High = 10,
  Critical = 20,
}
```

### Handler Type

```typescript
export type JobHandler<T extends JobAttributesData = JobAttributesData> = (
  job: Job<T>,
  done: (error?: Error, result?: string) => void
) => Promise<void>;
```

- `job.attrs.data` — the typed payload passed when scheduling
- `done()` — call with no args for success
- `done(error)` — call with an Error for failure (triggers retry if configured)

### Base Job Data

```typescript
export interface BaseJobData {
  metadata?: JobMetadata;
  retryAttempt?: number;
  maxRetries?: number;
  retryReason?: string;
}

export interface JobMetadata {
  triggeredBy?: string;
  webhookId?: string;
  receivedAt?: Date;
  source?: 'webhook' | 'api' | 'system' | 'schedule';
  originalTransactionId?: string;
}
```

---

## Job Definition Pattern

**File:** `apps/web/services/jobs/definitions/`

Each job type has a **definition file** that exports:
1. A handler function
2. A `define*Jobs()` registration function

### Example: Appointment Reminder

```typescript
// apps/web/services/jobs/definitions/appointmentReminder.ts

export const appointmentReminder: JobHandler = async (_job: Job, done) => {
  const { force } = _job.attrs.data;

  // 1. Check feature flag (skip gracefully if disabled)
  if (
    !(await FeatureFlagService.getFeatureFlag(FeatureFlag.APPOINTMENT_REMINDER)) &&
    !force
  ) {
    done();
    return;
  }

  // 2. Check preconditions
  const range = isTimeOutsideWorkingRange();
  if (range) {
    done();
    return;
  }

  try {
    // 3. Main logic
    const from = dayAtMidnightAfter(2);
    const to = dayAtMidnightAfter(3);
    const cursor = await pendingAppointments({ from, to });
    const results = await cursor.toArray();

    for (const appointment of results) {
      await setAppointmentReminderMessage(appointment._id);
    }

    // 4. Success
    done();
  } catch (error) {
    // 5. Failure (may trigger retry)
    done(error as Error);
  }
};

export const defineAppointmentReminderJobs = async (): Promise<void> => {
  await defineJob('appointmentReminder', appointmentReminder, {
    priority: 'high',
    concurrency: 1,
    lockLifetime: 1800000, // 30 minutes
  });
};
```

### Handler Pattern Summary

1. Read `job.attrs.data` for the payload
2. Check feature flags / preconditions → `done()` to skip gracefully
3. Execute main logic inside `try-catch`
4. Call `done()` on success
5. Call `done(error)` on failure

### Job Registration

**File:** `apps/web/services/jobs/definitions/index.ts`

```typescript
export const defineAllJobs = async (): Promise<void> => {
  await definePatientJobs();
  await defineFinancialJobs();
  await defineAppointmentReminderJobs();
  await defineRxPreferredClaimsJobs();
  // ... ~30 more ...
  logger.info('All job definitions registered successfully');
};
```

### `defineJob()` Options

```typescript
await defineJob('jobName', handler, {
  concurrency: 3,           // Max 3 instances of this job type at once
  lockLifetime: 300000,     // 5 min lock timeout for this job
  priority: 'high',         // Default priority for this job type
  attempts: 3,              // Retry up to 3 times on failure
  backoff: {
    type: 'exponential',    // or 'fixed'
    delay: 10000,           // Base delay between retries (ms)
  },
});
```

---

## Worker Startup Sequence

**File:** `apps/web/worker.ts`

```
1. Start health check HTTP server (port 8080)
2. Connect to MongoDB
3. Call initializeAndStartPulse():
   a. Define all job types
   b. Set up recurring jobs (cron schedules)
   c. Clean up stale locks from previous crashes
   d. Start Pulse → begin polling for jobs
4. Mark worker as ready (health probes pass)
5. Start periodic metrics logging (every 60 seconds)
```

### Health Probes

| Endpoint | Check |
|----------|-------|
| `GET /health/liveness` | Worker alive and processing (fails if idle > 10 minutes) |
| `GET /health/readiness` | Pulse initialized and ready |
| `GET /health/startup` | HTTP server is listening |

### Graceful Shutdown

On `SIGTERM` or `SIGINT`:

1. Mark not-ready (fail health probes immediately)
2. Stop Pulse — wait for in-flight jobs to complete (50 second timeout)
3. Grace period for final cleanup (5 seconds)
4. Stop health server
5. Disconnect from MongoDB
6. Exit

---

## Stale Lock Cleanup

**File:** `apps/web/services/jobs/stale-lock-cleanup.ts`

Runs on worker startup, **before** Pulse begins processing. Clears locks older than 2 hours — these are left behind by unclean shutdowns (SIGKILL, OOM, crash).

```typescript
const MAX_LOCK_LIFETIME_MS = 7200000; // 2 hours

export const cleanupStaleLocks = async (): Promise<void> => {
  const cutoffDate = new Date(Date.now() - MAX_LOCK_LIFETIME_MS);

  const result = await db.collection(collectionName).updateMany(
    { lockedAt: { $ne: null, $lt: cutoffDate } },
    { $set: { lockedAt: null } }
  );

  if (result.modifiedCount > 0) {
    logger.info(`Cleaned up ${result.modifiedCount} stale job lock(s)`);
  }
};
```

Without this, jobs locked by a crashed worker would stay locked forever (until `defaultLockLifetime` expires).

---

## Who Schedules Jobs?

Jobs are scheduled from multiple entry points in the web app:

| Source | Example |
|--------|---------|
| **API routes** | Clinical review analysis → `scheduleImmediateJob('analyzeClinicalReview', ...)` |
| **Webhook handlers** | Patient insurance webhook → `scheduleImmediateJob('updatePatientInsuranceStatus', ...)` |
| **Recurring (automatic)** | Appointment reminders → `scheduleRecurringJob('appointmentReminder', '0 9 * * *', ...)` |
| **Job trigger API** | `POST /api/jobs { name, data }` — allows frontend to schedule jobs directly |
| **Service layer** | Any service can call `scheduleImmediateJob()` |

---

## Key Files Reference

| Component | File |
|-----------|------|
| Pulse singleton & config | `apps/web/services/jobs/pulse.ts` |
| Scheduling functions | `apps/web/services/jobs/scheduler.ts` |
| Dual-mode initialization | `apps/web/services/jobs/initialize-jobs.ts` |
| Job type definitions | `apps/web/services/jobs/types.ts` |
| Job handler definitions | `apps/web/services/jobs/definitions/` |
| Job registration index | `apps/web/services/jobs/definitions/index.ts` |
| Worker process | `apps/web/worker.ts` |
| Stale lock cleanup | `apps/web/services/jobs/stale-lock-cleanup.ts` |
| Health server | `apps/web/services/worker-health.ts` |
