# Skill: TypeScript — D2: Compiler and Tooling

## Summary

Covers tsconfig architecture for multi-app monorepos, project references, build tools (tsc, tsup, esbuild, swc), and path alias resolution end-to-end. Knowledge cutoff: 2026-05-25. Audit risk: **High** — all 5 flagged gaps resolved during research.

**D1 dependency**: Assumes D1 (TypeScript Language Core) is complete. tsconfig flags related to the type system (`strict`, `noUncheckedIndexedAccess`, etc.) are documented there.

**Critical finding**: tsup has been **deprecated**. Latest version is 8.5.1 (no 9.x exists). Official recommendation is to migrate to tsdown (Rolldown-based). This document covers both: tsup 8.x for existing projects, tsdown for new projects.

## Version landscape

| Tool | Version | Status | Notes |
|------|---------|--------|-------|
| **TypeScript** | 5.9.2 (stable, Jul 2025) | Active | 6.0 is next (no 5.10). 7.0 will be native Go port. |
| **tsup** | 8.5.1 (Nov 2025) | **Deprecated** | No 9.x exists. Migrate to tsdown. |
| **tsdown** | Current (Rolldown-based) | Recommended successor | Drop-in replacement with migration CLI. |
| **esbuild** | 0.27.x (stable) / 0.28.0 (Apr 2026) | Active | 0.27.0 had breaking changes (Go 1.25). |
| **@swc/core** | 1.15.x | Active | Rust-based. Config format stable. |

**Minimum baseline**: TypeScript 5.5+. tsconfig flags removed or changed before 5.5 are not documented.

**Breaking changes to be aware of**:
- esbuild 0.27.0: Updated Go compiler, potential behavior changes.
- tsup → tsdown migration: format defaults changed (`cjs` → `esm`), `clean` defaulted to `true`, `dts` auto-detected.
- TypeScript 5.6: `tsc --build` no longer stops on first error (use `--stopOnBuildErrors` to restore old behavior).
- TypeScript 5.6: `.tsbuildinfo` always emitted during `--build`, even without `--incremental`.

## tsconfig architecture for multi-app monorepo

### Layer model

```
tsconfig.base.json          ← Root: shared compiler defaults
  ├── apps/backend/tsconfig.json
  ├── apps/frontend/tsconfig.json
  ├── apps/workers/tsconfig.json
  ├── packages/shared-types/tsconfig.json
  └── packages/utils/tsconfig.json
```

### Decision table: what goes in base vs per-app

| Category | Base (`tsconfig.base.json`) | Per-app (`tsconfig.json`) |
|----------|---------------------------|--------------------------|
| `strict`, `target`, `lib` | Default values | Override if needed |
| `module`, `moduleResolution` | **Do NOT set in base** | Each app sets its own |
| `paths`, `baseUrl` | Define here (single source of truth) | Extends automatically |
| `outDir`, `rootDir` | Do NOT set in base | Each app sets its own |
| `declaration`, `declarationMap`, `sourceMap` | Enable by default | Disable if not needed |
| `composite` | Do NOT set in base | Set per-package if using project references |
| `references` | N/A | Set per-package |
| `include`, `exclude` | Reasonable defaults | Refine per-app |
| `noEmit` | Do NOT set | Frontend sets `true` (bundler emits), backend sets `false` |

**Rule**: If a flag depends on the runtime environment (Node vs browser) or the build pipeline (bundler vs tsc emit), it belongs in the per-app config, not the base.

### moduleResolution choice per app type

| App/package | `moduleResolution` | `module` | Why |
|-------------|-------------------|----------|-----|
| `/apps/frontend` (Next.js) | `bundler` | `esnext` | Bundler handles resolution. Extensions not needed. |
| `/apps/backend` (Node, Fastify) | `nodenext` | `nodenext` | Emits for Node.js runtime. Extensions required in imports. |
| `/apps/workers` (Node) | `nodenext` | `nodenext` | Same as backend. |
| `/packages/*` (shared libraries) | `nodenext` | `nodenext` | **Must** work for both bundler and Node consumers. `nodenext` `.d.ts` output is portable; `bundler` `.d.ts` output is not. |

