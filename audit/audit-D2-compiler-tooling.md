# Audit D2 — Compiler & Tooling

## 1. Knowledge Level: Solid (with version-sensitive gaps)

I understand tsconfig deeply: `base` + `extends` patterns, `compilerOptions`, `include`/`exclude`, `references` for project references, `composite`, `declarationMap`, `paths`, `rootDir`/`outDir`, `moduleResolution` (`node`, `node16`, `nodenext`, `bundler`). I know tsup, esbuild, and swc as build tools and their trade-offs. My knowledge of their exact current APIs and flag names is potentially stale.

## 2. Versions Covered

- **tsup**: Confident through ~8.x (early 2024). The move toward 9.x may have changed configuration defaults or plugin APIs.
- **esbuild**: The core API is stable; confident through ~0.20.x.
- **swc**: Confident through the 2023–early 2024 era. The Rust-based SWC configuration landscape evolves quickly.
- **tsconfig/moduleResolution**: Confident through the `bundler` resolution introduction. Any newer mode would be a gap.

## 3. Identified Gaps

- **tsup 9.x**: I don't know if the configuration format changed, if `esbuild` options passthrough changed, or if DTS generation behavior (`dts: true` with `tsc` vs `isolatedDeclarations`) changed.
- **`moduleResolution: "bundler"` edge cases**: I know the idea (modeled after how bundlers resolve). I don't know the latest edge cases with `package.json` `exports` field resolution, or how it interacts with `module: "preserve"`.
- **`module: "preserve"`**: I know it exists in TypeScript 5.4+ but don't know its exact interaction with different `moduleResolution` values and build tools.
- **Project references + incremental builds**: I know `composite` + `incremental` + `.tsbuildinfo` — but I don't know if the caching model changed in recent TS versions, or if Turborepo's caching obviates or conflicts with `.tsbuildinfo`.
- **esbuild plugins for tsconfig paths**: I know `tsconfig-paths` plugin exists for esbuild but don't know its current maintenance status or alternatives.
- **swc config format**: The `.swcrc` schema may have evolved; I'm uncertain about recent additions.
- **tsx vs ts-node vs tsimp**: I know the tools but not their current relative performance/feature standing as of mid-2025.

## 4. Hallucination Risk

Highest risk areas:
- Recommending a tsup config flag that was renamed or removed in tsup 9.x.
- Explaining `moduleResolution` interactions that have been refined in TS 5.5+.
- Suggesting `.tsbuildinfo` patterns that conflict with Turborepo caching advice.
- Claiming esbuild plugin APIs that have changed.

## 5. Confidence: 6/10

The tools evolve independently and quickly. Even if I know the concepts well, recommending the exact current config syntax without verification is risky.

## Intersection Notes

- **D3 (Turborepo)**: The build tool choice (tsup/esbuild/swc) must integrate with Turborepo's `dependsOn` and caching. If tsup 9.x changed its output format or `--clean` behavior, Turbo cache keys could silently mismatch.
- **D1 (declaration emit)**: If `isolatedDeclarations` becomes the standard, build tools need to support it. I don't know which of tsup/esbuild/swc support it natively.
- **S2 (Docker)**: Multi-stage Docker builds need to know which build tool produces which output, and where.

## Recommended Phase 2 Search Queries

1. `tsup 9.x changelog configuration changes 2025`
2. `TypeScript moduleResolution bundler preserve interactions 2025`
3. `tsup esbuild isolatedDeclarations support 2025`
4. `esbuild tsconfig-paths plugin alternatives 2025`
5. `TypeScript project references tsbuildinfo vs Turborepo caching 2025`
6. `swc configuration 2025 latest`
