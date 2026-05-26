# Skill: TypeScript — S2: Per-App Containerization

## Summary

Covers per-app containerization for the monorepo: multi-stage Dockerfiles for all four deployment units (frontend, backend, workers, migration job), a local-development `docker-compose.yml`, environment variable management, and image optimization. Uses `turbo prune` from D3 to produce minimal build contexts. Targets **Node.js 24.x** (Krypton, Active LTS), **pnpm 11.3.0**, and **Alpine** base images (the stack is pure JavaScript — no native modules, no musl libc issues).

**Data layer (S1):** Drizzle ORM has no binary. No `drizzle-kit generate` or `drizzle-kit migrate` inside any app Dockerfile. Migrations run as a separate one-shot container before backend and workers start.

**Knowledge cutoff:** 2026-05-25. **Dependencies:** D3 (Turborepo 2.x, turbo prune), D8 (Fastify, SIGTERM shutdown, /health), D9 (Next.js, `output: 'standalone'`), D10 (BullMQ workers, SIGTERM drain, no HTTP server), S1 (Drizzle, no ORM binary, separate migration step). **This is the final skill in the series.**

## Version landscape

| Component | Version | Notes |
|---|---|---|
| Node.js | **24.x (Krypton)** | Active LTS. 22.x is Maintenance LTS. 20.x EOL Apr 2026. |
| pnpm | **11.3.0** | Consistent with D3. Installed via corepack. |
| Turborepo | **2.9.14** | Consistent with D3. `turbo prune --docker` produces `out/json/` + `out/full/`. |
| Next.js | **16.2.6** | `output: 'standalone'` in next.config.ts (D9). |
| Docker | **BuildKit (stable)** | `docker compose` plugin v2, not `docker-compose` Python v1. |
| drizzle-kit | **0.31.10** | Consistent with S1. Migrate runs as one-shot container, not in app Dockerfiles. |

### Alpine vs Debian slim

This stack is pure JavaScript across all deployment units:

| Component | Native modules? |
|---|---|
| `postgres.js` (database driver) | No — pure JS |
| Drizzle ORM | No — pure JS |
| BullMQ + ioredis | No — pure JS |
| Next.js standalone | No — pure JS |
| Fastify + pino | No — pure JS |
| Zod | No — pure JS |

**Recommendation: `node:24-alpine`.** With zero native modules, Alpine's musl libc is not a compatibility risk. The image size advantage (~50 MB vs ~180 MB for slim) matters across four images. If a future dependency introduces native modules (e.g., `sharp` for image processing), switch to `node:24-slim` for that specific image only.

### pnpm in Docker (Node.js 24)

Node.js 24 still bundles Corepack, so the classic approach works:

```dockerfile
FROM node:24-alpine AS base
ENV PNPM_HOME="/pnpm"
ENV PATH="$PNPM_HOME/bin:$PATH"
RUN corepack enable && corepack prepare pnpm@11 --activate
```

`PNPM_HOME` and `PATH` are set so pnpm's global store is at a known location. The `corepack prepare pnpm@11 --activate` pins the exact major version.

**Note:** Node.js 25 removed Corepack (October 2025). If upgrading beyond 24.x, install Corepack via `npm install -g corepack@latest` first. For Node 24.x, the bundled Corepack is sufficient.

### docker compose CLI

Use `docker compose` (plugin v2), **not** `docker-compose` (Python v1, deprecated since 2023). The Compose file format is the same; the CLI is different.

## Multi-stage Dockerfile — shared pattern

All app Dockerfiles follow the same four-stage structure. The only variation is in the `builder` and `runner` stages (different build commands, different artifacts).

```
Stage: base     →  Node.js Alpine + pnpm (shared, cached)
Stage: pruner   →  turbo prune --docker <app>  →  out/json/ + out/full/
Stage: installer →  COPY out/json/ → pnpm install --frozen-lockfile
Stage: builder  →  COPY out/full/ → pnpm turbo run build --filter=<app>...
Stage: runner   →  COPY only production artifacts → non-root user → CMD
```

### Why json/ and full/ are separate stages

`turbo prune --docker` produces two directories:
- `out/json/` — only `package.json` files + pruned lockfile. Changes rarely (only when deps change).
- `out/full/` — full source code of needed packages. Changes on every source edit.

By copying `json/` first, installing dependencies, THEN copying `full/` and building, the `pnpm install` layer is cached as long as `package.json`/lockfile don't change. Without this split, `pnpm install` re-runs on every source change.

### @packages/db inclusion

`@packages/db` is a workspace dependency of `@app/backend` and `@app/workers`. `turbo prune` follows the dependency graph — if an app depends on `@packages/db`, the package's full source is included in `out/full/` and its `package.json` is in `out/json/`. No explicit inclusion needed.

### Layer cache strategy

```
1. COPY package.json / lockfile  →  pnpm install     [CACHED unless deps change]
2. COPY source                    →  build            [RUNS on source changes]
3. COPY build artifacts only      →  runner           [Minimal final layer]
```

This ordering ensures the expensive `pnpm install` is cached across builds that only touch source code.

### Non-root user

All runner stages use `USER node`. The `node` user exists by default in official Node.js images (uid 1000). For Alpine, the user and group are both named `node`.

### .dockerignore