**The `/packages/*` trap**: If a shared package uses `moduleResolution: "bundler"`, its emitted `.d.ts` files will contain extensionless imports. Consumers using `nodenext` (backend, workers) will get compile errors. `nodenext` is the safe common denominator — code that works in Node.js works in bundlers, but not vice versa.

### `module` and `target` interaction

Certain `module` values imply `target`:

| `module` | Implied `target` | When to use |
|----------|-----------------|-------------|
| `node16` | `es2022` | Node 16 (EOL, avoid) |
| `node18` | `es2022` | Node 18 |
| `node20` | `es2023` | Node 20 |
| `nodenext` | `esnext` (floating) | Node 22+, keeps up with latest |
| `esnext` | No implication | Use with `bundler` resolution |
| `preserve` | No implication | Leave imports untouched for bundler |

For the monorepo architecture in this document, `/apps/backend` and `/apps/workers` target Node 22+, so `"module": "nodenext"` with its implied `esnext` target is idiomatic. `/apps/frontend` uses `"module": "esnext"` (Next.js manages the output).

### `verbatimModuleSyntax`

Enabled by default in the base config for all apps and packages. It:
- Forces `import type` / `export type` for type-only imports (they are erased, not emitted).
- Prevents module elision — the compiler won't silently drop imports it thinks are "unused."
- Ensures esbuild/swc/babel see the same module graph as tsc.
- Implies `isolatedModules` (redundant to set both, but allowed since TS 5.0.4).

### `isolatedDeclarations`

Enabled only in `/packages/*` that produce `.d.ts` output. It enforces:
- Every exported function must have an explicit return type annotation.
- Every exported variable/constant with a complex type must be explicitly typed.
- No inference across file boundaries.

This is **not** enabled in `/apps/*` because apps don't emit declarations. The value is in making declaration generation parallelizable — tools like oxc and `transpileDeclaration()` can generate `.d.ts` per-file without a full type checker. However, current tsup/tsdown don't yet leverage this for speed gains (they use rollup-plugin-dts or tsc + API Extractor).

### `erasableSyntaxOnly` (TS 5.8+)

Not set in base — enable per-app if targeting Node's native type stripping:
- Disallows `enum`, `namespace` with runtime code, parameter properties, `import =` aliases.
- Pairs with `verbatimModuleSyntax` for full erasable-syntax compliance.
- Relevant only if you plan to use Node's `--experimental-strip-types` or avoid a compile step.

### strict mode — what `"strict": true` enables

All 9 flags below are enabled by `"strict": true`. You can selectively disable any.

| Flag | What it catches |
|------|----------------|
| `noImplicitAny` | Parameters/variables with inferred `any` |
| `strictNullChecks` | `null`/`undefined` not assignable to every type |
| `strictFunctionTypes` | Unsound callback parameter bivariance |
| `strictBindCallApply` | Wrong argument types to `.bind/.call/.apply` |
| `strictPropertyInitialization` | Uninitialized class properties |
| `noImplicitThis` | `this` with implicit `any` type |
| `useUnknownInCatchVariables` | `catch` variables are `unknown`, not `any` |
| `alwaysStrict` | Emits `"use strict"` for each file |
| `strictBuiltinIteratorReturn` | (TS 5.0+) Stricter iterator `return()` checking |

Set `"strict": true` in `tsconfig.base.json` and never disable individual flags unless you have a specific, documented reason.

## Project references

### When to use them

Project references are appropriate when:
- You have packages that depend on other packages, and you want `tsc --build` to compile them in order.
- You want editor "Go to Definition" to navigate to source (`.ts`) rather than `.d.ts` files.

**Caveat**: Turborepo recommends **against** project references (see Antipatterns section). If you're using Turborepo for task orchestration (→ D3), its caching and dependency graph may make project references redundant. Decide before wiring them up — they add configuration that's hard to remove later.

### How to configure

**Packages that are referenced by others** (e.g., `packages/shared-types`):

```jsonc
{
  "compilerOptions": {
    "composite": true,            // Required for project references
    "declaration": true,          // Implied by composite
    "declarationMap": true,       // Enables source navigation for consumers
    "rootDir": "src",
    "outDir": "dist"
    // moduleResolution, module, etc. as normal
  }
}
```

**Packages that reference others** (e.g., `apps/backend`):

