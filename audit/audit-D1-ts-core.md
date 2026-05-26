# Audit D1 — TypeScript Language Core

## 1. Knowledge Level: Solid

My knowledge of TypeScript's type system is thorough: generics (including constrained, conditional, variadic tuple), mapped types (key remapping via `as`, template literal types), conditional types with `infer`, utility types (`Pick`, `Omit`, `Partial`, `Required`, `Record`, `Exclude`, `Extract`, `NonNullable`, `ReturnType`, `Parameters`, `Awaited`, `InstanceType`), the `satisfies` operator, type and control-flow narrowing, and `.d.ts` declaration file authoring.

## 2. Versions Covered

Confident through TypeScript 5.4.x (early 2024). My reliable knowledge ends around mid-2024. Features stabilized after that window — such as `isolatedDeclarations` (5.5), inferred type predicates (5.5), `--noUncheckedIndexedAccess` refinements, and any new utility types or control-flow improvements in 5.6+ — are gaps.

## 3. Identified Gaps

- **`isolatedDeclarations`** (TS 5.5+): The exact semantics, limitations, and interaction with `declaration: true` and `composite` are uncertain. I know it enables per-file DTS emit without full-program context, but I don't know the edge cases or current ecosystem adoption.
- **Inferred type predicates** (TS 5.5+): I know the feature exists but don't know the precise rules about when inference triggers and when it doesn't.
- **`satisfies` with `as const` interplay**: Solid on the basics, but edge cases around `satisfies` narrowing in conditional branches may have evolved.
- **`using` declarations (explicit resource management)**: Stage 3 proposal; I don't know if/when it landed in TS stable and under what `lib` target.
- **Decorators (Stage 3 / TC39)**: I know the new decorator spec shipped in TS 5.0, but I don't have deep knowledge of edge cases vs the legacy experimental decorators — particularly around metadata, `accessor`, and auto-accessor behavior.
- **`noUncheckedIndexedAccess` edge cases**: I know the flag exists; I don't know the latest refinements (e.g., narrowing after `in` checks, `hasOwnProperty` guards).

## 4. Hallucination Risk

Highest risk areas:
- Claiming a TS feature "doesn't exist yet" when it shipped in 5.5–5.7.
- Inventing syntax for `isolatedDeclarations` configuration.
- Confidently explaining inferred type predicate rules that may be wrong.
- Mixing up legacy and TC39 decorator semantics.

## 5. Confidence: 7/10

Strong on the type system fundamentals, but the 5.5+ feature set and its interactions with the architecture (especially declaration emit in a monorepo) are uncertain enough to dock 3 points.

## Intersection Notes

- **D2 (project references)**: `isolatedDeclarations` directly impacts how project references and composite builds work. If a shared package uses `isolatedDeclarations`, I need to know whether `composite` is still required, how `declarationMap` interacts, and whether Turborepo caching is affected.
- **D3 (monorepo)**: Shared types packages rely on DTS emit. Any change in declaration emit behavior ripples across the entire `/packages/` layer.

## Recommended Phase 2 Search Queries

1. `TypeScript 5.5 5.6 5.7 release notes new features`
2. `TypeScript isolatedDeclarations composite project references`
3. `TypeScript inferred type predicates rules limitations`
4. `TypeScript satisfies operator edge cases narrowing`
5. `TypeScript 5.x decorators TC39 vs experimental differences 2025`