```
node_modules
.turbo
.git
**/.next/cache
**/coverage
**/dist
**/*.md
.env
.env.*
!.env.example
```

The build context must include the entire monorepo (so `turbo prune` can see all packages), but `node_modules`, caches, and build artifacts are excluded.

## Dockerfile — /apps/backend (Fastify)

```dockerfile
# syntax=docker/dockerfile:1

# ---- Stage 1: Base ----
FROM node:24-alpine AS base
ENV PNPM_HOME="/pnpm"
ENV PATH="$PNPM_HOME/bin:$PATH"
RUN corepack enable && corepack prepare pnpm@11 --activate

# ---- Stage 2: Prune ----
FROM base AS pruner
WORKDIR /app
COPY . .
RUN pnpm dlx turbo prune @app/backend --docker

# ---- Stage 3: Install dependencies ----
FROM base AS installer
WORKDIR /app
COPY --from=pruner /app/out/json/ .
RUN pnpm install --frozen-lockfile

# ---- Stage 4: Build ----
FROM installer AS builder
COPY --from=pruner /app/out/full/ .
RUN pnpm turbo run build --filter=@app/backend...

# ---- Stage 5: Runner ----
FROM node:24-alpine AS runner
WORKDIR /app

# Create non-root user early
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nodejs

# Copy production dependencies from installer
COPY --from=installer --chown=nodejs:nodejs /app/node_modules ./node_modules

# Copy compiled output
COPY --from=builder --chown=nodejs:nodejs /app/apps/backend/dist ./dist
COPY --from=builder --chown=nodejs:nodejs /app/apps/backend/package.json .

# Copy @packages/db compiled output (needed for migrations at runtime)
COPY --from=builder --chown=nodejs:nodejs /app/packages/db/dist ./packages/db/dist
COPY --from=builder --chown=nodejs:nodejs /app/packages/db/package.json ./packages/db/package.json

# Copy pruned workspace config (pnpm needs this to resolve workspace:*)
COPY --from=pruner /app/out/json/pnpm-workspace.yaml ./

USER nodejs
EXPOSE 3000
ENV NODE_ENV=production
STOPSIGNAL SIGTERM

HEALTHCHECK --interval=15s --timeout=5s --start-period=10s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health/live || exit 1

CMD ["node", "dist/main.js"]
```

**Key points:**
- `EXPOSE 3000` — matches the `PORT` default in D8's env validation.
- `STOPSIGNAL SIGTERM` — aligns with D8's graceful shutdown handler. Docker's default is SIGTERM, but explicit declaration ensures consistency across images.
- `HEALTHCHECK` — hits `/health/live` (D8 defines this route). `start-period=10s` gives Fastify time to boot. Uses `wget` (available in Alpine) instead of `curl` (extra package).
- `--chown=nodejs:nodejs` on all COPY commands ensures the non-root user owns all files.
- `@packages/db/dist` is copied because `@packages/db` is compiled TypeScript — the backend imports its compiled JS at runtime via `workspace:*`.
- `pnpm-workspace.yaml` is needed so the pruned `node_modules` symlinks resolve correctly.

## Dockerfile — /apps/workers (BullMQ)

```dockerfile
# syntax=docker/dockerfile:1

# ---- Stage 1: Base ----
FROM node:24-alpine AS base
ENV PNPM_HOME="/pnpm"
ENV PATH="$PNPM_HOME/bin:$PATH"
RUN corepack enable && corepack prepare pnpm@11 --activate

# ---- Stage 2: Prune ----
FROM base AS pruner
WORKDIR /app
COPY . .
RUN pnpm dlx turbo prune @app/workers --docker

# ---- Stage 3: Install dependencies ----
FROM base AS installer
WORKDIR /app
COPY --from=pruner /app/out/json/ .
RUN pnpm install --frozen-lockfile

# ---- Stage 4: Build ----
FROM installer AS builder
COPY --from=pruner /app/out/full/ .
RUN pnpm turbo run build --filter=@app/workers...

# ---- Stage 5: Runner ----
FROM node:24-alpine AS runner
WORKDIR /app

RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nodejs

COPY --from=installer --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nodejs:nodejs /app/apps/workers/dist ./dist
COPY --from=builder --chown=nodejs:nodejs /app/apps/workers/package.json .
COPY --from=builder --chown=nodejs:nodejs /app/packages/db/dist ./packages/db/dist
COPY --from=builder --chown=nodejs:nodejs /app/packages/db/package.json ./packages/db/package.json
COPY --from=pruner /app/out/json/pnpm-workspace.yaml ./

USER nodejs
# No EXPOSE — workers have no HTTP server for business traffic
ENV NODE_ENV=production
STOPSIGNAL SIGTERM

# Minimal HTTP server for health checks only (see D10 health monitoring pattern)
# The worker's main.ts starts a lightweight http server on HEALTH_PORT for probes.
# For docker-compose local dev where depends_on condition can use Redis, this is optional.
# HEALTHCHECK --interval=15s --timeout=5s --start-period=15s --retries=3 \
#   CMD wget --no-verbose --tries=1 --spider http://localhost:3001/health/live || exit 1

CMD ["node", "dist/main.js"]
```

