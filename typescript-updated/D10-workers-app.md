# Skill: TypeScript — D10: Workers App

## Summary

Backend job processing layer built with **BullMQ 5.77.3** (stable, May 2026). The workers app is an independently deployable Node.js process with no HTTP server — it pulls jobs from Redis queues, processes them, and exits cleanly on shutdown signals. It shares no runtime state with `/apps/backend` beyond the Redis instance and `@packages/shared` types.

All job payload types come from `@shared/jobs` (D5). Workers import from `@shared`, never define their own payload shapes. Zod validates job payloads at the consumption boundary (D4). The three-way type contract — D5 payload → D8 enqueue → D10 consume — is enforced at compile time and verified at runtime.

**Audit risk:** Moderate (all 6 flagged gaps resolved). **Dependencies:** D1 (TS 6.0.3), D2 (compiler), D3 (monorepo), D4 (Zod 4.4.3), D5 (shared types/job payloads), D6 (testing with mocked BullMQ), D7 (ESLint no-restricted-imports), D8 (backend enqueuing). Knowledge cutoff: 2026-05-25.

**Three-way type contract status: CONSISTENT.** D5 defines `JobPayload` discriminated union with `JobName` string literals. D8 enqueues via `Queue<JobPayload, void, JobName>`. D10 consumes via `Worker<JobPayload, void, JobName>`. Zod schemas from `@shared/jobs` validate at the worker boundary. Graceful shutdown pattern matches D8 (force-exit timeout, SIGTERM/SIGINT, connection cleanup).

## Version landscape

| Package | Version | Notes |
|---|---|---|
| BullMQ | **5.77.3** (May 2026) | Current stable major. No v6 exists. v5.x is the current major. |
| ioredis | 5.x (auto-installed with BullMQ) | BullMQ's default Redis client. Can also use `node-redis` via adapter. |
| croner | **10.0.1** (Feb 2026) | Recommended for system-level scheduled tasks. v10 removed default export — use named `{ Cron }` + `new`. |
| node-cron | 4.2.1 (Jul 2025) | Stable alternative. croner preferred for this architecture (see Scheduled tasks). |
| TypeScript | 6.0.3 (D1) | Baseline 5.5+ as declared in scope. |
| Node.js | 20+ LTS | BullMQ 5 requires Node.js 18+. Project uses LTS consistent with D3. |

### BullMQ v5 — no v6 exists

As of May 2026, BullMQ is on v5.77.x. There is no v6. The v5 API has been stable with over 77 minor releases in the 5.x line. Key characteristics of the current API:

- **QueueScheduler: REMOVED in v2.0 (September 2022).** Do not import or reference it. The Worker class handles delayed jobs, stalled jobs, and retries internally. Any code referencing `QueueScheduler` is targeting BullMQ v1.x and is wrong for current versions.
- **Generics:** `Queue<DataType, ResultType, NameType>` and `Worker<DataType, ResultType, NameType>` — three type parameters, stable since v3.x.
- **Repeatable jobs:** `queue.upsertJobScheduler()` (v5.16.0+) replaces the legacy `queue.add()` with `repeat` option. The older `repeat` option on `add()` still works but `upsertJobScheduler` is the recommended modern API.
- **JobScheduler:** A new class introduced in v5.16.0 for managing repeatable job schedules. Not to be confused with the removed `QueueScheduler`. `JobScheduler` is for creating/updating/deleting repeatable job definitions; `QueueScheduler` was for stalled job detection (now built into Worker).
- **SandboxedJob type:** Available as a named import since early v5.x. Fully typed as of v5.73.1.
- **AbortSignal in processors:** Worker passes an `AbortSignal` as the third processor parameter, triggered on `worker.close()`.
- **`job.updateProgress()`:** Accepts `string | number | boolean | object` (expanded from `number | object`).

