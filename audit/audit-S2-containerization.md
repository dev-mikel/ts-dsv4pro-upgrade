# Audit S2 — Containerization (Docker + turbo prune + docker-compose)

## 1. Knowledge Level: Solid (fundamental patterns stable, integration gaps)

I know Docker deeply: multi-stage builds, `FROM`/`COPY`/`RUN`, layer caching, `.dockerignore`, `ARG`/`ENV`, `WORKDIR`, `EXPOSE`, `HEALTHCHECK` (with `CMD` or `CMD-SHELL`), `ENTRYPOINT` vs `CMD`, `USER` for non-root, `docker compose` (`services`, `volumes`, `networks`, `depends_on`, `environment`, `healthcheck`). I know the `turbo prune` flow for creating a minimal Docker context.

## 2. Versions Covered

- **Docker / BuildKit**: Confident through the BuildKit v1 era (Docker Engine ~24.x–25.x). BuildKit v2 may exist.
- **docker compose**: Confident through Compose v2.x (plugin, not docker-compose Python). The spec is relatively stable.
- **turbo prune**: Confident through Turborepo v1.x. **2.x prune behavior is unknown** (see D3 gap).

## 3. Identified Gaps

- **Turborepo 2.x `turbo prune`**: This is the critical S2 gap inherited from D3. The prune output directory structure, `--docker` flag behavior, and the generated `package.json` format may have changed. This directly impacts multi-stage Dockerfile correctness.
- **BuildKit cache mounting (`--mount=type=cache`)**: I know the basic pattern for caching `node_modules` and `pnpm store`. I don't know the current recommended `RUN --mount=type=cache` targets for pnpm store in 2025.
- **pnpm in Docker**: I know the recommended approach is to use `corepack enable` or install pnpm via npm. I don't know if there's a newer pattern (e.g., official pnpm Docker image, or `pnpm deploy` for slimmer containers).
- **`pnpm deploy`**: pnpm 8.x+ has a `pnpm deploy` command that produces a deploy-ready directory with only production dependencies. I don't know if this replaces `turbo prune` for some use cases or how they compare.
- **Docker healthcheck for non-HTTP services**: For the workers app (BullMQ), a simple HTTP health check won't work. I don't know the current recommended pattern (Redis ping? Custom script?).
- **`docker compose` profiles**: I know profiles exist for conditional service startup. I don't know if the syntax or behavior changed.
- **Multi-architecture builds (`docker buildx`)**: I know it exists. I don't know if it's now the default or still opt-in.
- **Secrets management in Docker**: I know `--secret` / `secrets` in compose. I don't know the current best practice for build-time vs run-time secrets.
- **Watch mode in compose**: `docker compose watch` (vs `docker compose up --watch`) — I don't know the current syntax or if it's stable.

## 4. Hallucination Risk

Highest risk areas:
- **Writing a multi-stage Dockerfile with incorrect `turbo prune` flags for Turborepo 2.x. This is a concrete, high-impact error.**
- Recommending `pnpm store` cache mount paths that don't match the current pnpm version's store layout.
- Incorrect healthcheck syntax for workers (abusing `CMD` when `CMD-SHELL` is needed, or vice versa).
- Suggesting `docker-compose` (Python v1) when the project uses `docker compose` (plugin v2).

## 5. Confidence: 7/10

Core Docker patterns are stable and I know them well. The primary risks are Turborepo 2.x integration (inherited from D3) and pnpm-specific Docker optimizations that may have evolved. The fundamentals are solid.

## Intersection Notes

- **D3 (Turborepo)**: `turbo prune` is the bridge. Any change in 2.x prune behavior directly breaks the Dockerfile. This is the highest-risk intersection in the entire architecture.
- **D8 (Fastify)**: Backend Dockerfile needs `EXPOSE` for the Fastify port, and an HTTP-based healthcheck.
- **D9 (Next.js)**: Frontend Dockerfile needs `output: "standalone"` and the correct `COPY` paths from the standalone output.
- **D10 (workers)**: Workers Dockerfile needs a non-HTTP healthcheck and must handle SIGTERM correctly for graceful job drain.
- **D1 (TypeScript)**: Build output structure (where `dist/` lands) must match what the Dockerfile `COPY`s.
- **S1 (Prisma)**: Database migrations — run in Docker (init container, startup script, or separate job). The Prisma client must be generated in the Docker build (or pre-built).

## Recommended Phase 2 Search Queries

1. `Turborepo 2.x turbo prune Docker multi-stage build 2025`
2. `pnpm deploy vs turbo prune Docker comparison 2025`
3. `pnpm store path Docker BuildKit cache mount 2025`
4. `Docker healthcheck non-HTTP worker process BullMQ`
5. `Next.js standalone output Docker 15 2025`
6. `docker compose profiles 2025 syntax`
7. `BuildKit v2 multi-architecture buildx 2025`