**Key points:**
- **No `EXPOSE`** — workers have no HTTP server for business traffic. D10 documents a minimal HTTP health server on port 3001 for Kubernetes probes; if that pattern is implemented, the HEALTHCHECK above works. Otherwise, omit the healthcheck.
- In docker-compose, the workers service uses `depends_on` on Redis (with `condition: service_healthy`) and the migration service (with `condition: service_completed_successfully`). Docker Compose will wait for those dependencies but won't monitor the worker process itself — the restart policy handles crashes.
- `STOPSIGNAL SIGTERM` — aligns with D10's drain semantics: `worker.close()` waits for active jobs to complete.
- `start-period=15s` — longer than backend because workers need Redis connection + BullMQ worker initialization.

### Non-HTTP healthcheck for workers — options

D10 documents a minimal HTTP health server on a side port. This is the best practice for Kubernetes probes. For docker-compose, three options exist:

| Option | Docker Compose | K8s | Verdict |
|---|---|---|---|
| Minimal HTTP server on side port (D10 pattern) | Works | Required | Best — consistent across environments |
| `depends_on` Redis only, no worker healthcheck | Works | N/A | Acceptable for local dev only |
| `pgrep` process check | Works | Unreliable | Avoid — can't detect Redis disconnection |

**Recommendation:** Implement the minimal HTTP health server from D10 (port 3001, `/health/live` and `/health/ready`). It costs ~20 lines of code and gives consistent health checking across docker-compose and Kubernetes.

## Dockerfile — /apps/frontend (Next.js)

```dockerfile
# syntax=docker/dockerfile:1

# ---- Stage 1: Base ----
FROM node:24-alpine AS base
ENV PNPM_HOME="/pnpm"
ENV PATH="$PNPM_HOME/bin:$PATH"
RUN corepack enable && corepack prepare pnpm@11 --activate

# ---- Stage 2: Prune ----
FROM base AS pruner
WORKDIR /app
COPY . .
RUN pnpm dlx turbo prune @app/frontend --docker

# ---- Stage 3: Install dependencies ----
FROM base AS installer
WORKDIR /app
COPY --from=pruner /app/out/json/ .
RUN pnpm install --frozen-lockfile

# ---- Stage 4: Build ----
FROM installer AS builder
COPY --from=pruner /app/out/full/ .
RUN pnpm turbo run build --filter=@app/frontend...

# ---- Stage 5: Runner ----
FROM node:24-alpine AS runner
WORKDIR /app

RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nodejs

# 1. Copy public assets (favicon.ico, robots.txt, images, etc.)
COPY --from=builder --chown=nodejs:nodejs /app/apps/frontend/public ./public

# 2. Create writable .next directory for runtime cache
#    Next.js writes prerender cache and optimized images here at runtime.
RUN mkdir .next && chown nodejs:nodejs .next

# 3. Copy standalone server (server.js + traced node_modules)
COPY --from=builder --chown=nodejs:nodejs /app/apps/frontend/.next/standalone ./

# 4. Copy static assets into .next/static (JS/CSS bundles, hashed files)
COPY --from=builder --chown=nodejs:nodejs /app/apps/frontend/.next/static ./.next/static

USER nodejs
EXPOSE 3000
ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1
STOPSIGNAL SIGTERM

HEALTHCHECK --interval=15s --timeout=5s --start-period=15s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/ || exit 1

CMD ["node", "server.js"]
```

**Key points:**
- `output: 'standalone'` must be set in `next.config.ts` (D9). Without it, `.next/standalone` is not produced.
- `NEXT_TELEMETRY_DISABLED=1` — disables Next.js telemetry in production.
- **COPY sequence is order-sensitive:**
  1. `public/` first — static files that don't depend on the build
  2. `mkdir .next` — ensures the directory exists and is writable
  3. `.next/standalone/` — the server and traced dependencies
  4. `.next/static/` — the compiled JS/CSS chunks (must land at `.next/static`, NOT `./static`)
- `CMD ["node", "server.js"]` — Next.js standalone produces `server.js` in the standalone root. This is the entry point, not `next start`.
- The standalone output includes a pruned `node_modules` with only the dependencies traced by Next.js's dependency analysis. No `pnpm install` needed in the runner stage.
- HEALTHCHECK hits `/` (the homepage). Next.js doesn't ship a built-in health endpoint at `/health`, but `/` returns 200 on a healthy server. For production, consider adding a dedicated health route via a route handler or `middleware.ts`.

### Why standalone instead of full node_modules

Without `output: 'standalone'`, the runner stage would need the full `node_modules` directory (~500 MB+ for a Next.js project). The standalone output:
- Traces the dependency tree and copies only modules actually imported
- Includes the built Next.js server as `server.js`
- Produces a ~100-150 MB runner image instead of ~1 GB

The tradeoff is the manual COPY sequence above. The three separate COPY commands for `public`, `.next/standalone`, and `.next/static` are mandatory — the standalone directory does not contain `static` or `public`.

## Dockerfile — migration job (Drizzle)

The migration job runs `drizzle-kit migrate` before the backend and workers start. It is a one-shot container, not a long-running service.

### Approach: reuse the backend image

Since `@packages/db` is a dependency of `@app/backend`, `turbo prune @app/backend --docker` already includes `@packages/db`'s source and compiled output in the pruned output. Rather than building a separate image, the migration job uses the backend image with a different CMD:

```dockerfile
# Dockerfile.migrate — shares base, pruner, installer, builder with backend Dockerfile
# The runner stage differs: no HTTP server, runs migration then exits.

# syntax=docker/dockerfile:1

FROM node:24-alpine AS base
ENV PNPM_HOME="/pnpm"
ENV PATH="$PNPM_HOME/bin:$PATH"
RUN corepack enable && corepack prepare pnpm@11 --activate

FROM base AS pruner
WORKDIR /app
COPY . .
RUN pnpm dlx turbo prune @app/backend --docker

FROM base AS installer
WORKDIR /app
COPY --from=pruner /app/out/json/ .
RUN pnpm install --frozen-lockfile

FROM installer AS builder
COPY --from=pruner /app/out/full/ .
RUN pnpm turbo run build --filter=@app/backend...

FROM node:24-alpine AS runner
WORKDIR /app

RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nodejs

COPY --from=installer --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nodejs:nodejs /app/packages/db/dist ./packages/db/dist
COPY --from=builder --chown=nodejs:nodejs /app/packages/db/package.json ./packages/db/package.json
COPY --from=pruner /app/out/json/pnpm-workspace.yaml ./

# Copy migration SQL files — needed by drizzle-kit migrate
COPY --from=builder --chown=nodejs:nodejs /app/packages/db/src/migrations ./packages/db/src/migrations

USER nodejs
ENV NODE_ENV=production

# One-shot: run migrations and exit
CMD ["node", "-e", "require('drizzle-orm/postgres-js/migrator').migrate(require('drizzle-orm/postgres-js').drizzle(require('postgres')(process.env.DATABASE_URL, {max:1})), {migrationsFolder: './packages/db/src/migrations'}).then(() => {console.log('Migrations complete'); process.exit(0)}).catch((e) => {console.error(e); process.exit(1)})"]
```

**Simpler alternative — a dedicated migrate script in the backend image:**

Instead of a separate Dockerfile, include a `migrate.js` script in the backend project that the migration job references:

```ts
// apps/backend/src/migrate.ts — compiled to dist/migrate.js
import { drizzle } from 'drizzle-orm/postgres-js';
import { migrate } from 'drizzle-orm/postgres-js/migrator';
import postgres from 'postgres';

async function main() {
  const client = postgres(process.env.DATABASE_URL!, { max: 1 });
  const db = drizzle(client);
  await migrate(db, { migrationsFolder: './packages/db/src/migrations' });
  await client.end();
  console.log('Migrations complete');
}

main().catch((e) => {
  console.error(e);
  process.exit(1);
});
```

Then the migration Dockerfile reuses the backend's compiled output and runs:

```dockerfile
CMD ["node", "dist/migrate.js"]
```

**Recommendation:** Write a dedicated `src/migrate.ts` in the backend app. It's explicit, debuggable, and the backend image already has everything it needs (compiled `@packages/db`, migration SQL files, postgres.js). The migration job uses the same image as the backend with a different CMD — no separate image build required.

### Why migrations never run in the application CMD

1. **Startup ordering:** If migrations run as part of `node dist/main.js`, the application serves requests while migrations are still in progress. A request could hit a table whose migration hasn't completed.
2. **Failure isolation:** If migrations fail, the app shouldn't start. A one-shot container that exits non-zero prevents downstream services from starting (via `depends_on` with `condition: service_completed_successfully`).
3. **Concurrency:** In a multi-instance deployment, multiple backend replicas could attempt to run migrations simultaneously. A dedicated one-shot migration job runs exactly once.
4. **Idempotency:** `drizzle-kit migrate` is safe to run multiple times — it tracks applied migrations in `__drizzle_migrations` and only applies unapplied ones. Re-running it after a failed partial migration is safe.

## docker-compose.yml — local development

```yaml
# docker-compose.yml — local development only
# Production deployments use separate per-service configs.

services:
  # ---- Infrastructure ----
  postgres:
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
      POSTGRES_DB: app_dev
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app_dev"]
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 5s
    networks:
      - internal
    restart: unless-stopped

  redis:
    image: redis:8-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 5s
    networks:
      - internal
    restart: unless-stopped

  # ---- Migration (one-shot) ----
  migrate:
    build:
      context: .
      dockerfile: apps/backend/Dockerfile.migrate
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      DATABASE_URL: postgres://app:app@postgres:5432/app_dev
    networks:
      - internal
    restart: on-failure

  # ---- Applications ----
  backend:
    build:
      context: .
      dockerfile: apps/backend/Dockerfile
    ports:
      - "3000:3000"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
      migrate:
        condition: service_completed_successfully
    environment:
      NODE_ENV: development
      PORT: "3000"
      HOST: "0.0.0.0"
      DATABASE_URL: postgres://app:app@postgres:5432/app_dev
      REDIS_URL: redis://redis:6379
      LOG_LEVEL: info
      JWT_SECRET: dev-secret-change-in-production-min-32-chars
    networks:
      - internal
    restart: unless-stopped

  workers:
    build:
      context: .
      dockerfile: apps/workers/Dockerfile
    depends_on:
      redis:
        condition: service_healthy
      migrate:
        condition: service_completed_successfully
    environment:
      NODE_ENV: development
      REDIS_HOST: redis
      REDIS_PORT: "6379"
      LOG_LEVEL: info
      EMAIL_CONCURRENCY: "5"
      NOTIFICATION_CONCURRENCY: "10"
    networks:
      - internal
    restart: unless-stopped

  frontend:
    build:
      context: .
      dockerfile: apps/frontend/Dockerfile
    ports:
      - "3001:3000"
    depends_on:
      backend:
        condition: service_started
    environment:
      NODE_ENV: development
      NEXT_PUBLIC_API_URL: http://localhost:3000
    networks:
      - internal
    restart: unless-stopped

volumes:
  postgres_data:

networks:
  internal:
    driver: bridge
```

