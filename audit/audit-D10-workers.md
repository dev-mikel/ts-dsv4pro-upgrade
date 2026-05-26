# Audit D10 — Workers App (BullMQ + node-cron + Worker Threads)

## 1. Knowledge Level: Solid (BullMQ API stable, integration gaps)

I know BullMQ: `Queue`, `Worker`, `QueueEvents`, `FlowProducer` (for job flows/chains), job options (`delay`, `priority`, `backoff`, `attempts`, `removeOnComplete`, `removeOnFail`), `addBulk`, sandboxed processors, `Job` type, job lifecycle events, Redis connection configuration, `QueueScheduler` (or `QueueEvents` for newer versions). I know `node-cron` for scheduled tasks, and Node.js `worker_threads` conceptually (though BullMQ handles worker distribution via Redis, so worker_threads usage in this architecture is likely for CPU-heavy work within a job handler, not for job distribution).

## 2. Versions Covered

- **BullMQ**: Confident through ~4.x or ~5.x (2023–early 2024). The API is relatively stable, but version-specific gaps exist.
- **ioredis** (BullMQ's Redis client): Confident through v5.x.
- **node-cron**: Stable simple API, low risk of drift.
- **worker_threads**: Stable Node.js API.

## 3. Identified Gaps

- **BullMQ 5.x/6.x**: I don't know the current major version. If BullMQ shipped a new major, possible changes include: Redis Cluster support improvements, telemetry/OpenTelemetry integration, new job option fields, `QueueScheduler` deprecation status, and `Worker` constructor options.
- **BullMQ + TypeScript generics**: The `Queue<PayloadType, ReturnType>` and `Worker<PayloadType, ReturnType>` generics pattern. I know the pattern but don't know if it changed (e.g., additional type parameters for job name discrimination).
- **Job concurrency patterns**: `Worker` concurrency setting, rate limiting, group-level concurrency — I know these exist but don't know the current exact API.
- **Sandboxed processors in TypeScript**: Using `path.resolve` to a processor file for forked execution. I don't know if there's a newer recommended pattern (e.g., using `tsx` for TS processors without pre-compilation).
- **Queue drain / graceful shutdown**: The BullMQ pattern for draining workers gracefully (stop accepting new jobs, finish in-progress ones) on SIGTERM. I know the concept but not the exact current `worker.close()` / `queue.close()` semantics or timeouts.
- **`node-cron` alternatives in a worker app**: I don't know if `@nestjs/schedule` or `cron` (different package) or `node-schedule` is preferred in 2025.
- **Job event listening**: `QueueEvents` vs direct worker events — I know both exist but don't know which pattern is currently recommended for observability.
- **Redis connection sharing in a monorepo**: If both `@apps/backend` and `@apps/workers` connect to Redis (backend for enqueuing, workers for consuming), should they share a Redis client via a shared package? I don't know the current best practice.

## 4. Hallucination Risk

Highest risk areas:
- Using BullMQ v3/v4 API for a project on v5/v6.
- Incorrect `Worker` constructor options that were renamed or re-typed.
- Recommending `QueueScheduler` when it was removed or fully deprecated.
- Wrong graceful shutdown sequence — BullMQ close semantics can be version-specific.
- Suggesting sandboxed processor patterns that don't work with the project's TS build setup.

## 5. Confidence: 6/10

BullMQ's core API is stable and I know the patterns. Version drift and integration with shared types/Turborepo/graceful shutdown are the primary gaps — mostly at the edges, not the core.

## Intersection Notes

- **D5 (shared types)**: Job payload types are the critical contract. The backend (`Queue.add('jobName', payload)`) and the worker (`Worker.process('jobName', handler)`) must agree on types. These types should come from `@shared-types/jobs` with Zod schemas for runtime validation.
- **D4 (Zod)**: Workers should validate incoming job payloads with Zod. Even though the backend also validated, workers are the defensive boundary — data could be stale or corrupted in Redis.
- **D8 (backend)**: The backend is the primary enqueuer. Job type definitions shared via `@shared-types` ensure the `Queue.add` call type-checks against the Worker's expected payload.
- **D3 (Turborepo)**: The workers app has its own `turbo.json` task. Worker-specific build output for the 2.x schema is unknown.
- **S2 (Docker)**: The worker Dockerfile needs the correct build output. Workers often run long-lived processes — healthcheck strategy differs from HTTP services.
- **S1 (Prisma)**: Workers may access the database directly (bypassing the backend API) for performance. This creates a coupling risk that needs explicit architectural rules.

## Recommended Phase 2 Search Queries

1. `BullMQ 6.x changelog migration guide 2025`
2. `BullMQ TypeScript generics job type discrimination 2025`
3. `BullMQ QueueScheduler deprecation status 2025`
4. `BullMQ graceful shutdown worker drain pattern 2025`
5. `BullMQ sandboxed processor TypeScript tsx 2025`
6. `BullMQ Redis connection sharing monorepo pattern`
7. `node-cron alternatives 2025 cron package comparison`