Source: [BullMQ GitHub releases](https://github.com/taskforcesh/bullmq/releases), [BullMQ v2.0 breaking changes](https://github.com/taskforcesh/bullmq/pull/1413)

## Typed job patterns with @shared

### The three-way contract (D5 ↔ D8 ↔ D10)

The job payload contract lives in `@shared/src/jobs/index.ts` (D5). The backend enqueues with it (D8). The workers consume with it (D10). All three import the same types:

```
@shared/src/jobs/index.ts  ← canonical source
    ↑                    ↑
    │                    │
/apps/backend          /apps/workers
(enqueues)             (consumes)
```

### Queue and Worker generics

BullMQ v5 uses three generic type parameters:

```ts
Queue<DataType, ResultType, NameType>
Worker<DataType, ResultType, NameType>
```

| Parameter | What it types | Typical value from @shared |
|---|---|---|
| `DataType` | `job.data` in the processor | `JobPayload` (the discriminated union) |
| `ResultType` | Return value of the processor (`job.returnvalue`) | `void` if fire-and-forget, or a result type |
| `NameType` | `job.name` string literal union | `JobName` (e.g., `"send-email" \| "send-notification"`) |

### Discriminated union pattern

The BullMQ maintainers recommend discriminating on a field inside `job.data` rather than relying on `job.name` for TypeScript narrowing. D5 already uses this pattern: each payload has a `jobName` literal field that serves as the discriminant.

```ts
// @shared/src/jobs/index.ts (from D5 — canonical source)
export interface EmailJobPayload {
  version: 1;
  jobName: "send-email";
  to: string;
  subject: string;
  body: string;
  templateId?: string;
}

export interface NotificationJobPayload {
  version: 1;
  jobName: "send-notification";
  userId: string;
  title: string;
  message: string;
  channel: "push" | "in-app";
}

export type JobPayload = EmailJobPayload | NotificationJobPayload;

export const JOB_NAMES = ["send-email", "send-notification"] as const;
export type JobName = (typeof JOB_NAMES)[number];
```

### Type narrowing inside the handler

Discriminate on `job.data.jobName` (the data field), which TypeScript narrows correctly:

```ts
import { Worker } from "bullmq";
import type { JobPayload, JobName } from "@shared/jobs";

const worker = new Worker<JobPayload, void, JobName>(
  "notifications",
  async (job) => {
    switch (job.data.jobName) {
      case "send-email": {
        // job.data narrowed to EmailJobPayload
        // job.data.to, job.data.subject, job.data.body are typed
        await sendEmail(job.data);
        break;
      }
      case "send-notification": {
        // job.data narrowed to NotificationJobPayload
        // job.data.userId, job.data.title, job.data.message, job.data.channel are typed
        await sendPushNotification(job.data);
        break;
      }
    }
  },
  { connection }
);
```

**Why `job.data.jobName` and not `job.name`:** BullMQ's TypeScript generics do not distribute `NameType` across the `DataType` union. This means `switch (job.name)` does not narrow `job.data` — TypeScript sees `job.data` as `JobPayload` regardless of which `case` branch you're in. The discriminant must be a field within `job.data` itself. This is the official BullMQ maintainer recommendation from [GitHub issue #1156](https://github.com/taskforcesh/bullmq/issues/1156).

### Zod validation at consumption boundary

Workers validate job payloads at runtime using the Zod schemas from `@shared/jobs`. Even though the backend validated at enqueue time (D8), workers are the defensive boundary:

```ts
import { EmailJobPayloadSchema, NotificationJobPayloadSchema } from "@shared/jobs";

const worker = new Worker<JobPayload, void, JobName>(
  "notifications",
  async (job) => {
    // Validate before processing — catch serialization issues, version mismatches
    switch (job.data.jobName) {
      case "send-email": {
        const parsed = EmailJobPayloadSchema.safeParse(job.data);
        if (!parsed.success) {
          logger.error("Invalid email job payload", {
            jobId: job.id,
            errors: parsed.error.flatten(),
          });
          throw new UnrecoverableError(
            `Invalid payload for job ${job.id}: ${parsed.error.message}`
          );
        }
        // parsed.data is fully typed as EmailJobPayload — no casting
        await sendEmail(parsed.data);
        break;
      }
      case "send-notification": {
        const parsed = NotificationJobPayloadSchema.safeParse(job.data);
        if (!parsed.success) {
          logger.error("Invalid notification job payload", {
            jobId: job.id,
            errors: parsed.error.flatten(),
          });
          throw new UnrecoverableError(
            `Invalid payload for job ${job.id}: ${parsed.error.message}`
          );
        }
        await sendPushNotification(parsed.data);
        break;
      }
    }
  },
  { connection }
);
```

**Why validate at consumption:**
- Serialization/deserialization issues (dates become strings in Redis)
- Version mismatches during rolling deployments (old worker receives new payload fields)
- Queue corruption or manual Redis manipulation
- The Zod schema is the runtime enforcement of the type contract

Invalid payloads throw `UnrecoverableError` — they should not be retried because the data itself is wrong.

### Shared validation wrapper (DRY)

Extract the validation pattern into a reusable wrapper to avoid repeating it in every handler:

```ts
// apps/workers/src/workers/base.worker.ts
import type { ZodSchema } from "zod/v4";
import { UnrecoverableError } from "bullmq";

export function validatePayload<T>(
  schema: ZodSchema<T>,
  data: unknown,
  jobId?: string,
): T {
  const result = schema.safeParse(data);
  if (!result.success) {
    throw new UnrecoverableError(
      `Invalid payload for job ${jobId ?? "unknown"}: ${result.error.message}`
    );
  }
  return result.data;
}
```

Usage in processor:

```ts
const parsed = validatePayload(EmailJobPayloadSchema, job.data, job.id);
// parsed is typed as EmailJobPayload
```

## Worker setup and job processing

### Worker instantiation

```ts
import { Worker } from "bullmq";
import type { JobPayload, JobName } from "@shared/jobs";

const worker = new Worker<JobPayload, void, JobName>(
  // Queue name — matches the queue name used by backend when enqueuing
  "notifications",

  // Processor function — called for each job
  async (job) => {
    // job: Job<JobPayload, void, JobName>
    // job.data: JobPayload (discriminated union)
    // job.name: JobName (string literal)
    // job.id: string | undefined
    // job.attemptsMade: number
    // job.opts: JobsOptions (the options the job was added with)
  },

  // Worker options
  {
    connection, // Redis connection (shared instance)
    concurrency: 10, // Process up to 10 jobs simultaneously
    // limiter: { max: 100, duration: 1000 }, // Rate limit: 100 jobs/second
    // lockDuration: 30000, // Job lock TTL in ms (prevents stalled-job double-processing)
    // stalledInterval: 30000, // How often to check for stalled jobs
    // maxStalledCount: 1, // Max times a job can be stalled before permanent failure
    // drainDelay: 5, // Wait time before pulling new jobs when queue is empty
  }
);
```

### Key worker options

| Option | Type | Default | Purpose |
|---|---|---|---|
| `connection` | `Redis \| IORedis` | (required) | Redis connection |
| `concurrency` | `number` | `1` | Jobs processed simultaneously by this worker |
| `limiter` | `{ max, duration }` | none | Rate limit: max jobs per duration (ms). Group-level limiting via `groupKey`. |
| `lockDuration` | `number` | `30000` | How long a job lock is held (ms). Job must finish within this time or be considered stalled. |
| `stalledInterval` | `number` | `30000` | How often to check for stalled jobs (ms) |
| `maxStalledCount` | `number` | `1` | Max times a job can be stalled before moved to failed |
| `drainDelay` | `number` | `5` | Wait time in seconds before pulling new jobs when the queue is empty |
| `settings` | `WorkerSettings` | `{}` | Advanced settings including `backoffStrategy` |
| `autorun` | `boolean` | `true` | If false, call `worker.run()` manually to start |

### Processor function

The processor is `async (job: Job<DataType, ResultType, NameType>, token?: string, signal?: AbortSignal) => ResultType`:

```ts
const worker = new Worker<JobPayload, void, JobName>(
  "notifications",
  async (job, token, signal) => {
    // token: string — used as the lock token for the job (rarely needed directly)
    // signal: AbortSignal | undefined — aborted when worker.close() is called

    // Listen for shutdown signal to clean up mid-job resources
    signal?.addEventListener("abort", () => {
      // Close DB connections, cancel HTTP requests, etc.
      logger.info(`Job ${job.id} received abort signal during processing`);
    });

    // Access typed job data
    const { jobName } = job.data;

    // Report progress (0-100 or arbitrary value)
    await job.updateProgress(50);

    // Log to the job's log
    await job.log(`Processing step for ${jobName}`);

    // Do the work
    await processStep();

    await job.updateProgress(100);
    // Implicit return — result is stored as job.returnvalue
  },
  { connection }
);
```

### Accessing job properties

| Property | Type | Description |
|---|---|---|
| `job.data` | `DataType` | The payload passed to `queue.add()` — fully typed |
| `job.name` | `NameType` | The job name string literal |
| `job.id` | `string \| undefined` | Unique job ID (auto-generated or custom) |
| `job.attemptsMade` | `number` | How many attempts have been made (0 on first run) |
| `job.attemptsStarted` | `number` | Total times the job moved to active (includes rate-limit restarts) |
| `job.opts` | `JobsOptions` | The options the job was enqueued with |
| `job.returnvalue` | `ResultType` | The value returned by the processor (only if the job completed) |
| `job.progress` | `JobProgress` | Last reported progress value |
| `job.finishedOn` | `number \| undefined` | Timestamp when the job completed |
| `job.processedOn` | `number \| undefined` | Timestamp when the job started processing |

### Progress reporting

```ts
await job.updateProgress(42);                          // number
await job.updateProgress("step-3-complete");            // string
await job.updateProgress({ step: 3, total: 10 });      // object
```

Progress is stored in Redis as a JSON-serialized value. Any value that survives `JSON.stringify` / `JSON.parse` is valid.

### Job logging

```ts
await job.log("Starting image processing...");
await job.log(`Processed chunk ${i + 1}/${total}`);
```

Returns `Promise<number>` — the total number of log entries for this job. Control log retention with `keepLogs` option when adding the job. Logs are stored in Redis and persist for the job's lifetime (until removed by `removeOnComplete`/`removeOnFail`).

## Queue setup and enqueueing (reference)

This section documents the pattern used by `/apps/backend` (D8). Workers do NOT enqueue jobs — they only consume. This is provided as a reference only.

### Queue instantiation

```ts
import { Queue } from "bullmq";
import type { JobPayload, JobName } from "@shared/jobs";

const notificationQueue = new Queue<JobPayload, void, JobName>(
  "notifications",
  {
    connection, // Shared Redis connection
    defaultJobOptions: {
      attempts: 3,
      backoff: { type: "exponential", delay: 1000 },
      removeOnComplete: { age: 3600, count: 1000 },
      removeOnFail: { age: 86400, count: 5000 },
    },
  }
);
```

### queue.add() — single job

```ts
await notificationQueue.add(
  "send-email",          // NameType — must be a valid JobName
  {                       // DataType — must match EmailJobPayload shape
    version: 1,
    jobName: "send-email",
    to: "user@example.com",
    subject: "Welcome",
    body: "Thanks for signing up!",
  },
  {                       // JobsOptions
    delay: 0,             // Delay before processing (ms)
    attempts: 5,          // Override default attempts
    backoff: { type: "fixed", delay: 5000 },
    priority: 1,          // 0 = highest, 2,097,152 = lowest
    removeOnComplete: true,
    removeOnFail: 1000,
  }
);
```

### queue.addBulk() — multiple jobs

```ts
await notificationQueue.addBulk([
  {
    name: "send-email",
    data: { version: 1, jobName: "send-email", to: "a@example.com", subject: "A", body: "..." },
    opts: { jobId: "email-a" },
  },
  {
    name: "send-email",
    data: { version: 1, jobName: "send-email", to: "b@example.com", subject: "B", body: "..." },
    opts: { jobId: "email-b" },
  },
  {
    name: "send-notification",
    data: { version: 1, jobName: "send-notification", userId: "uuid", title: "Alert", message: "...", channel: "push" },
    opts: { delay: 60000 }, // Deliver in 1 minute
  },
]);
```

`addBulk` is the most efficient way to enqueue many jobs — it pipelines Redis commands.

### Job naming conventions

Job names come from the `JOB_NAMES` const array in `@shared/jobs` (D5). The backend and workers must use the same names:

```ts
// Both backend and workers import from @shared
import { JOB_NAMES, type JobName } from "@shared/jobs";

// TypeScript enforces that only valid names are used
queue.add("send-email", payload);     // ✅ TypeScript-verified
queue.add("invalid-job", payload);    // ❌ TypeScript error
```

## Error handling and retries

### Throwing in a processor

Any error thrown in the processor triggers BullMQ's retry mechanism (if `attempts > 1`):

```ts
const worker = new Worker<JobPayload, void, JobName>(
  "notifications",
  async (job) => {
    const result = await externalApi.call();
    if (result.status === "error") {
      throw new Error(`API call failed: ${result.message}`);
      // BullMQ will retry based on backoff config
    }
  },
  { connection }
);
```

Exceptions that do NOT count as an attempt:
- `RateLimitError` — retried later, does not increment `attemptsMade`
- `DelayedError` — retried later, does not increment `attemptsMade`
- `WaitingChildrenError` — waits for child jobs, does not increment `attemptsMade`

### UnrecoverableError

`UnrecoverableError` marks a job as permanently failed — it skips all remaining retry attempts and moves directly to the failed set:

```ts
import { UnrecoverableError } from "bullmq";

const worker = new Worker<JobPayload, void, JobName>(
  "notifications",
  async (job) => {
    switch (job.data.jobName) {
      case "send-email": {
        const parsed = EmailJobPayloadSchema.safeParse(job.data);
        if (!parsed.success) {
          // Bad data — retrying won't fix it
          throw new UnrecoverableError(
            `Invalid email payload: ${parsed.error.message}`
          );
        }

        const result = await emailService.send(parsed.data);
        if (result.error === "INVALID_ADDRESS") {
          // Permanent failure — retrying won't fix a bad email
          throw new UnrecoverableError(`Invalid email address: ${parsed.data.to}`);
        }

        if (result.error === "RATE_LIMITED") {
          // Transient failure — retry with backoff
          throw new Error(`Rate limited for ${parsed.data.to}`);
        }
        break;
      }
    }
  },
  { connection }
);
```

### attempts and backoff

```ts
await queue.add("send-email", payload, {
  attempts: 5,                              // Try up to 5 times
  backoff: { type: "exponential", delay: 1000 }, // 1s, 2s, 4s, 8s, 16s
});

await queue.add("send-notification", payload, {
  attempts: 3,
  backoff: { type: "fixed", delay: 5000 },  // 5s each retry
});

// Custom backoff strategy (defined on Worker settings)
const worker = new Worker("notifications", processor, {
  connection,
  settings: {
    backoffStrategy: (attemptsMade: number, type: string, err: Error) => {
      if (err.message.includes("RATE_LIMITED")) {
        return 30_000; // 30s for rate limits
      }
      return attemptsMade * 1000; // 1s, 2s, 3s...
    },
  },
});
```

Return `-1` from `backoffStrategy` to skip retry (equivalent to `UnrecoverableError`).

### removeOnComplete / removeOnFail

Control Redis memory by auto-removing old jobs:

```ts
// Remove immediately
removeOnComplete: true,
removeOnFail: true,

// Keep at most N jobs
removeOnComplete: 1000,

// Keep based on age and count (preferred for production)
removeOnComplete: { age: 3600, count: 1000 },  // Keep up to 1h or 1000 jobs
removeOnFail: { age: 86400, count: 5000 },      // Keep up to 24h or 5000 jobs
```

Defaults keep all jobs indefinitely — this will exhaust Redis memory in production. Always configure limits.

### Failed event handler

Monitor failures globally on the Worker:

```ts
worker.on("failed", (job: Job<JobPayload, void, JobName> | undefined, err: Error) => {
  if (!job) {
    logger.error("Worker failed event without job", { err: err.message });
    return;
  }

  logger.error(`Job ${job.id} (${job.name}) failed after ${job.attemptsMade + 1} attempts`, {
    jobId: job.id,
    jobName: job.name,
    attemptsMade: job.attemptsMade,
    error: err.message,
    stack: err.stack,
    data: job.data, // Include payload for debugging — scrub PII in production
  });
});
```

Other useful worker events:

```ts
worker.on("completed", (job, result) => { /* job completed successfully */ });
worker.on("active", (job) => { /* job started processing */ });
worker.on("progress", (job, progress) => { /* job reported progress */ });
worker.on("stalled", (jobId) => { /* job stalled (worker died mid-processing) */ });
worker.on("drained", () => { /* queue is empty */ });
worker.on("error", (err) => { /* worker-level error (not job error) */ });
worker.on("closing", (msg) => { /* worker is shutting down */ });
worker.on("closed", () => { /* worker has fully closed */ });
```

### Dead-letter queue pattern

Jobs that fail with `UnrecoverableError` should be moved to a separate dead-letter queue for manual inspection:

```ts
import { Queue, Worker, UnrecoverableError } from "bullmq";

const deadLetterQueue = new Queue("notifications-dlq", { connection });

const worker = new Worker<JobPayload, void, JobName>(
  "notifications",
  async (job) => {
    try {
      await processJob(job);
    } catch (err) {
      if (err instanceof UnrecoverableError) {
        // Move to dead-letter queue with original data and error context
        await deadLetterQueue.add(job.name, {
          originalJobId: job.id,
          originalData: job.data,
          error: err.message,
          failedAt: new Date().toISOString(),
          attemptsMade: job.attemptsMade,
        });
        throw err; // Still throw so the original job moves to failed
      }
      throw err; // Transient — let BullMQ retry
    }
  },
  { connection }
);
```

This preserves the failed job for debugging while keeping the main queue clean. The dead-letter queue can have a separate retention policy (keep failed jobs for 30 days for inspection).

## Scheduled tasks

### BullMQ repeatable jobs vs croner/node-cron

| Consideration | BullMQ repeatables | croner / node-cron |
|---|---|---|
| Backed by | Redis (same infrastructure) | In-process (no external dependency) |
| Persists across restarts | Yes | No (unless state file is managed) |
| Distributed-safe | Yes (only one worker processes) | No (each process runs independently) |
| Visibility | BullMQ dashboard / queue metrics | Application logs only |
| Complexity | Depends on Redis being up | Works as long as the process is up |
| Cron features | Basic cron + custom strategies (RRULE) | Full cron features (L, W, #, seconds, years for croner) |
| Job lifecycle | Full (attempts, backoff, events, logs) | None — just a function call |
| Use case | Work pipeline tasks (cleanup, reports, sync) | System tasks (health checks, metrics flush, cache warm) |

### Recommendation for this architecture

**BullMQ repeatable jobs** for tasks that are part of the job processing pipeline: periodic cleanup, scheduled reports, data syncs, email digests. These benefit from the job lifecycle (retries, monitoring, dead-letter queue).

**croner** for system-level tasks that are not jobs: health check pings to an external monitor, metrics aggregation and flush, cache warming, temporary file cleanup.

croner is chosen over node-cron because: (a) more actively maintained (last push May 2026 vs Feb 2026), (b) 6 open issues vs 33, (c) built-in error handling and overrun protection, (d) zero dependencies, (e) richer cron feature set if needed later.

### BullMQ repeatable jobs (modern API: upsertJobScheduler)

```ts
// apps/workers/src/schedulers/daily-report.scheduler.ts
import { Queue } from "bullmq";

export async function registerDailyReportScheduler(
  queue: Queue<JobPayload, void, JobName>,
) {
  await queue.upsertJobScheduler(
    // Scheduler ID — unique identifier for this schedule
    "daily-report-scheduler",

    // Repeat options
    {
      pattern: "0 7 * * *", // Every day at 7:00 AM
      // Alternatively, use every:
      // every: 86_400_000, // Every 24 hours in ms
    },

    // Job definition
    {
      name: "send-email",
      data: {
        version: 1,
        jobName: "send-email",
        to: "admin@example.com",
        subject: "Daily Report",
        body: "Report content placeholder",
      },
      opts: {
        attempts: 3,
        backoff: { type: "fixed", delay: 60_000 }, // Retry after 1 min
      },
    },
  );
}
```

**Important:** `upsertJobScheduler` (v5.16.0+) is the recommended modern API. The older `queue.add()` with `repeat` option still works but `upsertJobScheduler` provides better idempotency — calling it multiple times with the same scheduler ID updates the existing schedule instead of creating duplicates.

### Repeatable jobs lifecycle

- Only one delayed job exists per scheduler at any time
- When the delayed job is processed, BullMQ creates the next delayed job
- If the worker is down, jobs accumulate as delayed and process when the worker comes back
- To remove a repeatable schedule: `await queue.removeJobScheduler("scheduler-id")`

### croner for system tasks

```ts
// apps/workers/src/cron/metrics-flush.cron.ts
import { Cron } from "croner";

export function registerMetricsFlush(intervalMinutes = 5) {
  return new Cron(`*/${intervalMinutes} * * * *`, {
    name: "metrics-flush",
    protect: true, // Overrun protection — skip if previous run hasn't finished
    catch: (err) => {
      logger.error("Metrics flush failed", { err: err.message });
    },
  }, async () => {
    // Flush aggregated metrics to external service
    await metricsService.flush();
  });
}
```

croner jobs are started in the application bootstrap and stopped during graceful shutdown:

```ts
const metricsJob = registerMetricsFlush();

// In graceful shutdown:
metricsJob.stop();
```

## Worker threads (CPU-bound tasks)

### When to use worker_threads vs BullMQ workers

| Factor | BullMQ Worker | Node.js worker_threads |
|---|---|---|
| Work distribution | Across processes/machines via Redis | Within a single process |
| Isolation | Process-level (separate Node.js process) | Thread-level (shared memory space) |
| Overhead | Higher (Redis round-trips, serialization) | Lower (structured clone or shared memory) |
| Scaling | Horizontal — add more worker processes | Vertical — limited by CPU cores on one machine |
| Failure isolation | Full — one worker crash doesn't affect others | Limited — uncaught exception kills the thread but not the main process |
| Best for | I/O-bound tasks (API calls, DB queries, email sending) | CPU-bound tasks within a job handler (image processing, PDF generation, data transformation) |

Use BullMQ workers for I/O-bound tasks (the default case). Use `worker_threads` only when a specific job handler needs to do heavy CPU work that would block the BullMQ worker's event loop.

### Spawning a worker thread from a job handler

```ts
// apps/workers/src/threads/image-processor.thread.ts
import { parentPort } from "node:worker_threads";

// Define typed message contracts
interface ProcessImageMessage {
  type: "process";
  imagePath: string;
  width: number;
  height: number;
}

interface ProcessImageResult {
  type: "result";
  outputPath: string;
  duration: number;
}

interface ProcessImageError {
  type: "error";
  message: string;
}

type WorkerResponse = ProcessImageResult | ProcessImageError;

parentPort?.on("message", async (msg: ProcessImageMessage) => {
  if (msg.type !== "process") return;

  try {
    const start = Date.now();
    // CPU-intensive work here
    const outputPath = await heavyImageResize(msg.imagePath, msg.width, msg.height);

    const result: ProcessImageResult = {
      type: "result",
      outputPath,
      duration: Date.now() - start,
    };
    parentPort?.postMessage(result);
  } catch (err) {
    const error: ProcessImageError = {
      type: "error",
      message: err instanceof Error ? err.message : String(err),
    };
    parentPort?.postMessage(error);
  }
});
```

### Typed message passing from the job handler

```ts
// apps/workers/src/workers/image.worker.ts
import { Worker as NodeWorker } from "node:worker_threads";
import { UnrecoverableError } from "bullmq";

async function processImageInThread(
  imagePath: string,
  width: number,
  height: number,
): Promise<string> {
  return new Promise((resolve, reject) => {
    const worker = new NodeWorker(
      new URL("../threads/image-processor.thread.js", import.meta.url),
    );

    const timeout = setTimeout(() => {
      worker.terminate();
      reject(new Error("Image processing timed out after 30s"));
    }, 30_000);

    worker.on("message", (msg: WorkerResponse) => {
      clearTimeout(timeout);
      worker.terminate();

      if (msg.type === "result") {
        resolve(msg.outputPath);
      } else {
        reject(new UnrecoverableError(msg.message));
      }
    });

    worker.on("error", (err) => {
      clearTimeout(timeout);
      worker.terminate();
      reject(err);
    });

    worker.postMessage({
      type: "process",
      imagePath,
      width,
      height,
    } satisfies ProcessImageMessage);
  });
}
```

### Limitations
- **No shared memory** for complex TypeScript objects — use `SharedArrayBuffer` for numeric data, structured clone for messages
- **Structured clone algorithm** for `postMessage` — similar to `JSON.parse(JSON.stringify())` but supports more types (Date, Map, Set, ArrayBuffer, etc.). Functions, WeakMaps, and circular references are not supported.
- **Thread pool limit** — the number of concurrent threads should not exceed `os.cpus().length`. Beyond that, context switching overhead dominates.
- **Worker threads are not free** — each thread has ~2MB memory overhead and spawns a separate V8 isolate. For one-off CPU work, consider whether the overhead is worth it vs just doing the work synchronously.

## Graceful shutdown (critical)

### Drain semantics of worker.close()

`worker.close()` (without force) does:
1. Marks the worker as "closing" — stops accepting new jobs
2. **Waits for all currently active (in-progress) jobs to complete or fail**
3. Closes Redis connections
4. Resolves the promise

`worker.close(true)` (force) skips step 2 — closes immediately without waiting.

**There is no built-in timeout.** If a job hangs, `worker.close()` will wait indefinitely. A force-exit timeout is mandatory.

Source: [BullMQ graceful shutdown docs](https://docs.bullmq.io/guide/workers/graceful-shutdown), confirmed via community reports ([GitHub issue #2237](https://github.com/taskforcesh/bullmq/issues/2237)).

### AbortSignal for in-job cancellation

The processor receives an `AbortSignal` as its third parameter. When `worker.close()` is called, the signal is aborted, allowing long-running jobs to clean up:

```ts
const worker = new Worker<JobPayload, void, JobName>(
  "notifications",
  async (job, _token, signal) => {
    signal?.addEventListener("abort", () => {
      // Release resources that would block shutdown
      httpClient.abortPending();
      dbClient.releaseConnection();
    });

    // Long-running work that checks the signal periodically...
  },
  { connection }
);
```

### Complete shutdown sequence

The shutdown ordering matters: workers first (stop accepting work), then queues (disconnect producers), then Redis (final cleanup).

```ts
// apps/workers/src/shutdown.ts
import type { Worker } from "bullmq";
import type { Redis } from "ioredis";
import type { Cron } from "croner";

const SHUTDOWN_TIMEOUT_MS = 25_000; // Must be < Docker stop_grace_period (30s default)

interface ShutdownDeps {
  workers: Worker[];
  queues: Queue[];
  cronJobs: Cron[];
  connection: Redis;
  logger: pino.Logger;
}

export function registerShutdownHandlers(deps: ShutdownDeps) {
  const { workers, queues, cronJobs, connection, logger } = deps;

  const shutdown = async (signal: string) => {
    logger.info(`Received ${signal}. Starting graceful shutdown...`);

    // Fallback: force exit if shutdown takes too long
    const forceExit = setTimeout(() => {
      logger.error("Forced shutdown after timeout — some jobs may not have completed");
      process.exit(1);
    }, SHUTDOWN_TIMEOUT_MS);
    forceExit.unref(); // Don't keep event loop alive for this timer

    try {
      // 1. Stop cron jobs first — no new scheduled work
      for (const job of cronJobs) {
        job.stop();
      }
      logger.info("Cron jobs stopped");

      // 2. Close workers — waits for active jobs to finish
      //    Order: close all workers in parallel
      logger.info(`Closing ${workers.length} workers...`);
      await Promise.all(workers.map((w) => w.close()));
      logger.info("All workers closed — active jobs drained");

      // 3. Close queues — disconnect queue producers
      for (const queue of queues) {
        await queue.close();
      }
      logger.info("All queues closed");

      // 4. Disconnect Redis — final cleanup
      await connection.quit();
      logger.info("Redis connection closed");

      clearTimeout(forceExit);
      logger.info("Graceful shutdown complete");
      process.exit(0);
    } catch (err) {
      logger.error("Error during graceful shutdown");
      logger.error(err);
      clearTimeout(forceExit);
      process.exit(1);
    }
  };

  process.on("SIGTERM", () => shutdown("SIGTERM"));
  process.on("SIGINT", () => shutdown("SIGINT"));
}
```

### Shutdown sequence rationale

1. **Cron jobs first:** Stop scheduled work immediately — no point starting new scheduled tasks during shutdown.
2. **Workers second:** `worker.close()` waits for active jobs to drain. This is the critical step — it ensures no in-progress work is lost. Workers are closed in parallel for speed.
3. **Queues third:** After workers drain, close queue connections. At this point no new jobs can be enqueued or dequeued.
4. **Redis last:** After everything else, disconnect the shared Redis connection.
5. **Force-exit timer:** If any step hangs, the 25s timer ensures the process eventually exits (before Docker's 30s SIGKILL).

### Docker alignment (consistent with D8)

```
Docker stop → sends SIGTERM to PID 1
  → SIGTERM handler runs shutdown()
    → cron jobs stopped
    → worker.close() drains active jobs (< 25s)
    → queue.close() disconnects producers
    → redis.quit() final cleanup
    → process.exit(0)
  → Docker waits stop_grace_period (default 30s)
  → If process hasn't exited, Docker sends SIGKILL

CRITICAL: SHUTDOWN_TIMEOUT_MS (25s) < Docker stop_grace_period (30s)
```

**Dockerfile CMD must use exec form:**

```dockerfile
CMD ["node", "dist/apps/workers/main.js"]
```

### What happens to unprocessed jobs?

Jobs waiting in the queue (not yet active) are NOT lost — they remain in Redis and are picked up by other workers when they come online. Only in-progress (active) jobs are affected if the worker crashes without draining. The force-exit timeout and stalled-job detection provide a safety net: if the worker crashes, the lock expires after `lockDuration` and BullMQ reassigns the job to another worker.

## Application bootstrap

### Entry point

```ts
// apps/workers/src/main.ts
import Redis from "ioredis";
import { Queue, Worker } from "bullmq";
import { loadEnv } from "./env.js";
import { registerShutdownHandlers } from "./shutdown.js";
import { createLogger } from "./logger.js";
import { registerEmailProcessor } from "./workers/email.worker.js";
import { registerNotificationProcessor } from "./workers/notification.worker.js";
import { registerDailyReportScheduler } from "./schedulers/daily-report.scheduler.js";
import { registerMetricsFlush } from "./cron/metrics-flush.cron.js";

async function main() {
  // 1. Validate environment — fail-fast before any connections
  const env = loadEnv();
  const logger = createLogger(env.LOG_LEVEL);

  logger.info("Starting workers app...");

  // 2. Create shared Redis connection
  const connection = new Redis({
    host: env.REDIS_HOST,
    port: env.REDIS_PORT,
    password: env.REDIS_PASSWORD || undefined,
    maxRetriesPerRequest: null, // Required for BullMQ Workers
    enableReadyCheck: false,
    keepAlive: 10_000,
    lazyConnect: false,
  });

  // 3. Instantiate queues (shared connection)
  const emailQueue = new Queue("email", { connection });
  const notificationQueue = new Queue("notifications", { connection });

  // 4. Create workers
  const emailWorker = new Worker("email", registerEmailProcessor(logger), {
    connection,
    concurrency: env.EMAIL_CONCURRENCY,
  });

  const notificationWorker = new Worker("notifications", registerNotificationProcessor(logger), {
    connection,
    concurrency: env.NOTIFICATION_CONCURRENCY,
  });

  // 5. Register repeatable job schedulers
  await registerDailyReportScheduler(emailQueue);

  // 6. Start cron-based system tasks
  const metricsJob = registerMetricsFlush(5);

  // 7. Register event listeners for observability
  emailWorker.on("completed", (job) => {
    logger.info({ jobId: job.id, jobName: job.name }, "Job completed");
  });
  emailWorker.on("failed", (job, err) => {
    logger.error({ jobId: job?.id, jobName: job?.name, err: err.message }, "Job failed");
  });

  notificationWorker.on("completed", (job) => {
    logger.info({ jobId: job.id, jobName: job.name }, "Job completed");
  });
  notificationWorker.on("failed", (job, err) => {
    logger.error({ jobId: job?.id, jobName: job?.name, err: err.message }, "Job failed");
  });

  // 8. Register graceful shutdown — must be last before idling
  registerShutdownHandlers({
    workers: [emailWorker, notificationWorker],
    queues: [emailQueue, notificationQueue],
    cronJobs: [metricsJob],
    connection,
    logger,
  });

  logger.info("Workers app started — waiting for jobs...");
}

main().catch((err) => {
  console.error("Failed to start workers app:", err);
  process.exit(1);
});
```

### Environment validation with Zod (same pattern as D8)

```ts
// apps/workers/src/env.ts
import { z } from "zod/v4";

const envSchema = z.object({
  NODE_ENV: z.enum(["development", "production", "test"]),
  REDIS_HOST: z.string().default("localhost"),
  REDIS_PORT: z.coerce.number().int().positive().default(6379),
  REDIS_PASSWORD: z.string().optional(),
  LOG_LEVEL: z.enum(["trace", "debug", "info", "warn", "error", "fatal"]).default("info"),
  EMAIL_CONCURRENCY: z.coerce.number().int().positive().default(5),
  NOTIFICATION_CONCURRENCY: z.coerce.number().int().positive().default(10),
});

export type WorkerEnv = z.output<typeof envSchema>;

export function loadEnv(): WorkerEnv {
  const result = envSchema.safeParse(process.env);
  if (!result.success) {
    console.error("Invalid environment variables:", result.error.toString());
    process.exit(1);
  }
  return result.data;
}
```

### Structured logging with pino

Use the same pino logger as `/apps/backend` (D8) for consistent log format across apps:

```ts
// apps/workers/src/logger.ts
import pino from "pino";

export function createLogger(level: string) {
  return pino({
    level,
    ...(process.env.NODE_ENV === "production"
      ? {} // JSON output for log aggregators
      : {
          transport: {
            target: "pino-pretty",
            options: { colorize: true },
          },
        }),
  });
}
```

### Health monitoring

The workers app has no HTTP server, so health monitoring is via worker events and Redis connectivity:

```ts
// Log worker health periodically
const healthInterval = setInterval(() => {
  logger.info({
    emailWorkerActive: emailWorker.isRunning(),
    notificationWorkerActive: notificationWorker.isRunning(),
    emailWorkerPaused: emailWorker.isPaused(),
  }, "Worker health check");
}, 30_000);

// In graceful shutdown:
clearInterval(healthInterval);
```

For external monitoring (Kubernetes liveness/readiness), either:
- Run a minimal HTTP health server on a sidecar port (simplest for K8s)
- Use Redis-based health checks (an external monitor pings Redis for worker heartbeat keys)

A lightweight health HTTP server option:

```ts
import http from "node:http";

// Minimal HTTP server just for K8s probes
const healthServer = http.createServer((req, res) => {
  if (req.url === "/health/live") {
    res.writeHead(200, { "Content-Type": "application/json" });
    res.end(JSON.stringify({ status: "ok" }));
  } else if (req.url === "/health/ready") {
    const ready = emailWorker.isRunning() && notificationWorker.isRunning();
    res.writeHead(ready ? 200 : 503, { "Content-Type": "application/json" });
    res.end(JSON.stringify({ status: ready ? "ok" : "degraded" }));
  } else {
    res.writeHead(404);
    res.end();
  }
});

healthServer.listen(env.HEALTH_PORT ?? 3001);
```

## Antipatterns and common mistakes

1. **Referencing QueueScheduler.** Removed in BullMQ v2.0 (September 2022). The Worker class handles stalled jobs, delayed jobs, and retries internally. Any import of `QueueScheduler` is targeting a version that hasn't existed for 4 years.

2. **No Zod validation at consumption.** Trusting that the backend always sends valid data. Workers are the defensive boundary — validate with `@shared/jobs` schemas before processing. Invalid payloads should throw `UnrecoverableError`.

3. **No graceful shutdown.** Without SIGTERM handling, Docker sends SIGKILL after `stop_grace_period`. Active jobs are killed mid-processing. On restart, stalled-job detection reassigns them — but work is duplicated. Always drain workers before exiting.

4. **Defining payload types inline.** Don't define `interface EmailPayload { ... }` in the worker file. Import from `@shared/jobs`. Duplicating types breaks the three-way contract.

5. **Discriminating on `job.name` instead of `job.data.jobName`.** TypeScript does not narrow `job.data` based on `job.name`. Use a discriminant field inside `job.data`. D5 already uses `jobName` for this purpose.

6. **Shared mutable state between concurrent worker instances.** With `concurrency > 1`, multiple jobs run in the same Node.js process. Module-level `let` variables are shared across all concurrent jobs. Use job-local state or explicitly scope data to each handler invocation.

7. **No `maxRetriesPerRequest: null` on Redis connection.** BullMQ Workers use blocking Redis commands. Without this setting, ioredis throws `ReplyError: maxRetriesPerRequest` on the first blocking call. This is a hard runtime failure.

8. **No `removeOnComplete` / `removeOnFail` configuration.** Default behavior keeps all completed and failed jobs in Redis indefinitely. In production, this exhausts Redis memory. Always configure retention limits.

9. **Missing force-exit timeout in shutdown.** If a job hangs, `worker.close()` waits forever. Always pair with a `setTimeout` + `process.exit(1)` fallback, with timer duration less than Docker's `stop_grace_period`.

10. **Using the legacy `queue.add()` with `repeat` option for repeatable jobs.** The modern API is `queue.upsertJobScheduler()` (v5.16.0+). The legacy API can create duplicate schedules.

11. **No `signal?.addEventListener("abort")` in long-running processors.** Without listening for the abort signal, the processor cannot clean up resources (DB connections, HTTP requests) during shutdown, causing the drain to hang.

12. **Creating a new Redis connection per Queue/Worker.** Creates unnecessary connections. BullMQ internally duplicates the connection for blocking commands. Share one `Redis` instance across all Queue, Worker, and QueueEvents instances.

## Audit-flagged gaps — resolution

### 1. BullMQ current major version — RESOLVED

Current stable is **v5.77.3** (May 25, 2026). There is no v6. The v5.x line has been the current major since at least early 2024 with 77+ minor releases. No breaking generics changes since the QueueScheduler removal in v2.0.

**Source:** [npm: bullmq](https://www.npmjs.com/package/bullmq), [GitHub releases](https://github.com/taskforcesh/bullmq/releases)

### 2. QueueScheduler deprecation/removal — RESOLVED

**Fully removed in BullMQ v2.0 (September 2022).** The Worker class absorbed all QueueScheduler responsibilities (stalled job detection, delayed job processing, retry management). This is a clean removal — no transitional API, no deprecation period in v5. Any reference to `QueueScheduler` in documentation or code is targeting BullMQ v1.x and is incorrect for all current versions.

Note: `JobScheduler` (introduced v5.16.0) is a separate class for managing repeatable job schedules. It is NOT a replacement for `QueueScheduler` — they serve completely different purposes despite the similar name.

**Source:** [BullMQ PR #1413](https://github.com/taskforcesh/bullmq/pull/1413), [commit 1f66e5a](https://github.com/taskforcesh/bullmq/commit/1f66e5a6c891d52e0671e58a685dbca511e45e7e)

### 3. Sandboxed processor TypeScript support — RESOLVED

Sandboxed processors can be typed using the `SandboxedJob` type (imported from `bullmq`, fully typed as of v5.73.1). For `.js` and `.mjs` processors: import `SandboxedJob` and use its typed properties. For `.ts` processors: compile to `.js` first (tsc/esbuild), then point the Worker to the compiled file. Native `.mts` support is in progress (PR #3984, targeting Node.js 22+ native TypeScript).

**Recommendation for this project:** For most I/O-bound work, use regular typed Workers with `concurrency` — sandboxed processors add fork overhead. If CPU-heavy work is needed, use `worker_threads` directly from a job handler (see Worker threads section). If process isolation is needed (untrusted code, native addons), compile the processor to `.js` and use the sandboxed processor pattern.

**Source:** [BullMQ sandboxed processors docs](https://docs.bullmq.io/guide/workers/sandboxed-processors), [PR #3984](https://github.com/taskforcesh/bullmq/pull/3984)

### 4. Graceful shutdown drain semantics — RESOLVED

`worker.close()` (without force) **waits for all active jobs to complete before resolving.** It stops accepting new jobs but drains in-progress ones. There is NO built-in timeout — if a job hangs, `close()` waits indefinitely. This is confirmed in the official docs, the API reference, and community bug reports (GitHub issue #2237).

The recommended pattern is: `worker.close()` + `setTimeout(forceExit, 25_000)` as fallback + `process.on("SIGTERM/SIGINT")` handlers. The AbortSignal (third processor parameter) allows in-job cleanup when `close()` is called.

Shutdown sequence: cron jobs → close all workers in parallel → close all queues → quit Redis → exit. This is operationally consistent with D8's shutdown pattern (Fastify → DB → Redis → exit).

**Source:** [BullMQ graceful shutdown guide](https://docs.bullmq.io/guide/workers/graceful-shutdown), [GitHub issue #2237](https://github.com/taskforcesh/bullmq/issues/2237)

### 5. Discriminated union + Worker typing — RESOLVED

BullMQ's generics do NOT distribute `NameType` across the `DataType` union. `switch (job.name)` does NOT narrow `job.data`. The BullMQ maintainers recommend discriminating on a field inside `job.data` (e.g., `job.data.jobName`). D5 already uses this pattern with the `jobName` literal field in each payload interface.

The pattern: define a discriminated union payload where each member has a unique literal field → pass the union as `DataType` to `Worker` → switch on the discriminant field in `job.data` → each case branch gets narrowed data type.

**Source:** [BullMQ GitHub issue #1156](https://github.com/taskforcesh/bullmq/issues/1156), [discussion #3509](https://github.com/taskforcesh/bullmq/discussions/3509)

### 6. node-cron vs croner vs BullMQ repeatables — RESOLVED

**node-cron** (v4.2.1, Jul 2025): Stable, ~4.2M weekly downloads, 33 open issues, last GitHub push Feb 2026. Simple 5-field cron. Good for basic scheduled tasks.

**croner** (v10.0.1, Feb 2026): More actively maintained, ~2M weekly downloads, 6 open issues, last push May 2026. Full cron features (seconds, L, W, #, years), overrun protection, built-in error handling, cross-platform (Deno, Bun, browser). Technically superior.

**BullMQ repeatables:** Redis-backed, distributed-safe, integrated with job lifecycle (retries, events, monitoring). Best for work-pipeline scheduled tasks.

**Recommendation:** BullMQ repeatable jobs (`upsertJobScheduler`) for tasks that are part of the job processing pipeline. croner for system-level tasks (metrics flush, cache warming). This split avoids coupling system tasks to Redis availability while keeping work-pipeline tasks in the job system where they get retries and monitoring.

**Source:** [croner GitHub](https://github.com/Hexagon/croner), [node-cron GitHub](https://github.com/kelektiv/node-cron), [BullMQ repeatable jobs docs](https://docs.bullmq.io/guide/jobs/repeatable)

## Reference snippets

### apps/workers/src/queues/email.queue.ts

```ts
import { Queue } from "bullmq";
import type { JobPayload, JobName } from "@shared/jobs";
import type { Redis } from "ioredis";

export function createEmailQueue(connection: Redis) {
  return new Queue<JobPayload, void, JobName>("email", {
    connection,
    defaultJobOptions: {
      attempts: 3,
      backoff: { type: "exponential", delay: 1000 },
      removeOnComplete: { age: 3600, count: 1000 },
      removeOnFail: { age: 86400, count: 5000 },
    },
  });
}
```

### apps/workers/src/workers/email.worker.ts (discriminated union handler)

```ts
import { Worker, UnrecoverableError } from "bullmq";
import { EmailJobPayloadSchema } from "@shared/jobs";
import type { JobPayload, JobName } from "@shared/jobs";
import type { Redis } from "ioredis";
import type { Logger } from "pino";
import { validatePayload } from "./base.worker.js";

export function createEmailWorker(connection: Redis, logger: Logger) {
  const worker = new Worker<JobPayload, void, JobName>(
    "email",
    async (job, _token, signal) => {
      signal?.addEventListener("abort", () => {
        logger.info({ jobId: job.id }, "Email job received abort signal");
      });

      switch (job.data.jobName) {
        case "send-email": {
          const data = validatePayload(EmailJobPayloadSchema, job.data, job.id);
          logger.info({ jobId: job.id, to: data.to }, "Sending email");

          await job.updateProgress(25);
          await emailService.send(data);
          await job.updateProgress(100);

          return;
        }

        default:
          throw new UnrecoverableError(
            `Unknown job type: ${job.data.jobName}`
          );
      }
    },
    { connection, concurrency: 5 },
  );

  return worker;
}
```

### apps/workers/src/workers/base.worker.ts (Zod validation wrapper)

```ts
import type { ZodSchema } from "zod/v4";
import { UnrecoverableError } from "bullmq";

export function validatePayload<T>(
  schema: ZodSchema<T>,
  data: unknown,
  jobId?: string,
): T {
  const result = schema.safeParse(data);
  if (!result.success) {
    throw new UnrecoverableError(
      `Invalid payload for job ${jobId ?? "unknown"}: ${result.error.message}`,
    );
  }
  return result.data;
}
```

### Dead-letter queue handler

```ts
import { Queue, UnrecoverableError } from "bullmq";
import type { JobPayload, JobName } from "@shared/jobs";

export async function handleDeadLetter(
  deadLetterQueue: Queue,
  job: { name: string; id?: string; data: unknown; attemptsMade: number },
  err: UnrecoverableError,
) {
  await deadLetterQueue.add(job.name, {
    originalJobId: job.id,
    originalData: job.data,
    error: err.message,
    failedAt: new Date().toISOString(),
    attemptsMade: job.attemptsMade,
  });
  throw err;
}
```

### Graceful shutdown module (full sequence)

```ts
import type { Worker, Queue } from "bullmq";
import type { Redis } from "ioredis";
import type { Cron } from "croner";
import type { Logger } from "pino";

const SHUTDOWN_TIMEOUT_MS = 25_000;

interface ShutdownDeps {
  workers: Worker[];
  queues: Queue[];
  cronJobs: Cron[];
  connection: Redis;
  logger: Logger;
}

export function registerShutdownHandlers(deps: ShutdownDeps) {
  const { workers, queues, cronJobs, connection, logger } = deps;

  const shutdown = async (signal: string) => {
    logger.info(`Received ${signal}. Starting graceful shutdown...`);

    const forceExit = setTimeout(() => {
      logger.error("Forced shutdown after timeout");
      process.exit(1);
    }, SHUTDOWN_TIMEOUT_MS);
    forceExit.unref();

    try {
      for (const job of cronJobs) job.stop();
      logger.info("Cron jobs stopped");

      logger.info(`Closing ${workers.length} workers...`);
      await Promise.all(workers.map((w) => w.close()));
      logger.info("All workers closed");

      for (const queue of queues) await queue.close();
      logger.info("All queues closed");

      await connection.quit();
      logger.info("Redis connection closed");

      clearTimeout(forceExit);
      logger.info("Graceful shutdown complete");
      process.exit(0);
    } catch (err) {
      logger.error("Error during graceful shutdown");
      logger.error(err);
      clearTimeout(forceExit);
      process.exit(1);
    }
  };

  process.on("SIGTERM", () => shutdown("SIGTERM"));
  process.on("SIGINT", () => shutdown("SIGINT"));
}
```

### Scheduled task (croner — system tasks)

```ts
import { Cron } from "croner";
import type { Logger } from "pino";

export function registerMetricsFlush(logger: Logger, intervalMinutes = 5) {
  return new Cron(
    `*/${intervalMinutes} * * * *`,
    {
      name: "metrics-flush",
      protect: true,
      catch: (err) => {
        logger.error({ err: err.message }, "Metrics flush failed");
      },
    },
    async () => {
      await metricsService.flush();
    },
  );
}
```

### Scheduled task (BullMQ repeatable — work pipeline)

```ts
import { Queue } from "bullmq";
import type { JobPayload, JobName } from "@shared/jobs";

export async function registerDailyReportScheduler(
  queue: Queue<JobPayload, void, JobName>,
) {
  await queue.upsertJobScheduler(
    "daily-report-scheduler",
    { pattern: "0 7 * * *" },
    {
      name: "send-email",
      data: {
        version: 1,
        jobName: "send-email",
        to: "reports@example.com",
        subject: "Daily Report",
        body: "Report placeholder",
      },
      opts: {
        attempts: 3,
        backoff: { type: "fixed", delay: 60_000 },
      },
    },
  );
}
```

### apps/workers/src/env.ts (Zod-validated environment)

```ts
import { z } from "zod/v4";

const envSchema = z.object({
  NODE_ENV: z.enum(["development", "production", "test"]),
  REDIS_HOST: z.string().default("localhost"),
  REDIS_PORT: z.coerce.number().int().positive().default(6379),
  REDIS_PASSWORD: z.string().optional(),
  LOG_LEVEL: z
    .enum(["trace", "debug", "info", "warn", "error", "fatal"])
    .default("info"),
  EMAIL_CONCURRENCY: z.coerce.number().int().positive().default(5),
  NOTIFICATION_CONCURRENCY: z.coerce.number().int().positive().default(10),
  HEALTH_PORT: z.coerce.number().int().positive().default(3001),
});

export type WorkerEnv = z.output<typeof envSchema>;

export function loadEnv(): WorkerEnv {
  const result = envSchema.safeParse(process.env);
  if (!result.success) {
    console.error("Invalid environment variables:", result.error.toString());
    process.exit(1);
  }
  return result.data;
}
```

### apps/workers/src/main.ts (bootstrap entry point)

```ts
import Redis from "ioredis";
import { loadEnv } from "./env.js";
import { createLogger } from "./logger.js";
import { createEmailQueue } from "./queues/email.queue.js";
import { createNotificationQueue } from "./queues/notification.queue.js";
import { createEmailWorker } from "./workers/email.worker.js";
import { createNotificationWorker } from "./workers/notification.worker.js";
import { registerDailyReportScheduler } from "./schedulers/daily-report.scheduler.js";
import { registerMetricsFlush } from "./cron/metrics-flush.cron.js";
import { registerShutdownHandlers } from "./shutdown.js";

async function main() {
  const env = loadEnv();
  const logger = createLogger(env.LOG_LEVEL);

  logger.info("Starting workers app...");

  const connection = new Redis({
    host: env.REDIS_HOST,
    port: env.REDIS_PORT,
    password: env.REDIS_PASSWORD || undefined,
    maxRetriesPerRequest: null,
    enableReadyCheck: false,
    keepAlive: 10_000,
  });

  const emailQueue = createEmailQueue(connection);
  const notificationQueue = createNotificationQueue(connection);

  const emailWorker = createEmailWorker(connection, logger);
  const notificationWorker = createNotificationWorker(connection, logger);

  await registerDailyReportScheduler(emailQueue);

  const metricsJob = registerMetricsFlush(logger, 5);

  emailWorker.on("completed", (job) => {
    logger.info({ jobId: job.id, jobName: job.name }, "Job completed");
  });
  emailWorker.on("failed", (job, err) => {
    logger.error({ jobId: job?.id, jobName: job?.name, err: err.message }, "Job failed");
  });

  notificationWorker.on("completed", (job) => {
    logger.info({ jobId: job.id, jobName: job.name }, "Job completed");
  });
  notificationWorker.on("failed", (job, err) => {
    logger.error({ jobId: job?.id, jobName: job?.name, err: err.message }, "Job failed");
  });

  registerShutdownHandlers({
    workers: [emailWorker, notificationWorker],
    queues: [emailQueue, notificationQueue],
    cronJobs: [metricsJob],
    connection,
    logger,
  });

  logger.info("Workers app started — waiting for jobs...");
}

main().catch((err) => {
  console.error("Failed to start workers app:", err);
  process.exit(1);
});
```

## Sources used

- [BullMQ GitHub releases](https://github.com/taskforcesh/bullmq/releases) — official
- [BullMQ v2.0 breaking changes — QueueScheduler removal PR #1413](https://github.com/taskforcesh/bullmq/pull/1413) — official
- [BullMQ graceful shutdown guide](https://docs.bullmq.io/guide/workers/graceful-shutdown) — official
- [BullMQ v5 Worker API](https://api.docs.bullmq.io/classes/v5.Worker.html) — official
- [BullMQ v5 Queue API](https://api.docs.bullmq.io/classes/v5.Queue.html) — official
- [BullMQ v5 Job API](https://api.docs.bullmq.io/classes/v5.Job.html) — official
- [BullMQ v5 UnrecoverableError API](https://api.docs.bullmq.io/classes/v5.UnrecoverableError.html) — official
- [BullMQ v5 BaseJobOptions API](https://api.docs.bullmq.io/interfaces/v5.BaseJobOptions.html) — official
- [BullMQ repeatable jobs — upsertJobScheduler](https://docs.bullmq.io/guide/jobs/repeatable) — official
- [BullMQ sandboxed processors](https://docs.bullmq.io/guide/workers/sandboxed-processors) — official
- [BullMQ retrying failing jobs](https://docs.bullmq.io/guide/retrying-failing-jobs) — official
- [BullMQ auto-removal of jobs](https://docs.bullmq.io/guide/queues/auto-removal-of-jobs) — official
- [BullMQ GitHub issue #1156 — discriminated union pattern](https://github.com/taskforcesh/bullmq/issues/1156) — official
- [BullMQ GitHub discussion #3509 — typed queue pattern](https://github.com/taskforcesh/bullmq/discussions/3509) — community
- [BullMQ GitHub issue #2237 — worker.close() hang](https://github.com/taskforcesh/bullmq/issues/2237) — official
- [BullMQ PR #3984 — native TypeScript sandbox (.mts/.cts)](https://github.com/taskforcesh/bullmq/pull/3984) — official
- [BullMQ PR #3476 — ESM sandbox support (.mjs)](https://github.com/taskforcesh/bullmq/pull/3476) — official
- [npm: bullmq](https://www.npmjs.com/package/bullmq) — official
- [croner — GitHub](https://github.com/Hexagon/croner) — official
- [node-cron — GitHub](https://github.com/kelektiv/node-cron) — official
- [Node.js worker_threads — v26.1.0 docs](https://nodejs.org/dist/latest/docs/api/worker_threads.html) — official
