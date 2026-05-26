# Audit D8 — Backend App (Fastify)

## 1. Knowledge Level: Solid (with Fastify 5.x uncertainty)

I know Fastify 4.x deeply: plugin registration model (`register` with `prefix`), encapsulated contexts, `fastify.addHook`, `fastify.decorate`/`decorateRequest`/`decorateReply`, route schemas with JSON Schema (or Zod via type providers), `fastify.setErrorHandler`, `fastify.setNotFoundHandler`, graceful shutdown (`fastify.close`), `fastify.inject` for testing, `@fastify/cors`, `@fastify/cookie`, `@fastify/env` (for env validation), `fastify.get`/`post`/`put`/`delete` with typed route handlers.

## 2. Versions Covered

- **Fastify**: Confident through v4.x (through early 2024). **Fastify 5.x is a major unknown** — I don't know if it shipped, what the breaking changes are, or whether the plugin ecosystem migrated.
- **@fastify/type-provider-zod** (or `fastify-type-provider-zod`): I don't know which package name is current or if it supports Fastify 5.x.
- **@fastify/env**: Confident through the 2023-era version. Schema format may have changed.

## 3. Identified Gaps

- **Fastify 5.x**: This is the dominant gap. Potential changes I'm unaware of: plugin API changes, lifecycle hook changes, request/reply type changes, serialization changes, route registration changes, error handling changes. Fastify 5 was planned to remove deprecated APIs — I don't know which ones were actually removed.
- **Zod type provider integration**: In Fastify 4.x, the pattern is `new ZodTypeProvider()` and `fastify.withTypeProvider<ZodTypeProvider>()`. I don't know if this changed in 5.x or if Zod type providers are now natively supported.
- **`@fastify/env` vs manual Zod env validation**: I don't know the current recommendation. Using Zod directly for env validation (with `z.coerce` for string env vars) may now be preferred over `@fastify/env`.
- **Route type inference with Zod**: `fastify.post('/', { schema: { body: zodSchema } }, handler)` — I know the body/response types flow through. I don't know if the inference improved in recent versions (e.g., better error messages for type mismatches).
- **Graceful shutdown pattern**: I know the SIGTERM/SIGINT handler with `fastify.close()`. I don't know if Fastify 5.x changed the close behavior (e.g., drain timeout, keep-alive handling).
- **`@fastify/swagger` / OpenAPI generation**: I know it exists. I don't know the current recommended setup, whether it supports Zod schemas directly, or if alternatives exist.
- **Plugin-per-module structure**: I understand the pattern (each domain module is a plugin). I don't know if Fastify 5.x changed plugin encapsulation semantics.

## 4. Hallucination Risk

Highest risk areas:
- Writing Fastify 4.x plugin registration code that doesn't compile in Fastify 5.x.
- Using the wrong type provider API (v4 vs v5).
- Recommending `@fastify/env` config format that changed.
- Assuming `fastify.close()` behavior that was refined in 5.x.
- Incorrectly suggesting the Zod type provider package name.

## 5. Confidence: 6/10

I'm solid on Fastify concepts and patterns. The Fastify 5.x version uncertainty is the primary drag — if the project is on 4.x, I'm at 8/10. If on 5.x, 4/10.

## Intersection Notes

- **D4 (Zod)**: The Zod ↔ Fastify bridge is critical. Route schemas, request validation, and response serialization all depend on correct type provider integration.
- **D5 (shared types)**: API types (`request.body`, `reply` shapes) likely come from `@shared-types`. The Fastify route handler types must match.
- **D7 (linting)**: Fastify plugin modules need `no-restricted-imports` rules — e.g., prevent importing from `@apps/frontend`.
- **S2 (Docker)**: The Fastify Dockerfile needs the correct build output from Turborepo prune.
- **D10 (workers)**: Backend enqueues BullMQ jobs — the job payload type comes from `@shared-types` and must match the worker's expectation.
- **D6 (testing)**: `app.inject()` tests need to work with the current Fastify version's lifecycle and type provider.

## Recommended Phase 2 Search Queries

1. `Fastify 5.x migration guide breaking changes 2025`
2. `Fastify 5.x Zod type provider integration`
3. `@fastify/env vs Zod env validation Fastify 2025`
4. `Fastify graceful shutdown patterns 5.x`
5. `Fastify plugin encapsulation changes 5.x`
6. `@fastify/swagger OpenAPI Zod schema 2025`