**Key points:**
- **`context: .`** — Docker builds from the repo root so `turbo prune` can see all packages. The `dockerfile` path is relative to context.
- **`restart: on-failure`** for the migration service — retries if PostgreSQL isn't ready yet, but doesn't restart once it succeeds (one-shot).
- **`depends_on` with `condition: service_completed_successfully`** — the migration service must complete successfully (exit code 0) before backend and workers start. This is the Compose v2 syntax; it replaces the legacy `condition: service_healthy` on short-lived services.
- **`depends_on` with `condition: service_healthy`** — backend and workers wait for PostgreSQL and Redis healthchecks to pass. The `healthcheck` definitions on postgres and redis enable this.
- **Frontend depends on backend `service_started`** — frontend can start before backend is fully ready (Next.js handles the loading state). Using `service_healthy` would add unnecessary startup delay.
- **Port mapping:** backend 3000:3000, frontend 3001:3000 (host:3001 → container:3000). Workers have no port mapping (no HTTP server for business traffic). Redis and PostgreSQL ports are exposed for local tool access.
- **No env_file** in this example — environment variables are inline for clarity. For projects with many variables, use `env_file: ./apps/backend/.env.local` per service.
- **`restart: unless-stopped`** on apps — restart on crash but not when explicitly stopped. The migration service uses `on-failure` to avoid restart loops on success.
- **Network:** All services share an `internal` bridge network. They communicate via service names (`postgres`, `redis`, `backend`).

### Migration service — restart policy

The migration service is a one-shot job. `restart: on-failure` means:
- If the migration exits with code 0 → no restart (success).
- If the migration exits with code non-zero → Docker restarts it (up to the default retry limit).
- Combined with `depends_on postgres: service_healthy`, this handles the case where PostgreSQL is slow to start — the migration will retry until it can connect.

## Environment variable management

### Build-time ARGs vs runtime ENV

| Type | Set via | When | Use for |
|---|---|---|---|
| `ARG` | `docker build --build-arg` or Compose `args:` | Build time only | `NEXT_PUBLIC_*` variables (Next.js inlines them at build) |
| `ENV` | Dockerfile `ENV` instruction | Baked into image (all containers) | Non-secret defaults: `NODE_ENV=production`, `NEXT_TELEMETRY_DISABLED=1` |
| Runtime env | `docker run -e` or Compose `environment:` | Container start | Secrets, per-environment config: `DATABASE_URL`, `REDIS_URL`, `JWT_SECRET` |

### Rules

1. **`DATABASE_URL`, `REDIS_URL`, `JWT_SECRET`** — never in a Dockerfile `ENV` instruction. Always injected at runtime via `docker run -e` or Compose `environment:`.
2. **`NEXT_PUBLIC_*` variables** — these are the exception. They are inlined into the client bundle at build time, so they must be available as `ARG` during `docker build`: `docker build --build-arg NEXT_PUBLIC_API_URL=https://api.example.com`. They CANNOT be changed at runtime — they're baked into the JS bundle.
3. **`NODE_ENV=production`** — safe to set as `ENV` in the Dockerfile runner stage. It's not a secret.
4. **`.env` files** — never committed (add to `.gitignore`). `.env.example` is committed with placeholder values. In docker-compose, use `env_file:` per service pointing to `.env.local` files, or use `environment:` directly.
5. **No `.env` in build context** — the `.dockerignore` excludes `.env*` (except `.env.example`) so secrets don't leak into image layers.

### Per-app .env files pattern

```
apps/backend/.env.example     ← committed: documents required vars
apps/backend/.env.local       ← gitignored: local dev values
apps/workers/.env.example
apps/workers/.env.local
apps/frontend/.env.example
apps/frontend/.env.local
```

In docker-compose:

```yaml
services:
  backend:
    env_file:
      - ./apps/backend/.env.local
```

Or use `environment:` directly for simpler setups with few variables.

### docker-compose env_file vs environment

| Approach | Best for |
|---|---|
| `environment:` block | Small number of variables, clarity, documentation-in-config |
| `env_file:` | Many variables, sharing between services, CI/CD integration |

For local development, `environment:` in the compose file is preferred — it keeps all configuration in one file and makes it obvious what each service needs.

## Drizzle migration deployment flow

### In docker-compose (local dev)

```
1. docker compose up -d postgres redis
2. docker compose run --rm migrate     ← runs drizzle-kit migrate, exits
3. docker compose up -d backend workers frontend
```

With `depends_on`, step 2 and 3 are automatic — `docker compose up` handles the ordering.

### In a real deployment pipeline