```jsonc
{
  "compilerOptions": {
    "composite": true,
    "rootDir": "src",
    "outDir": "dist"
  },
  "references": [
    { "path": "../../packages/shared-types" },
    { "path": "../../packages/utils" }
  ]
}
```

### `tsc --build` behavior

- `tsc -b` at the root (or `tsc -b tsconfig.json`) compiles all referenced projects in dependency order.
- `tsc -b --clean` removes all outputs including `.tsbuildinfo`.
- `tsc -b --force` rebuilds all projects regardless of whether they changed.
- **TS 5.6+**: `--build` continues past errors in upstream projects. Use `--stopOnBuildErrors` in CI for the old behavior.
- **TS 5.6+**: `.tsbuildinfo` is always written during `--build`, even without `--incremental` flag.

### declarationMap

`"declarationMap": true` generates `.d.ts.map` files that map declaration files back to source. This enables:
- "Go to Definition" in consumers navigates to `.ts` source, not `.d.ts`.
- Critical for monorepo developer experience — without it, clicking into a type from another package lands in a `.d.ts` file.

Set it in `tsconfig.base.json` alongside `"declaration": true` and `"sourceMap": true`.

### Known gotchas

1. **`composite` requires `rootDir`**: If `rootDir` is inferred and includes test files or `node_modules`, the build fails. Always set `rootDir` explicitly in composite projects.

2. **No cyclic references**: Project references form a DAG. Circular dependencies between packages must be resolved at the code level first.

3. **`.tsbuildinfo` is not portable**: File paths are absolute and timestamps differ across machines. Don't cache `.tsbuildinfo` in CI or Turborepo remote cache — it will produce wrong results.

4. **Declaration-only changes don't trigger rebuild**: If you change only types (no runtime code), `tsc -b` may consider downstream projects unchanged. This is usually fine because declarations are consumed at compile time, but can surprise if you expected a rebuild.

5. **Turborepo conflict**: Turborepo uses hard links for caching. `tsc --build` writes to existing output files in-place through those hard links, corrupting the cache. If using both, exclude `.tsbuildinfo` from Turborepo's outputs.

## Build tools

### tsc (TypeScript compiler)

**When to use directly**:
- **Type checking**: Always. Every other tool delegates to `tsc --noEmit` for type checking.
- **Emitting declarations**: When you don't use a bundler, `tsc --emitDeclarationOnly` generates `.d.ts` files.
- **Simple Node.js packages**: If you don't need bundling, `tsc` alone can compile and emit.

**When NOT to use for emit**:
- Apps that are bundled (frontend): Vite/Next.js handle JS emit.
- Packages that need multiple formats (CJS + ESM): `tsc` only emits one format per tsconfig.

**Recommendation**: Use `tsc --noEmit` as the type-checking step in all projects (CI, pre-commit). Delegate JS emit to a bundler.

### tsup 8.x (deprecated)

**Status**: Deprecated as of late 2025. No 9.x exists. The official recommendation is to migrate to **tsdown**. This section documents tsup 8.x for existing projects.

**Purpose**: Zero-config TypeScript bundler powered by esbuild. Produces CJS, ESM, and IIFE bundles with built-in DTS generation.

**Install**: `npm install -D tsup` (v8.5.1)

**Config format** (`tsup.config.ts`):

```ts
import { defineConfig } from 'tsup';

export default defineConfig({
  entry: ['src/index.ts'],
  outDir: 'dist',
  format: ['cjs', 'esm'],
  target: 'es2022',
  dts: true,
  sourcemap: true,
  clean: true,
  splitting: true,          // ESM code splitting
  external: [],             // Dependencies to leave as external imports
  platform: 'node',
  minify: false,

  // esbuild passthrough: mutate options per format
  esbuildOptions: (options, { format }) => {
    if (format === 'esm') {
      options.banner = { js: '// ESM build' };
    }
    options.drop = ['console'];
  },

  // esbuild plugins
  esbuildPlugins: [],

  onSuccess: async () => {
    console.log('Build complete');
  },
});
```

**DTS generation** — tsup has two mutually exclusive strategies:

| Strategy | Flag | Engine | Notes |
|----------|------|--------|-------|
| Stable | `dts: true` | `rollup-plugin-dts` | Good for most cases. Runs in worker thread. Supports watch mode. |
| Experimental | `dts: { experimentalDts: true }` | `tsc` + `@microsoft/api-extractor` | Handles declaration merging, conditional types, ambient modules. No watch mode. |

