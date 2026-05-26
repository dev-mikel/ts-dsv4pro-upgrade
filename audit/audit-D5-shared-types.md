# Audit D5 — Shared Types & Contracts

## 1. Knowledge Level: Solid (architectural, not tool-specific)

This domain is more about design patterns than a specific tool version. I understand: barrel exports from `@shared-types`, the distinction between types and runtime schemas, coupling risks (shared importing from apps), contract-first API design, job payload types shared between backend and workers, Zod schema colocation with types in shared, `z.infer` as the bridge, and versioning considerations for shared packages.

## 2. Versions Covered

N/A — architectural knowledge. The patterns themselves are version-independent, though their implementation depends on TypeScript (D1) and Zod (D4) versions.

## 3. Identified Gaps

- **pnpm `workspace:` protocol versions**: I know `workspace:*` but don't know if pnpm 10.x introduced any changes to workspace protocol versioning (e.g., `workspace:^` behavior).
- **Package export maps (`package.json` `exports` field)**: I know the concept but don't know the current best practice for multi-entry shared packages (e.g., `@shared-types/schemas`, `@shared-types/utils`) and how they interact with TypeScript's `moduleResolution` and `paths`.
- **Barrel export performance**: I know barrel files can cause tree-shaking issues and slow type-checking, but I don't know the current recommended mitigation (e.g., `internal` package.json exports to block imports).
- **Contract versioning for workers**: When the backend enqueues a job with a payload type, and the worker dequeues it, version mismatches are a deployment-coordination problem. I understand the problem space but don't know the current recommended patterns (schema version field? separate queues per version?).
- **`@types` package scope**: I'm uncertain whether placing types in `@shared-types` (not `@types/shared-types`) is the current convention or if there's a reason to avoid the `@types` scope.
- **Type-only imports in barrel files**: `import type` vs `import` in re-export barrels — I know the distinction but may not know the latest TypeScript enforcements (e.g., `verbatimModuleSyntax` interactions).

## 4. Hallucination Risk

Highest risk areas:
- Recommending `package.json` `exports` field patterns that don't work with the project's specific `moduleResolution` setting.
- Suggesting coupling patterns (e.g., "just import from the backend") that violate the architecture's macro-service boundary.
- Inventing contract versioning strategies that are unnecessarily complex or don't align with current best practice.

## 5. Confidence: 8/10

The principles are stable and version-independent. Implementation details depend on D1 (TS), D3 (pnpm), and D4 (Zod), which is where the real uncertainty lives.

## Intersection Notes

- **D8 ↔ D10 (backend ↔ workers)**: The shared job payload types are the contract between enqueuer and dequeuer. I need to verify the current best practice for:
  - Shared types location (`@shared-types/jobs` or colocated with the worker?)
  - Whether Zod schemas should be shared too (so the worker can validate at the boundary)
  - How to handle schema evolution during rolling deployments
- **D4 ↔ D5 (Zod ↔ shared)**: Should Zod schemas live in `@shared-types` or closer to the consumer? The current recommended pattern for "single source of truth" matters.
- **D8 ↔ D9 (backend ↔ frontend)**: Shared API response types — should the frontend import types from `@shared-types` or generate them from OpenAPI? I don't know the current recommended approach for this architecture.

## Recommended Phase 2 Search Queries

1. `pnpm 10.x workspace protocol versioning 2025`
2. `package.json exports field TypeScript monorepo best practice 2025`
3. `barrel file performance TypeScript alternatives 2025`
4. `shared job payload types backend workers monorepo pattern`
5. `Zod schema sharing backend frontend monorepo pattern`
6. `schema evolution rolling deployment contract versioning`
