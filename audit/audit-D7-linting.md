# Audit D7 — Linting & Formatting

## 1. Knowledge Level: Solid (with ESLint flat config gap)

I know: typescript-eslint type-aware linting, parser configuration (`parserOptions.project`, `tsconfigRootDir`), `@typescript-eslint/no-unused-vars`, `no-restricted-imports` to enforce architectural boundaries (e.g., prevent `/apps/backend` from importing `/apps/frontend`), `import/no-cycle` for detecting circular dependencies, Prettier integration (either via `eslint-plugin-prettier` or `eslint-config-prettier` for disabling conflicting rules).

## 2. Versions Covered

- **ESLint**: Confident through v8.x with `.eslintrc`. **ESLint v9's flat config (`eslint.config.js`) is a known but not deeply internalized gap.**
- **typescript-eslint**: Confident through v7.x. v8.x may have shipped alongside ESLint 9.
- **Prettier**: Stable API. Confident through 3.x.
- **eslint-plugin-import / import/no-cycle**: I know it exists but don't know the current maintenance status or natively-supported alternatives.

## 3. Identified Gaps

- **ESLint flat config (`eslint.config.js`)**: This is the single largest gap in D7. The entire configuration format changed. I know it's ESM-native, uses `@eslint/js` and `typescript-eslint`'s recommended configs differently, and that `.eslintrc` files are deprecated. I don't know the exact migration pattern, how type-aware rules are configured in flat config, or how `tsconfigRootDir` maps to the new format.
- **`typescript-eslint` v8**: I don't know which rules were added, removed, or changed. The `strict`/`stylistic` rule split may have evolved.
- **`no-restricted-imports` in flat config**: The rule exists but the configuration format may differ.
- **`import/no-cycle` in flat config**: The `eslint-plugin-import` package may or may not support flat config. Alternatives like `eslint-plugin-import-x` may be the current recommendation.
- **Prettier + flat config**: I know `eslint-config-prettier` has a flat config mode. I don't know the exact setup or if Prettier integration is recommended differently now.
- **Turborepo lint task**: I don't know the 2.x pipeline configuration for linting in a monorepo (should lint be per-package, or root-level?).

## 4. Hallucination Risk

Highest risk areas:
- Writing `.eslintrc`-style config when the project uses ESLint v9 flat config. This is a file-format level mismatch that would immediately fail.
- Configuring `parserOptions.project` in the old format for a flat config project.
- Recommending `eslint-plugin-import` for a project that has already migrated to `eslint-plugin-import-x`.
- Assuming Prettier integration works the same way as v8.

## 5. Confidence: 5/10

If the project is on ESLint v8 + .eslintrc, I'm at 8/10. If on ESLint v9 + flat config (which is increasingly likely for a project started/updated in 2024-2025), I drop to 3/10 — I know the concept but not the concrete configuration surface.

## Intersection Notes

- **D3 (Turborepo)**: Lint tasks in `turbo.json` need correct 2.x configuration. Root-level vs per-package linting strategy affects pipeline design.
- **D5 (shared types)**: `no-restricted-imports` enforces the architectural boundary between apps and packages. Getting this rule wrong silently allows forbidden imports.
- **D2 (tsconfig)**: Type-aware linting depends on correct `tsconfig` paths. If the project uses project references, the linting tsconfig must include all referenced projects.
- **D8/D9/D10 (apps)**: Each app may need its own tsconfig for type-aware linting, inheriting from a root base.

## Recommended Phase 2 Search Queries

1. `ESLint v9 flat config migration guide typescript-eslint 2025`
2. `typescript-eslint v8 flat config setup parserOptions project`
3. `eslint-plugin-import flat config eslint-plugin-import-x 2025`
4. `eslint-config-prettier flat config setup 2025`
5. `Turborepo lint task configuration 2.x monorepo`
6. `no-restricted-imports flat config TypeScript monorepo boundaries`