Neither strategy leverages `isolatedDeclarations` or `transpileDeclaration()` for speed. Both run the full type checker.

**Key options reference**:

| Option | Type | Default | Notes |
|--------|------|---------|-------|
| `entry` | `string \| string[] \| Record<string, string>` | — | Entry point(s) |
| `outDir` | `string` | `dist` | Output directory |
| `format` | `('cjs' \| 'esm' \| 'iife')[]` | `['cjs']` | Output formats |
| `target` | `string` | `es2017` | esbuild target (e.g., `es2022`, `node18`) |
| `platform` | `'node' \| 'browser' \| 'neutral'` | `node` | Platform preset |
| `dts` | `boolean \| DTSOptions` | `false` | Generate `.d.ts` |
| `sourcemap` | `boolean \| 'inline'` | `false` | Source maps |
| `clean` | `boolean` | `false` | Clean outDir before build |
| `splitting` | `boolean` | `false` | ESM code splitting |
| `minify` | `boolean` | `false` | Minify output |
| `treeshake` | `'recommended' \| 'smallest' \| 'safest'` | — | Rollup tree shaking |
| `external` | `string[]` | `[]` | Packages to externalize |
| `bundle` | `boolean` | `true` | Bundle dependencies |
| `shims` | `boolean` | `false` | Inject CJS/ESM interop shims |
| `esbuildOptions` | `(opts, ctx) => void` | — | Mutate esbuild options per format |
| `esbuildPlugins` | `esbuild.Plugin[]` | `[]` | Raw esbuild plugins |
| `onSuccess` | `() => Promise<void>` | — | Hook after successful build |

**Path aliases**: tsup does **not** resolve `tsconfig.json` `paths` automatically. With `bundle: true` (the default), esbuild handles internal imports but won't resolve `@shared/*` aliases. Options:
1. Use `esbuild-plugin-tsconfig-paths` plugin.
2. Use `esbuildPlugins` with a custom alias resolver.
3. Switch to tsdown (which has native tsconfig paths support).

### tsdown (tsup successor)

**Install**: `npm install -D tsdown`

**Migration**: `npx tsdown-migrate` (automated). Manual config rename: `tsup.config.ts` → `tsdown.config.ts`.

**Key differences from tsup**:

| Aspect | tsup | tsdown |
|--------|------|--------|
| Engine | esbuild | Rolldown (Rust) |
| `format` default | `cjs` | `esm` |
| `clean` default | `false` | `true` |
| DTS | `dts: true` | Auto-enabled if `types` in package.json |
| `target` default | `es2017` | Auto-reads from `engines.node` |
| tsconfig paths | Manual plugins needed | Native support |
| `esbuildOptions` | Callback | N/A (Rolldown API) |
| Config file | `tsup.config.ts` | `tsdown.config.ts` |

**Config example**:

```ts
import { defineConfig } from 'tsdown';

export default defineConfig({
  entry: ['src/index.ts'],
  format: ['esm'],          // default in tsdown
  dts: true,
  clean: true,              // default in tsdown
  sourcemap: true,
});
```

**Recommendation**: For new projects, start with tsdown. For existing tsup projects, migrate when the tsdown ecosystem matures (plugins, watch mode stability). The migration is one-command but plugin compatibility should be verified.

### esbuild (direct use)

**Current version**: 0.27.x stable (0.28.0 Apr 2026)

**Purpose**: Extremely fast bundler and minifier. Written in Go. No type checking.

**When to use directly** (not via tsup/tsdown):
- Custom build pipelines where you need full control over the esbuild plugin API.
- Simple scripts with no DTS generation needed.
- As a minifier step after another bundler.

**What esbuild does NOT do**:
- Type checking (must pair with `tsc --noEmit`).
- DTS generation (must use `tsc --emitDeclarationOnly` or tsup/tsdown).
- tsconfig `paths` resolution (needs a plugin like `esbuild-plugin-tsconfig-paths`).

**Config example** (`esbuild.config.mjs`):