```
1. Build images:  docker build -t registry/app-backend:tag -f apps/backend/Dockerfile .
2. Push images:   docker push registry/app-backend:tag
3. Deploy migration job:
     docker run --rm \
       -e DATABASE_URL=postgres://... \
       registry/app-backend:tag \
       node dist/migrate.js
4. If migration succeeds (exit 0):
     docker run -d --name backend ... registry/app-backend:tag
     docker run -d --name workers ... registry/app-workers:tag
     docker run -d --name frontend ... registry/app-frontend:tag
```

**The migration must complete before the application containers start.** If the migration fails, the pipeline aborts and the previous version continues running (no downtime, no schema mismatch).

### Idempotency guarantee

`drizzle-kit migrate` is idempotent by design:
- It reads `__drizzle_migrations` to see which migrations have been applied.
- It only applies migrations not yet recorded in the table.
- Running it multiple times against the same database is safe — the second run applies zero migrations and exits successfully.

## Image optimization

### turbo prune size impact

Without pruning, a Docker build context for a monorepo includes every app's source and dependencies. `turbo prune @app/backend --docker` reduces the context to only `@app/backend` and its transitive workspace dependencies (`@packages/db`, `@packages/shared`, etc.). This:
- Reduces the `pnpm install` surface area (~70-90% fewer packages)
- Prevents cache invalidation from changes to unrelated apps
- Produces a smaller final image

### Layer cache ordering

The four-stage structure is designed for maximum cache hit rate:

```
BASE      → [cached forever unless Node.js/pnpm version changes]
PRUNER    → [re-runs on any file change, but turbo prune is fast (<1s)]
INSTALLER → [cached unless package.json/lockfile change]
BUILDER   → [re-runs on source changes]
RUNNER    → [minimal — only COPY, no RUN]
```

### Alpine image size

Using `node:24-alpine` as the base:
- Backend image: ~150-200 MB (Alpine + node_modules + dist)
- Workers image: ~150-200 MB
- Frontend image: ~150-200 MB (standalone includes traced node_modules)
- Migration image: same as backend (reuses the same image layers)

If image size is critical, the final `RUNNER` stage can omit `node_modules` not needed at runtime by using `pnpm install --prod` in a separate stage. For this stack, the size difference is negligible (~20 MB) and not worth the extra complexity of a second install stage.

### Production-only dependencies

In the shared pattern above, the `RUNNER` stage copies `node_modules` from the `INSTALLER` stage (which ran `pnpm install --frozen-lockfile` without `--prod`). This includes devDependencies. For production, add a separate `deps` stage:

```dockerfile
FROM base AS deps
WORKDIR /app
COPY --from=pruner /app/out/json/ .
RUN pnpm install --frozen-lockfile --prod
```

Then copy from `deps` instead of `installer` in the runner stage. This shaves ~20-50 MB but adds build complexity. For this stack, the simpler approach (copy all node_modules) is acceptable given Alpine's small base and the fact that devDependencies are mostly TypeScript types.

## Antipatterns and common mistakes

1. **Running `drizzle-kit migrate` in an app CMD or ENTRYPOINT.**
   Migrations must be a separate container that completes before the app starts. If the migration runs inside the app process, race conditions occur: the app may serve requests before the migration completes, or multiple replicas may attempt conflicting migrations.

2. **Missing `.next/static` or `public` in Next.js standalone COPY.**
   The standalone directory only contains `server.js` and traced `node_modules`. Forgetting to copy `.next/static` results in 404s for all JS/CSS bundles. Forgetting `public/` results in missing `favicon.ico`, `robots.txt`, etc.

3. **Running containers as root.**
   The `node` user exists in official Node.js images. Use `USER node` (or create a dedicated user). Running as root means a compromised application has root access to the container.

4. **Baking secrets into image layers.**
   `ENV DATABASE_URL=...` in a Dockerfile embeds the connection string in the image history (`docker history` reveals it). Always inject secrets at runtime.

5. **No STOPSIGNAL (relying on Docker default).**
   Docker's default is SIGTERM, but explicit `STOPSIGNAL SIGTERM` documents the contract between the Dockerfile and the application's shutdown handler (D8, D10). If the app uses a different signal or if the default changes, the explicit declaration prevents silent breakage.

6. **Workers starting before Redis is healthy.**
   Without `depends_on redis: service_healthy`, the worker container starts immediately and crashes because Redis isn't accepting connections. The `restart: unless-stopped` will retry, but this creates noisy crash loops during startup.

7. **Using `docker-compose push` for production deployments.**
   `docker compose` is for local development. Production deployments use a container registry and an orchestrator. The compose file documents the local setup; it is not a production deployment manifest.

8. **Copying the full monorepo into the runner stage.**
   Every app only needs its own `dist/` (or `.next/standalone`) and its dependencies. Copying the full monorepo produces a 1 GB+ image with every app's source code.

9. **Not pinning the pnpm version.**
   `corepack enable` without `corepack prepare pnpm@11 --activate` uses whatever pnpm version Corepack defaults to, which may change. Pin the major version.

10. **Forgetting `pnpm-workspace.yaml` in the runner stage.**
    If the runner stage uses `workspace:*` dependencies (e.g., `@packages/db`), pnpm needs `pnpm-workspace.yaml` to resolve them. The pruned output includes this file — copy it.

## Reference snippets

### Dockerfile.backend (full, annotated, multi-stage with turbo prune)

