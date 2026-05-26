# Audit D3 — Multi-App Monorepo (pnpm + Turborepo)

## 1. Knowledge Level: Solid (with major Turborepo 2.x uncertainty)

I understand pnpm workspaces deeply: `pnpm-workspace.yaml`, `workspace:` protocol, catalog support, `overrides`, hoisting, `injected` dependencies. I know Turborepo concepts: `turbo.json`, pipelines with `dependsOn`, `outputs` for caching, `inputs`, `env` dependencies, `turbo prune`, internal packages with `workspace:*`, and the `apps/` vs `packages/` distinction.

## 2. Versions Covered

- **pnpm**: Confident through v9.x (2024). v10.x may have workspaces behavior changes.
- **Turborepo**: Confident through v1.13.x (early 2024). **Turborepo 2.x is my largest single architecture-level gap.** I know it exists and that the configuration format changed significantly, but I don't know the exact new schema.

## 3. Identified Gaps

- **Turborepo 2.x `turbo.json` schema**: This is a critical gap. I know v1.x syntax (`pipeline` key, `dependsOn`, `outputs`, `inputs`, `env`). Turborepo 2.x reportedly moved to a `tasks` key instead of `pipeline`, changed caching configuration, and may have removed/changed `dependsOn` syntax. I don't know the exact migration path or backward compatibility.
- **`turbo prune` in 2.x**: I know the concept (create a sparse sub-tree for Docker). I don't know if the CLI flags, output directory structure, or behavior changed in 2.x.
- **pnpm catalogs**: Introduced in pnpm 9.x. I know the concept (`pnpm-workspace.yaml` `catalog:` field) but don't know the exact syntax, limitations, or whether it's considered stable.
- **pnpm `injected` dependencies**: I know the concept (for local packages that need peer deps resolved) but don't know the current best practice or edge cases.
- **Turborepo `globalDependencies` vs root `inputs`**: I don't know the current recommended way to express "rebuild everything if the root tsconfig changes."
- **Remote caching**: I know Turborepo supports Vercel remote cache; I don't know the current alternatives or self-hosted options.
- **`turbo watch`**: I don't know if Turborepo has a watch/dev mode for monorepo development (I recall it being experimental or newly added).

## 4. Hallucination Risk

Highest risk areas:
- **Writing `turbo.json` in v1.x syntax when the project uses Turborepo 2.x — this is the highest-impact hallucination risk in the entire architecture.**
- Recommending `turbo prune` flags that changed in 2.x.
- Suggesting pnpm workspaces configurations that don't align with the current pnpm version.
- Confidently explaining Turborepo cache behavior that changed between versions.

## 5. Confidence: 5/10

pnpm is solid. Turborepo 2.x is a major unknown — I know the *concepts* but not the *current configuration surface*. This is a must-research domain before touching any `turbo.json` or build pipeline code.

## Intersection Notes

- **S2 (Docker)**: `turbo prune` is the bridge between the monorepo and Docker builds. If the prune behavior changed in 2.x, multi-stage Dockerfiles break.
- **D2 (build tools)**: Turborepo `outputs` must match what tsup/esbuild actually produces. Changing either side breaks caching.
- **D8/D9/D10 (apps)**: Each app's `turbo.json` task configuration depends on the 2.x schema.
- **D5 (shared types)**: Internal package dependency graph resolution in Turborepo determines build order. If 2.x changed how `dependsOn` works for `workspace:*` packages, the entire pipeline topology is affected.

## Recommended Phase 2 Search Queries

1. `Turborepo 2.x turbo.json migration guide pipeline to tasks`
2. `Turborepo 2.x configuration reference 2025`
3. `Turborepo 2.x turbo prune Docker multi-stage`
4. `pnpm 10.x workspaces catalog breaking changes`
5. `Turborepo 2.x remote caching self-hosted`
6. `Turborepo 2.x watch mode dev`