```js
import * as esbuild from 'esbuild';
import { tsconfigPathsPlugin } from 'esbuild-plugin-tsconfig-paths';

await esbuild.build({
  entryPoints: ['src/index.ts'],
  bundle: true,
  platform: 'node',
  target: 'es2022',
  format: 'esm',
  outdir: 'dist',
  sourcemap: true,
  external: ['@shared/*'],  // Don't bundle monorepo packages
  plugins: [
    tsconfigPathsPlugin({
      cwd: process.cwd(),
      tsconfig: 'tsconfig.json',
    }),
  ],
});
```

**Recommendation**: Use tsup/tsdown for packages (they add DTS generation and multi-format output). Use esbuild directly only when you need custom plugins or the tsup abstraction gets in the way.

### swc (@swc/core)

**Current version**: 1.15.x

**Purpose**: Rust-based TypeScript/JavaScript compiler. Fast transpilation. Can replace tsc for JS emit.

**Use case vs esbuild**:
- **swc**: Better when you need fine-grained control over the output (e.g., specific ES target, decorator transforms). Good for libraries that need precise ES version output.
- **esbuild**: Better for bundling (tree shaking, code splitting, all deps into one file).

**Config** (`.swcrc`):

```jsonc
{
  "$schema": "https://swc.rs/schema.json",
  "jsc": {
    "parser": {
      "syntax": "typescript",
      "decorators": false,
      "dynamicImport": true
    },
    "target": "es2022",
    "externalHelpers": false,
    "transform": {}
  },
  "module": {
    "type": "es6"
    // "type": "commonjs" for CJS output
  },
  "sourceMaps": true,
  "isModule": true
}
```

**Using with tsup**: tsup can use swc for transpilation instead of esbuild via the `swcPlugin`:

```ts
import { defineConfig } from 'tsup';

export default defineConfig({
  // In tsup 8.5.0+, you can pass swc config:
  // swcPlugin passes custom config to @swc/core
});
```

**Recommendation**: Rarely needed directly. tsup/tsdown + esbuild handle most cases. Use swc directly when you need specific `jsc.target` or decorator transform behavior that esbuild doesn't support.

### Recommendation table: which tool for which app/package

| App/Package | Build tool | Type check | DTS emit | JS emit |
|-------------|-----------|------------|----------|---------|
| `/apps/backend` | tsdown or tsup | `tsc --noEmit` | Not needed (app) | tsdown/tsup → CJS or ESM |
| `/apps/workers` | tsdown or tsup | `tsc --noEmit` | Not needed (app) | tsdown/tsup → ESM |
| `/apps/frontend` | Next.js built-in (→ D8) | `tsc --noEmit` | Not needed (app) | Next.js (webpack/turbopack) |
| `/packages/shared-types` | tsdown or tsup | `tsc --noEmit` | tsdown/tsup `dts: true` | tsdown/tsup → CJS + ESM |
| `/packages/utils` | tsdown or tsup | `tsc --noEmit` | tsdown/tsup `dts: true` | tsdown/tsup → CJS + ESM |

**CI pipeline for each package**:

```bash
# 1. Type check
tsc --noEmit

# 2. Build (for packages that need emit)
tsdown          # or: tsup

# 3. (No DTS step for apps — they don't emit declarations)
```

## Path aliases end-to-end

### Defining aliases (single source of truth)

Define paths once in `tsconfig.base.json`:

```jsonc
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@shared/*": ["packages/shared-types/src/*"],
      "@utils/*": ["packages/utils/src/*"]
    }
  }
}
```

All per-app tsconfigs inherit these paths via `extends`. TypeScript understands them for editor support and type checking.

### Making aliases work in each tool

| Tool | What happens | What you need |
|------|-------------|---------------|
| **tsc** (type check) | Native — reads `paths` from tsconfig | Nothing. Works out of the box. |
| **tsc** (emit) | Does NOT rewrite paths in output JS | `tsc-alias` as post-build step |
| **tsup** | esbuild doesn't read tsconfig paths | `esbuild-plugin-tsconfig-paths` plugin, or enable `bundle: true` |
| **tsdown** | Native Rolldown plugin | Works automatically |
| **esbuild** (direct) | No built-in path resolution | `esbuild-plugin-tsconfig-paths` |
| **swc** | May need manual `jsc.paths` | Pass `baseUrl` and `paths` in `.swcrc` |