```dockerfile
# syntax=docker/dockerfile:1

FROM node:24-alpine AS base
ENV PNPM_HOME="/pnpm"
ENV PATH="$PNPM_HOME/bin:$PATH"
RUN corepack enable && corepack prepare pnpm@11 --activate

FROM base AS pruner
WORKDIR /app
COPY . .
RUN pnpm dlx turbo prune @app/backend --docker

FROM base AS installer
WORKDIR /app
COPY --from=pruner /app/out/json/ .
RUN pnpm install --frozen-lockfile

FROM installer AS builder
COPY --from=pruner /app/out/full/ .
RUN pnpm turbo run build --filter=@app/backend...

FROM node:24-alpine AS runner
WORKDIR /app
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nodejs
COPY --from=installer --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nodejs:nodejs /app/apps/backend/dist ./dist
COPY --from=builder --chown=nodejs:nodejs /app/apps/backend/package.json .
COPY --from=builder --chown=nodejs:nodejs /app/packages/db/dist ./packages/db/dist
COPY --from=builder --chown=nodejs:nodejs /app/packages/db/package.json ./packages/db/package.json
COPY --from=pruner /app/out/json/pnpm-workspace.yaml ./
USER nodejs
EXPOSE 3000
ENV NODE_ENV=production
STOPSIGNAL SIGTERM
HEALTHCHECK --interval=15s --timeout=5s --start-period=10s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health/live || exit 1
CMD ["node", "dist/main.js"]
```

### Dockerfile.frontend (full, annotated, Next.js standalone)

```dockerfile
# syntax=docker/dockerfile:1

FROM node:24-alpine AS base
ENV PNPM_HOME="/pnpm"
ENV PATH="$PNPM_HOME/bin:$PATH"
RUN corepack enable && corepack prepare pnpm@11 --activate

FROM base AS pruner
WORKDIR /app
COPY . .
RUN pnpm dlx turbo prune @app/frontend --docker

FROM base AS installer
WORKDIR /app
COPY --from=pruner /app/out/json/ .
RUN pnpm install --frozen-lockfile

FROM installer AS builder
COPY --from=pruner /app/out/full/ .
RUN pnpm turbo run build --filter=@app/frontend...

FROM node:24-alpine AS runner
WORKDIR /app
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nodejs
COPY --from=builder --chown=nodejs:nodejs /app/apps/frontend/public ./public
RUN mkdir .next && chown nodejs:nodejs .next
COPY --from=builder --chown=nodejs:nodejs /app/apps/frontend/.next/standalone ./
COPY --from=builder --chown=nodejs:nodejs /app/apps/frontend/.next/static ./.next/static
USER nodejs
EXPOSE 3000
ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1
STOPSIGNAL SIGTERM
HEALTHCHECK --interval=15s --timeout=5s --start-period=15s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/ || exit 1
CMD ["node", "server.js"]
```

### Dockerfile.workers (full, annotated, no HTTP)

```dockerfile
# syntax=docker/dockerfile:1

FROM node:24-alpine AS base
ENV PNPM_HOME="/pnpm"
ENV PATH="$PNPM_HOME/bin:$PATH"
RUN corepack enable && corepack prepare pnpm@11 --activate

FROM base AS pruner
WORKDIR /app
COPY . .
RUN pnpm dlx turbo prune @app/workers --docker

FROM base AS installer
WORKDIR /app
COPY --from=pruner /app/out/json/ .
RUN pnpm install --frozen-lockfile

FROM installer AS builder
COPY --from=pruner /app/out/full/ .
RUN pnpm turbo run build --filter=@app/workers...

FROM node:24-alpine AS runner
WORKDIR /app
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nodejs
COPY --from=installer --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nodejs:nodejs /app/apps/workers/dist ./dist
COPY --from=builder --chown=nodejs:nodejs /app/apps/workers/package.json .
COPY --from=builder --chown=nodejs:nodejs /app/packages/db/dist ./packages/db/dist
COPY --from=builder --chown=nodejs:nodejs /app/packages/db/package.json ./packages/db/package.json
COPY --from=pruner /app/out/json/pnpm-workspace.yaml ./
USER nodejs
ENV NODE_ENV=production
STOPSIGNAL SIGTERM
CMD ["node", "dist/main.js"]
```

### Dockerfile.migrate (drizzle-kit migrate one-shot)

```dockerfile
# syntax=docker/dockerfile:1

FROM node:24-alpine AS base
ENV PNPM_HOME="/pnpm"
ENV PATH="$PNPM_HOME/bin:$PATH"
RUN corepack enable && corepack prepare pnpm@11 --activate

FROM base AS pruner
WORKDIR /app
COPY . .
RUN pnpm dlx turbo prune @app/backend --docker

FROM base AS installer
WORKDIR /app
COPY --from=pruner /app/out/json/ .
RUN pnpm install --frozen-lockfile

FROM installer AS builder
COPY --from=pruner /app/out/full/ .
RUN pnpm turbo run build --filter=@app/backend...

FROM node:24-alpine AS runner
WORKDIR /app
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nodejs
COPY --from=installer --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nodejs:nodejs /app/packages/db/dist ./packages/db/dist
COPY --from=builder --chown=nodejs:nodejs /app/packages/db/package.json ./packages/db/package.json
COPY --from=builder --chown=nodejs:nodejs /app/packages/db/src/migrations ./packages/db/src/migrations
COPY --from=pruner /app/out/json/pnpm-workspace.yaml ./
USER nodejs
ENV NODE_ENV=production
CMD ["node", "-e", "const{ migrate }=require('drizzle-orm/postgres-js/migrator');const{ drizzle }=require('drizzle-orm/postgres-js');const postgres=require('postgres');const c=postgres(process.env.DATABASE_URL,{max:1});const db=drizzle(c);migrate(db,{migrationsFolder:'./packages/db/src/migrations'}).then(()=>{console.log('Migrations complete');process.exit(0)}).catch(e=>{console.error(e);process.exit(1)})"]
```

