# Audit D6 — Per-App Testing (Vitest + expect-type + Fastify inject)

## 1. Knowledge Level: Solid (with integration-version gaps)

I know Vitest well: workspace config (`vitest.workspace.ts`), test structure, mocking (`vi.fn`, `vi.mock`, `vi.spyOn`), assertions, coverage via `c8`/`v8`, `test.each`, `describe`/`it`/`beforeEach`, `expect` API, snapshot testing, and `vi.waitFor`. I know `expect-type` conceptually (compile-time type assertions). I know Fastify's `app.inject()` for integration testing without a real HTTP server.

## 2. Versions Covered

- **Vitest**: Confident through ~1.6.x (early 2024). v2.x or v3.x may exist by now.
- **expect-type**: I know it exists and the core API (`expectTypeOf`, `.toEqualTypeOf`, `.toMatchTypeOf`, `.not`) but don't know the current version.
- **Fastify inject**: I know the pattern. Version coupling with Fastify itself (D8).

## 3. Identified Gaps

- **Vitest 2.x/3.x**: I don't know if a major version shipped, what changed, or whether the workspace config format evolved. The `vitest.workspace.ts` vs `vitest.workspace.json` vs inline array in `vitest.config.ts` options may have changed.
- **Vitest + Turborepo**: I don't know the current recommended integration pattern — whether tests should be their own Turborepo task, or if `vitest --run` integrates with Turbo's caching.
- **`expect-type` updates**: I don't know if the API surface changed, particularly around `.branded`, `.toBeAny`, `.toBeNever`, `.toBeUnknown`, or the new `Equal`/`Expect` type-level assertion API.
- **Fastify `inject` with async context**: I don't know the current best practice for testing request-scoped services (e.g., database transactions, auth context) with `inject`.
- **Vitest browser mode**: I know it exists experimentally. I don't know if it's stable or recommended for frontend testing vs jsdom/happy-dom.
- **Vitest typechecking**: I know Vitest can run `tsc --noEmit` tests. I don't know if the integration changed or if `expect-type` is the recommended replacement.
- **Test isolation in a monorepo**: I don't know the current recommended way to run only "affected" tests (e.g., "my change to `@shared-types` means these 3 apps' tests should run").

## 4. Hallucination Risk

Highest risk areas:
- Writing Vitest 1.x config for a Vitest 3.x workspace.
- Incorrectly configuring Turborepo test task dependencies (testing depends on build, but should tests be cached?).
- Recommending `expect-type` APIs that don't exist.
- Assuming `app.inject()` behavior that changed with Fastify 5.x.

## 5. Confidence: 6/10

Vitest version uncertainty plus the multi-factor integration with Turborepo and Fastify drops confidence. The testing concepts are solid; the exact config is version-sensitive.

## Intersection Notes

- **D3 (Turborepo)**: Test tasks in `turbo.json` must be configured correctly. I don't know the 2.x syntax for test pipelines vs build pipelines.
- **D8 (Fastify)**: `app.inject()` is coupled to Fastify's internal request lifecycle. If Fastify 5.x changed the lifecycle, inject behavior changes.
- **D4 (Zod)**: Testing validation logic depends on Zod schemas. If Zod 4 changed behavior, validation tests may pass/fail incorrectly.
- **D9 (frontend)**: Frontend test configuration in a monorepo (jsdom, RTL, which tsconfig) needs to align with the build config.

## Recommended Phase 2 Search Queries

1. `Vitest 3.x changelog workspace config migration 2025`
2. `Vitest Turborepo integration test caching best practice 2025`
3. `expect-type latest API reference 2025`
4. `Fastify inject testing request-scoped async context`
5. `Vitest typecheck expect-type recommended approach 2025`