### Runtime resolution: tsc-alias vs tsconfig-paths

When `tsc` emits JS, `@shared/types` stays as `@shared/types` in the output. Node.js can't resolve it. Two solutions:

**tsc-alias** (recommended — build-time):

```bash
npm install -D tsc-alias
```

```jsonc
// package.json
{
  "scripts": {
    "build": "tsc && tsc-alias -p tsconfig.json"
  }
}
```

Rewrites aliases to relative paths in the compiled JS. Zero runtime dependency. Portable output.

**tsconfig-paths** (runtime — good for dev):

```bash
node -r tsconfig-paths/register dist/index.js
```

Patches Node's module loader at startup to resolve aliases. Adds a runtime dependency. Not suitable for libraries — consumers won't have the alias resolution.

**Recommendation**: Use `tsc-alias` when `tsc` is the emitter. Use bundler-based resolution (tsup/tsdown/esbuild plugins) when a bundler emits the JS. Avoid `tsconfig-paths` in production.

### Node.js subpath imports (native alternative)

Node.js 20+ supports `imports` in `package.json`, understood natively by Node and TypeScript:

```jsonc
// package.json
{
  "imports": {
    "#utils/*": "./src/utils/*.js"
  }
}
```

```ts
// Uses # prefix (npm convention), no tsconfig paths needed
import { helper } from '#utils/helper';
```

This is the direction of the ecosystem — no build tool plugins needed. However, it requires the `#` prefix and paths must point to compiled output (`.js`), not source (`.ts`). For monorepo cross-package resolution (`@shared/*`), tsconfig `paths` + build tool plugins are still needed.

## Antipatterns and common mistakes

### 1. Wrong `moduleResolution` for the app type

```jsonc
// WRONG: shared package using bundler
// packages/shared-types/tsconfig.json
{ "compilerOptions": { "moduleResolution": "bundler" } }
```

The emitted `.d.ts` files will have extensionless imports. Backend (using `nodenext`) will fail to consume them.

```jsonc
// CORRECT: shared package using nodenext
{ "compilerOptions": { "moduleResolution": "nodenext" } }
```

### 2. Path aliases that work in VS Code but break at runtime

You get no red squigglies, `tsc --noEmit` passes — but `node dist/index.js` fails with "Cannot find module '@shared/types'".

**Why**: TypeScript understands `paths` for type checking. Runtime (Node.js) does not. The compiled JS still contains `require('@shared/types')`.

**Fix**: Use `tsc-alias` (if tsc emits) or an esbuild plugin (if tsup/esbuild emits) to rewrite aliases to relative paths at build time.

### 3. Forgetting `declarationMap`

Without it, "Go to Definition" from a consumer lands in a `.d.ts` file. Developer experience degrades quickly in monorepos.

```jsonc
// In tsconfig.base.json:
{
  "compilerOptions": {
    "declaration": true,
    "declarationMap": true,    // ← don't skip this
    "sourceMap": true
  }
}
```

### 4. Mixing `composite` and non-composite packages

If package A references package B, but B is not `composite: true`, TypeScript errors. All referenced packages must be composite. Non-composite packages can still use project references to depend on composite packages — the direction matters.

### 5. Using project references AND Turborepo caching without excluding `.tsbuildinfo`

Turborepo uses hard links. `tsc --build` mutates files through those hard links, corrupting caches. Either:
- Don't use project references (Turborepo's recommendation).
- Exclude `.tsbuildinfo` from Turborepo's task outputs: `"outputs": ["dist/**", "!dist/*.tsbuildinfo"]`.

### 6. Setting `module` or `moduleResolution` in the base tsconfig

These are runtime/bundler specific. A value that works for the frontend (`bundler`) breaks the backend's consumers of shared packages. Each app/package must set its own.

### 7. Using `tsc` for multi-format output

`tsc` emits one format per tsconfig. To produce CJS + ESM with tsc, you'd need two tsconfigs and two compilation passes. Use a bundler (tsup/tsdown) instead — they produce both formats in a single build.

## Audit-flagged gaps — resolution

### Gap 1: tsup 9.x config format → RESOLVED

**No 9.x exists.** tsup's last release was 8.5.1 (November 2025). The project has been deprecated in favor of **tsdown** (Rolldown-based). The deprecation notice is in the README. The `defineConfig` API, `esbuildOptions` callback, and `dts` options documented in the 8.x section above are current and final.