### apps/backend/src/migrate.ts (dedicated migration script)

```ts
// apps/backend/src/migrate.ts — compiled to dist/migrate.js
// Run as: node dist/migrate.js
// Before starting the backend and workers.

import { drizzle } from 'drizzle-orm/postgres-js';
import { migrate } from 'drizzle-orm/postgres-js/migrator';
import postgres from 'postgres';

async function main() {
  const client = postgres(process.env.DATABASE_URL!, { max: 1 });
  const db = drizzle(client);

  await migrate(db, {
    migrationsFolder: './packages/db/src/migrations',
  });

  await client.end();
  console.log('Migrations complete');
}

main().catch((e) => {
  console.error('Migration failed:', e);
  process.exit(1);
});
```

### .dockerignore (monorepo-aware)

```
node_modules
.turbo
.git
**/.next/cache
**/coverage
**/dist
**/*.md
.env
.env.*
!.env.example
.gitignore
Dockerfile*
docker-compose*.yml
```

### docker-compose.yml (full local dev, annotated)

```yaml
services:
  postgres:
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
      POSTGRES_DB: app_dev
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app_dev"]
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 5s
    networks:
      - internal
    restart: unless-stopped

  redis:
    image: redis:8-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 5s
    networks:
      - internal
    restart: unless-stopped

  migrate:
    build:
      context: .
      dockerfile: apps/backend/Dockerfile.migrate
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      DATABASE_URL: postgres://app:app@postgres:5432/app_dev
    networks:
      - internal
    restart: on-failure

  backend:
    build:
      context: .
      dockerfile: apps/backend/Dockerfile
    ports:
      - "3000:3000"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
      migrate:
        condition: service_completed_successfully
    environment:
      NODE_ENV: development
      PORT: "3000"
      HOST: "0.0.0.0"
      DATABASE_URL: postgres://app:app@postgres:5432/app_dev
      REDIS_URL: redis://redis:6379
      LOG_LEVEL: info
      JWT_SECRET: dev-secret-change-in-production-min-32-chars
    networks:
      - internal
    restart: unless-stopped

  workers:
    build:
      context: .
      dockerfile: apps/workers/Dockerfile
    depends_on:
      redis:
        condition: service_healthy
      migrate:
        condition: service_completed_successfully
    environment:
      NODE_ENV: development
      REDIS_HOST: redis
      REDIS_PORT: "6379"
      LOG_LEVEL: info
      EMAIL_CONCURRENCY: "5"
      NOTIFICATION_CONCURRENCY: "10"
    networks:
      - internal
    restart: unless-stopped

  frontend:
    build:
      context: .
      dockerfile: apps/frontend/Dockerfile
    ports:
      - "3001:3000"
    depends_on:
      backend:
        condition: service_started
    environment:
      NODE_ENV: development
      NEXT_PUBLIC_API_URL: http://localhost:3000
    networks:
      - internal
    restart: unless-stopped

volumes:
  postgres_data:

networks:
  internal:
    driver: bridge
```

## Sources used

- [D3 — Turborepo 2.x skill (turbo prune Docker pattern)](./D3-multi-app-monorepo.md) — prior skill
- [D8 — Fastify backend skill (SIGTERM shutdown, /health endpoint)](./D8-backend-app-fastify.md) — prior skill
- [D9 — Frontend skill (Next.js standalone output, next.config.ts)](./D9-frontend-app.md) — prior skill
- [D10 — Workers skill (SIGTERM drain, no HTTP server, health HTTP option)](./D10-workers-app.md) — prior skill
- [S1 — Drizzle ORM skill (no binary, separate migration step)](./S1-data-layer-drizzle-orm.md) — prior skill
- [Next.js Deploying Documentation](https://nextjs.org/docs/app/building-your-application/deploying) — official
- [Next.js with-docker Example (GitHub)](https://github.com/vercel/next.js/tree/canary/examples/with-docker) — official
- [Docker Docs: Containerize Next.js](https://docs.docker.com/guides/nextjs/containerize/) — official
- [pnpm Docker Installation](https://pnpm.io/docker) — official
- [Turborepo Deploying with Docker](https://turborepo.dev/docs/handbook/deploying-with-docker) — official
- [Docker Compose services spec (depends_on, healthcheck)](https://docs.docker.com/compose/compose-file/05-services/) — official
- [Docker Dockerfile reference (HEALTHCHECK, STOPSIGNAL, USER)](https://docs.docker.com/reference/dockerfile/) — official
- [Node.js Release Schedule](https://github.com/nodejs/Release/blob/main/README.md) — official
- [Corepack removal from Node.js 25](https://nodejs.org/en/blog/announcements/v25-release-announce) — official