Source: [tsup GitHub repository](https://github.com/egoist/tsup) — official

### Gap 2: isolatedDeclarations + DTS generation → RESOLVED

**tsup does not natively leverage `isolatedDeclarations`**. Neither DTS strategy (`dts: true` via rollup-plugin-dts, nor `experimentalDts` via tsc + API Extractor) uses TypeScript's `transpileDeclaration()` API for parallel declaration generation. `isolatedDeclarations` in tsconfig is purely a constraint: it errors if your exports aren't explicitly annotated. It doesn't change how tsup/tsdown emit declarations.

For tsup's `dts: true`, DTS generation works independently of `isolatedDeclarations`. Having `isolatedDeclarations: true` in tsconfig doesn't break tsup's DTS output — it just means your source must have explicit return types on all exports.

Newer tools (tsdown, bunup, oxc) are being designed to leverage `isolatedDeclarations` natively. tsdown uses oxc under the hood and may gain this in a future release. As of 2026-05, this is still emerging.

Source: [TypeScript 5.5 release notes](https://devblogs.microsoft.com/typescript/announcing-typescript-5-5/) — official

### Gap 3: esbuild passthrough options in tsup → RESOLVED

**The `esbuildOptions` callback is confirmed and unchanged in tsup 8.x.** Signature:

```ts
esbuildOptions?: (options: BuildOptions, context: { format: Format }) => void;
```

It receives esbuild's `BuildOptions` object (mutable) and the current format. Called per format in multi-format builds. Applied via an internal esbuild plugin named `modify-options`.

The companion `esbuildPlugins` option accepts raw esbuild `Plugin[]` for plugin lifecycle hooks.

Source: [tsup source code](https://github.com/egoist/tsup/blob/main/src/esbuild/index.ts) — official

### Gap 4: moduleResolution "bundler" — when to use it vs nodenext → RESOLVED

**Rule**: `bundler` for end-apps that ship via a bundler (Next.js, Vite). `nodenext` for anything that emits code consumed by Node.js directly, including shared packages.

**The critical edge case for this monorepo architecture**: `/packages/*` are consumed by both the frontend (bundler) and backend/workers (Node). If the package emits declarations using `bundler` resolution, the `.d.ts` files will contain extensionless imports, which break `nodenext` consumers. **Use `nodenext` for all shared packages** — it produces portable declarations that work for both bundler and Node consumers.

The `"node"` export condition is disabled under `bundler` resolution. If a dependency uses it, add `"customConditions": ["node"]` to restore it.

Source: [TypeScript moduleResolution docs](https://www.typescriptlang.org/tsconfig/moduleResolution) — official; [TS#62490](https://github.com/microsoft/TypeScript/issues/62490) — official issue

### Gap 5: verbatimModuleSyntax interaction with isolatedDeclarations → RESOLVED

**Complementary, not redundant.** They address different problems:
- `verbatimModuleSyntax`: Module system correctness. Ensures `import type` is used for type-only imports, prevents import elision, and guarantees cross-tool consistency (esbuild/swc/babel see the same imports as tsc).
- `isolatedDeclarations`: Declaration emit parallelism. Ensures every export has an explicit type so `.d.ts` can be generated per-file without a full type checker.

Together with `erasableSyntaxOnly` (TS 5.8+), they form the TypeScript team's recommended baseline for modern library projects. `verbatimModuleSyntax` implies `isolatedModules`, so setting both was historically an error (fixed in TS 5.0.4 to allow explicit `isolatedModules: true` alongside it, for Next.js compatibility).

Source: [TypeScript 5.0 release notes](https://devblogs.microsoft.com/typescript/announcing-typescript-5-0/) (verbatimModuleSyntax); [TypeScript 5.5 release notes](https://devblogs.microsoft.com/typescript/announcing-typescript-5-5/) (isolatedDeclarations) — official

## Reference snippets

### tsconfig.base.json

```jsonc
{
  "compilerOptions": {
    // Strict
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,

    // Module safety
    "verbatimModuleSyntax": true,
    "isolatedModules": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,

    // Emit
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "incremental": true,

    // Path aliases — single source of truth
    "baseUrl": ".",
    "paths": {
      "@shared/*": ["packages/shared-types/src/*"],
      "@utils/*": ["packages/utils/src/*"]
    }
  },
  "exclude": ["node_modules", "dist", ".turbo"]
}
```

### tsconfig.json — /apps/backend (Node, Fastify)

```jsonc
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "outDir": "dist",
    "rootDir": "src"
  },
  "include": ["src"],
  "references": [
    { "path": "../../packages/shared-types" },
    { "path": "../../packages/utils" }
  ]
}
```

### tsconfig.json — /apps/frontend (Next.js)

```jsonc
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "module": "esnext",
    "moduleResolution": "bundler",
    "noEmit": true,
    "allowImportingTsExtensions": true,
    "jsx": "preserve",
    "lib": ["dom", "dom.iterable", "esnext"]
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx"],
  "exclude": ["node_modules"]
}
```

### tsconfig.json — /apps/workers

```jsonc
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "outDir": "dist",
    "rootDir": "src"
  },
  "include": ["src"],
  "references": [
    { "path": "../../packages/shared-types" },
    { "path": "../../packages/utils" }
  ]
}
```

### tsconfig.json — /packages/shared-types

```jsonc
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "composite": true,
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "outDir": "dist",
    "rootDir": "src",
    "isolatedDeclarations": true
  },
  "include": ["src"]
}
```

### tsdown.config.ts (current format — recommended for new projects)

```ts
import { defineConfig } from 'tsdown';

export default defineConfig({
  entry: ['src/index.ts'],
  format: ['esm'],
  dts: true,
  clean: true,
  sourcemap: true,
});
```

### tsup.config.ts (8.x format — for existing projects)

```ts
import { defineConfig } from 'tsup';

export default defineConfig({
  entry: ['src/index.ts'],
  outDir: 'dist',
  format: ['cjs', 'esm'],
  target: 'es2022',
  dts: true,
  sourcemap: true,
  clean: true,
  platform: 'node',
  external: ['@shared/*', '@utils/*'],
  esbuildOptions: (options, { format }) => {
    if (format === 'esm') {
      options.banner = { js: '// ESM' };
    }
  },
});
```

## Sources used

- [TypeScript tsconfig options](https://www.typescriptlang.org/tsconfig/) — official
- [TypeScript project references handbook](https://www.typescriptlang.org/docs/handbook/project-references.html) — official
- [TypeScript 5.5 release notes — isolatedDeclarations](https://devblogs.microsoft.com/typescript/announcing-typescript-5-5/) — official
- [TypeScript 5.6 release notes — build mode changes](https://devblogs.microsoft.com/typescript/announcing-typescript-5-6/) — official
- [TypeScript 5.8 release notes — erasableSyntaxOnly](https://devblogs.microsoft.com/typescript/announcing-typescript-5-8/) — official
- [TypeScript 5.9 release notes — module node20](https://devblogs.microsoft.com/typescript/announcing-typescript-5-9/) — official
- [TypeScript Node Target Mapping wiki](https://github.com/microsoft/TypeScript/wiki/Node-Target-Mapping) — official
- [TypeScript issue #62490 — .d.ts portability bundler vs nodenext](https://github.com/microsoft/TypeScript/issues/62490) — official
- [TypeScript issue #61357 — node condition disabled under bundler](https://github.com/microsoft/TypeScript/issues/61357) — official
- [tsup GitHub repository](https://github.com/egoist/tsup) — official
- [tsdown migration guide](https://tsdown.dev/guide/migrate-from-tsup) — official
- [esbuild changelog 2025](https://github.com/evanw/esbuild/blob/main/CHANGELOG-2025.md) — official
- [esbuild API docs](https://esbuild.github.io/api/) — official
- [SWC configuration docs](https://swc.rs/docs/configuration/compilation) — official
- [SWC module configuration](https://swc.rs/docs/configuration/modules) — official
- [esbuild-plugin-tsconfig-paths](https://github.com/wjfei/esbuild-plugin-tsconfig-paths) — community
- [Turborepo TypeScript guide (recommends against project references)](https://turborepo.dev/guides/tools/typescript) — official
- [Turborepo issue #839 — hard links corrupt tsbuildinfo](https://github.com/vercel/turborepo/issues/839) — official
