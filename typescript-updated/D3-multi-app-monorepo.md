# Skill: TypeScript — D3: Multi-App Monorepo

## Summary

Covers pnpm workspaces and Turborepo 2.x for orchestrating a multi-app monorepo: `pnpm-workspace.yaml`, `workspace:*` protocol, `turbo.json` 2.x schema (`tasks`, not `pipeline`), task dependency graph, caching, persistent tasks, `turbo prune` for Docker, and workspace scripts. Knowledge cutoff: 2026-05-25. Audit risk: **TIER 1 CRITICAL** — all 6 flagged gaps resolved during research.

**Dependencies**: D1 (TypeScript Language Core) and D2 (Compiler and Tooling) completed. D1 covers the type system; D2 covers tsconfig architecture and project references. This document references D2's tsconfig layer model and project references where they intersect with the Turborepo task graph.

**Intersection with S2**: `turbo prune` is the bridge between this skill and the Containerization skill (S2). The prune output is the input to per-app Dockerfiles. This document covers the prune flow and the Dockerfile pattern. Full Docker config is in S2.

**Critical finding**: The audit assumed pnpm 9.x and Turborepo 1.x uncertainty. As of May 2026, pnpm is at **11.3.0** and Turborepo is at **2.9.14** (stable). The `pipeline` → `tasks` rename is the single most important schema change — any `turbo.json` using `pipeline` is 1.x syntax and will not work in 2.x. The `$ENV_VAR` prefix in `dependsOn` was deprecated in 1.5 (before 2.x), moved to the dedicated `env` key.

## Version landscape

| Tool | Version | Status | Notes |
|------|---------|--------|-------|
| **Turborepo** | 2.9.14 (May 2026) | Stable | No 3.x yet. 2.x only — `tasks` key, not `pipeline`. |
| **pnpm** | 11.3.0 (May 2026) | Stable | v11.x. v10 → v11 migration guide available. |
| **Node.js** | 24.x (Krypton, Active LTS) | Production target | 22.x is Maintenance LTS. 20.x EOL (Apr 2026). 26.x is Current (LTS Oct 2026). |

**Breaking changes from Turborepo 1.x to 2.x:**

| Change | 1.x | 2.x | Impact |
|--------|-----|-----|--------|
| **`pipeline` → `tasks`** | `"pipeline": {}` | `"tasks": {}` | **Errors** — 1.x syntax rejected |
| **`outputMode` → `outputLogs`** | `"outputMode": "full"` | `"outputLogs": "full"` | Renamed for clarity |
| **`globalDotEnv` / `dotEnv`** | Separate keys | Removed | Use `inputs` to include `.env` files |
| **`turbo` in `package.json`** | Supported | **Removed** | Use `turbo.json` exclusively |
| **Default cache location** | `node_modules/.cache` | `.turbo/cache` | Local cache path changed |
| **`--scope` / `--ignore`** | CLI flags | Removed | Use `--filter` exclusively |
| **Strict env mode** | Opt-in | **Default** | Only listed env vars available to tasks |
| **`$ENV_VAR` in `dependsOn`** | Deprecated in 1.5 | Hard error in 2.x | Use `env` key |

Automated migration: `npx @turbo/codemod migrate` (full) or `npx @turbo/codemod rename-pipeline` (pipeline rename only).

**pnpm 11.x notes**: `injectWorkspacePackages` (global setting for injected deps) was introduced in v10, stable in v11. `catalog:` protocol is stable. No fundamental workspace behavior changes from v10 to v11 beyond `trustLockfile` setting and memory optimizations for large workspaces.

## Repository structure

```
/
├── turbo.json              ← Turborepo 2.x task definitions (root)
├── package.json            ← Root: workspace scripts, devDependencies (turbo, pnpm)
├── pnpm-workspace.yaml     ← Workspace package globs, catalog definitions
├── pnpm-lock.yaml          ← Single lockfile for all packages
├── tsconfig.base.json      ← Shared TypeScript compiler defaults (D2)
├── .npmrc                  ← pnpm config: inject-workspace-packages, etc.
├── apps/
│   ├── frontend/
│   │   ├── package.json    ← name: "@app/frontend"
│   │   ├── tsconfig.json   ← extends ../../tsconfig.base.json
│   │   └── turbo.json      ← App-level overrides (optional)
│   ├── backend/
│   │   ├── package.json    ← name: "@app/backend"
│   │   ├── tsconfig.json
│   │   └── turbo.json
│   └── workers/
│       ├── package.json    ← name: "@app/workers"
│       ├── tsconfig.json
│       └── turbo.json
└── packages/
    ├── shared-types/
    │   ├── package.json    ← name: "@shared/types" (internal, not published)
    │   └── tsconfig.json   ← composite: true (project references)
    └── utils/
        ├── package.json    ← name: "@shared/utils"
        └── tsconfig.json
```

**What lives where:**

| File | Root | Per-app | Per-package |
|------|------|---------|-------------|
| `turbo.json` | Global task definitions | Overrides (extends root) | Overrides (extends root) |
| `package.json` | Workspace scripts, root devDeps | App deps + scripts | Package deps + scripts |
| `tsconfig.json` | `tsconfig.base.json` (shared defaults) | Extends base, app-specific | Extends base, composite |
| `Dockerfile` | — | `apps/<name>/Dockerfile` | — |
| `.npmrc` | Global pnpm config | — | — |

## pnpm workspaces

### pnpm-workspace.yaml

```yaml
# pnpm-workspace.yaml
packages:
  - "apps/*"
  - "packages/*"
```

No nesting, no exclusions needed for this architecture. pnpm discovers packages by matching `name` in their `package.json`.

### workspace:* protocol

Internal packages are declared as dependencies using the `workspace:` protocol. This guarantees pnpm resolves from the local workspace, never the registry.

```jsonc
// apps/backend/package.json
{
  "name": "@app/backend",
  "dependencies": {
    "@shared/types": "workspace:*",
    "@shared/utils": "workspace:*"
  }
}
```

**How it works:**
- `workspace:*` — always uses the exact local version (most common for apps depending on packages)
- `workspace:^` / `workspace:~` — semver range from local workspace version (useful when publishing)
- Bare `workspace:` — treated as `workspace:*`
- On `pnpm publish` / `pnpm pack`, `workspace:*` is dynamically replaced with the actual version from the workspace

**Configuration** (`.npmrc` or pnpm config):
- `save-workspace-protocol=true` — ensures `pnpm add` saves internal deps with `workspace:` prefix
- `link-workspace-packages=true` (default) — symlinks local packages into `node_modules`
- `prefer-workspace-packages=true` (default) — prefers local workspace versions over registry versions

### pnpm --filter patterns

Filtering restricts commands to subsets of the workspace:

```sh
# Run build in a specific package
pnpm --filter @app/backend build

# Run build in a package AND all its dependencies
pnpm --filter @app/backend... build

# Run build in a package AND all its dependents
pnpm --filter ...@shared/types build

# Run in all packages (recursive)
pnpm -r build

# Run only in packages changed since main
pnpm --filter "[origin/main]" build

# Run in packages under a directory glob
pnpm --filter "./packages/*" build

# Combine filters (matches any)
pnpm --filter @app/backend --filter @shared/types test
```

**Selector reference:**

| Selector | Scope |
|----------|-------|
| `<pkg>` | Exact package name |
| `<pkg>...` | Package + all its dependencies |
| `...<pkg>` | Package + all its dependents |
| `<pkgA>...<pkgB>` | All packages between A and B in the graph |
| `"./apps/*"` | Glob matching directory paths |
| `"[<branch>]"` | Packages changed since branch/commit |

### Root package.json scripts

The root `package.json` delegates to Turborepo, which orchestrates across all packages:

```jsonc
// package.json (workspace root)
{
  "name": "my-monorepo",
  "private": true,
  "packageManager": "pnpm@11.3.0",
  "scripts": {
    "build": "turbo run build",
    "dev": "turbo run dev",
    "test": "turbo run test",
    "lint": "turbo run lint",
    "check-types": "turbo run check-types",
    "clean": "turbo run clean"
  },
  "devDependencies": {
    "turbo": "^2.9.14"
  }
}
```

The `packageManager` field is **required** by Turborepo 2.x. It tells `turbo` which package manager to invoke. Without it, `turbo` will error.

## Turborepo 2.x — turbo.json

### Full annotated turbo.json for this architecture

```jsonc
// turbo.json (repo root)
{
  "tasks": {
    // ---- Build ----
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".next/**", "!.next/cache/**"],
      "inputs": ["src/**", "tsconfig.json"],
      "env": ["NODE_ENV"],
      "cache": true
    },

    // ---- Dev (persistent - never exits) ----
    "dev": {
      "dependsOn": ["^build"],
      "cache": false,
      "persistent": true,
      "inputs": ["src/**", "tsconfig.json"]
    },

    // ---- Test ----
    "test": {
      "dependsOn": ["build"],
      "outputs": ["coverage/**"],
      "inputs": ["src/**", "test/**", "tsconfig.json"],
      "env": ["CI", "NODE_ENV"],
      "cache": true
    },

    // ---- Lint ----
    "lint": {
      "dependsOn": ["^build"],
      "inputs": ["src/**", "tsconfig.json", "eslint.config.*"],
      "cache": true
    },

    // ---- Type checking ----
    "check-types": {
      "dependsOn": ["^check-types"],
      "inputs": ["src/**", "tsconfig.json"],
      "cache": true
    },

    // ---- Clean ----
    "clean": {
      "cache": false
    }
  },

  // ---- Global environment ----
  "globalEnv": ["CI", "NODE_ENV"],
  "globalPassThroughEnv": ["NPM_CONFIG_REGISTRY", "npm_config_registry"]
}
```

### Task definition fields

**`dependsOn`** — declares task ordering:

| Syntax | Meaning | Example |
|--------|---------|---------|
| `^task` | Runs that task in **upstream packages** first (topological) | `"^build"` — build all dependencies before this package |
| `task` (no prefix) | Runs that task in the **same package** first | `"test"` dependsOn `"build"` — build self before testing |
| `package#task` | Targets a specific package's task | `"@shared/types#build"` |

`dependsOn` builds a DAG. Turborepo parallelizes tasks within the same dependency level. Tasks with no dependencies run first, all in parallel.

**`outputs`** — glob patterns for cached artifacts:

```jsonc
"outputs": [
  "dist/**",           // catch-all build output
  ".next/**",           // Next.js build output
  "!.next/cache/**",   // exclude Next.js cache from cache
  "coverage/**"         // test coverage reports
]
```

Defaults to `[]`. Without `outputs`, Turborepo still caches the task (it knows it succeeded) but cannot restore any files — the task re-runs on cache restore if outputs are needed.

**`inputs`** — glob patterns that narrow the cache hash:

```jsonc
"inputs": ["src/**", "tsconfig.json"]
```

Defaults to all workspace files. Use `inputs` to restrict what triggers a rebuild. For example, a README change shouldn't invalidate the build cache.

**`cache`** — boolean, default `true`. Set `false` for long-running dev/watch tasks.

**`persistent`** — boolean, default `false`. Set `true` for tasks that never exit (dev servers, watch modes). Persistent tasks cannot be in another task's `dependsOn` — they never finish, so dependent tasks would wait forever.

**`env`** — list of env var names that affect the task's cache hash. Changing any listed var causes a cache miss:

```jsonc
"env": ["DATABASE_URL", "API_BASE_URL"]
```

**`passThroughEnv`** — env vars made available at runtime but do NOT affect the cache hash. Use for vars that can change without invalidating the build:

```jsonc
"passThroughEnv": ["PORT", "LOG_LEVEL"]
```

### Global configuration

**`globalEnv`** — env vars that affect ALL tasks' cache hashes:

```jsonc
"globalEnv": ["CI", "NODE_ENV"]
```

If `CI` changes from `""` to `"true"`, every task's cache is invalidated.

**`globalPassThroughEnv`** — env vars available to all tasks at runtime without affecting any cache hash.

**`globalDependencies`** — files whose change invalidates ALL task caches:

```jsonc
"globalDependencies": ["tsconfig.base.json", ".env*"]
```

### Strict environment mode (default in 2.x)

Only env vars explicitly listed in `env`, `globalEnv`, `passThroughEnv`, or `globalPassThroughEnv` are passed to task processes. Any unlisted env var is stripped before the task runs. This makes cache hashing deterministic.

During migration, disable temporarily with `--env-mode=loose`:

```sh
turbo run build --env-mode=loose
```

### Package-level turbo.json overrides

Individual packages can override or extend the root `turbo.json`:

```jsonc
// apps/backend/turbo.json
{
  "extends": ["//"],
  "tasks": {
    "build": {
      "outputs": ["dist/**", "build/**"]
    },
    "dev": {
      "persistent": true,
      "cache": false
    }
  }
}
```

- `"extends": ["//"]` — inherit from workspace root (`//` is the root sentinel)
- Scalar fields (`cache`, `persistent`): inherited from root, overridden if specified
- Array fields (`outputs`, `env`, `inputs`, `dependsOn`): **replace** root values by default
- `"$TURBO_EXTENDS$"` — microsyntax to **append** to inherited arrays instead of replacing:

```jsonc
{
  "extends": ["//"],
  "tasks": {
    "build": {
      "outputs": ["$TURBO_EXTENDS$", "build/**"]
    }
  }
}
```

### Running tasks

```sh
# Run build across all packages
turbo run build

# Run build only for a filtered set
turbo run build --filter=@app/backend

# Force rebuild (skip cache)
turbo run build --force

# Skip cache read/write
turbo run build --no-cache

# Continue on error (don't abort entire pipeline)
turbo run build --continue

# Dry run (see what would execute)
turbo run build --dry-run
```

### Remote cache

**Vercel managed cache** (free on all plans):
```sh
turbo login
turbo link
```
Or set env vars: `TURBO_TOKEN` + `TURBO_TEAM`.

**Self-hosted cache**: set `TURBO_API` pointing to your cache server, plus `TURBO_TOKEN` and `TURBO_TEAM`. Popular community servers: `turbo-remote-cache` (Node/S3), `turborepo-remote-cache` (Go/S3), `ducktape` (Rust).

**Cache-only flags:**
- `TURBO_REMOTE_ONLY=true` — skip local cache
- `TURBO_REMOTE_CACHE_READ_ONLY` — download only, don't upload

### turbo watch (2.0+)

Dependency-aware file watcher that re-runs tasks on changes:

```sh
turbo watch dev lint test
```

- Runs tasks respecting the `dependsOn` DAG
- Persistent tasks (`persistent: true`) are ignored — they keep running independently
- `futureFlags.watchUsingTaskInputs` — filters at the task level using each task's `inputs` globs
- `"interruptible": true` — for tools without built-in watchers, turbo watch restarts them on relevant changes

### with — sidecar tasks (2.5+)

Since persistent tasks can't use `dependsOn`, `with` ensures multiple persistent tasks run together:

```jsonc
{
  "tasks": {
    "dev": {
      "with": ["api#start"],
      "persistent": true,
      "cache": false
    }
  }
}
```

When `web#dev` runs, `api#start` also starts. This solves "frontend needs backend running locally."

### turbo gen (scaffolding)

`turbo gen` generates workspace packages from templates using a `turbo/generators` directory. For this architecture, it's optional — manual `package.json` creation in `apps/` or `packages/` is straightforward and doesn't warrant generator overhead for 5 packages.

## Task dependency graph

### How Turborepo resolves order

Given this package dependency graph:

```
apps/backend  ──depends on──▶ packages/shared-types
apps/backend  ──depends on──▶ packages/utils
apps/frontend ──depends on──▶ packages/shared-types
packages/utils ──depends on──▶ packages/shared-types
```

And this `turbo.json`:

```jsonc
"build": { "dependsOn": ["^build"] },
"test":  { "dependsOn": ["build"] }
```

Turborepo resolves the build DAG as:

```
Level 1 (no deps):        packages/shared-types#build
Level 2 (depends on L1):  packages/utils#build, apps/frontend#build
Level 3 (depends on L2):  apps/backend#build
```

Within each level, tasks run in parallel. `test` runs after `build` within each package (same-package dependency, no `^` prefix).

### Relationship to TypeScript project references (D2)

D2 documents TypeScript project references as the mechanism for `tsc --build` to resolve inter-package dependencies. Here's how they relate:

| Concern | Turborepo task graph | TS project references |
|---------|---------------------|----------------------|
| **What it orchestrates** | Task execution (build, test, lint) | TypeScript compilation order |
| **Scope** | Any task: tsc, esbuild, vitest, eslint | Only `tsc --build` |
| **Resolution** | npm/pnpm dependency graph | `"references"` in tsconfig |
| **Cache** | Task-level caching of outputs | `.tsbuildinfo` incremental compilation |

**Both are needed.** Turborepo doesn't know about TypeScript's internal compilation graph — it only sees package dependencies. Project references tell `tsc --build` the order to compile TypeScript files. Turborepo tells the build system the order to run all tasks.

**Practical interaction**: If `packages/shared-types` depends on nothing and `packages/utils` depends on `shared-types`, then:
1. Turborepo sees `utils` → `shared-types` in `package.json` and orders `shared-types#build` before `utils#build` via `"^build"`.
2. If both use `tsc --build`, project references ensure `utils` waits for `shared-types`'s `.d.ts` files.
3. The Turborepo ordering runs first (process-level). The TypeScript ordering runs inside each build task (compiler-level).

**Turborepo's position on project references**: Turborepo maintains that `tsc --build` is optional — a single `tsc --noEmit` across the repo is sometimes simpler. For this architecture, project references are recommended for `/packages/*` (documented in D2) because they enable `tsc --build --incremental` with `.tsbuildinfo` for fast iteration on shared types.

## turbo prune — Docker integration

### Why it matters

Without pruning, a Docker build copies the entire monorepo — every app, every package, every `node_modules`. This means:
- Docker layer cache invalidates on any file change anywhere in the repo
- `pnpm install` resolves all packages, not just the ones needed
- Build times are proportional to monorepo size, not app size

`turbo prune` creates a sparse subset of the monorepo containing only the target app and its dependency chain.

### Exact command

```sh
turbo prune @app/backend --docker
```

Flags:
- `--docker` — optimizes output for Docker layer caching (splits into `json/` and `full/`)
- `--out-dir <path>` — custom output directory (default: `out/`)

### What it produces

```
out/
├── json/
│   ├── package.json          ← pruned: only packages the target needs
│   ├── pnpm-lock.yaml        ← pruned: only entries for needed packages
│   ├── pnpm-workspace.yaml   ← unchanged
│   ├── turbo.json            ← unchanged
│   └── apps/
│   │   └── backend/
│   │       └── package.json
│   └── packages/
│       ├── shared-types/
│       │   └── package.json
│       └── utils/
│           └── package.json
└── full/
    ├── package.json           ← same as json/
    ├── pnpm-lock.yaml         ← same as json/
    ├── pnpm-workspace.yaml
    ├── turbo.json
    ├── .npmrc                 ← if exists
    ├── apps/
    │   └── backend/
    │       └── src/...        ← FULL source
    └── packages/
        ├── shared-types/
        │   └── src/...        ← FULL source
        └── utils/
            └── src/...        ← FULL source
```

**Key**: `json/` has only `package.json` files (for `pnpm install` layer caching). `full/` has the actual source code. The Dockerfile copies `json/` first, installs dependencies, then copies `full/` and builds.

### Multi-stage Dockerfile pattern (pnpm)

```dockerfile
# apps/backend/Dockerfile
FROM node:24-alpine AS base
RUN corepack enable && corepack prepare pnpm@11 --activate

# ---- Stage 1: Prune ----
FROM base AS pruner
WORKDIR /app
COPY . .
RUN pnpm dlx turbo prune @app/backend --docker

# ---- Stage 2: Install dependencies ----
FROM base AS installer
WORKDIR /app
COPY --from=pruner /app/out/json/ .
RUN pnpm install --frozen-lockfile

# ---- Stage 3: Build ----
FROM installer AS builder
COPY --from=pruner /app/out/full/ .
RUN pnpm turbo run build --filter=@app/backend...

# ---- Stage 4: Runtime ----
FROM node:24-alpine AS runner
WORKDIR /app
COPY --from=builder /app/apps/backend/dist ./dist
COPY --from=builder /app/apps/backend/package.json .
# Production deps only
COPY --from=builder /app/out/json/ .
RUN pnpm install --frozen-lockfile --prod
CMD ["node", "dist/index.js"]
```

Built from the monorepo root:
```sh
docker build -f apps/backend/Dockerfile -t backend .
```

**Layer caching rationale:**
1. `COPY . .` — base image + source rarely changes
2. `turbo prune` — runs on every change (fast, just file filtering)
3. `COPY out/json/` + `pnpm install` — **cached** unless `package.json`/lockfile changes
4. `COPY out/full/` + `turbo build` — runs when source changes

Without the split (`json/` vs `full/`), `pnpm install` would re-run on every source change, wasting minutes.

### Known issues with pnpm (as of May 2026)

- **Fixed**: `turbo prune --docker` producing broken `pnpm-lock.yaml` with missing resolutions (Issue #12002, Feb 2026). Fixed — use `--skip-infer` if still hitting this on older versions.
- **Open**: `bin` entries missing in `out/json/` folder (Issue #12610, Apr 2026). When a workspace package has a `"bin"` field and another package depends on that binary, `pnpm install` fails in the pruned context because no stub files exist. Workaround: add `--ignore-scripts` to `pnpm install` in the installer stage, or ensure `bin` packages are built before the prune (pre-build step).

## Workspace scripts

### Root package.json

```jsonc
// package.json (workspace root)
{
  "name": "my-monorepo",
  "private": true,
  "packageManager": "pnpm@11.3.0",
  "scripts": {
    "build": "turbo run build",
    "dev": "turbo run dev",
    "test": "turbo run test",
    "lint": "turbo run lint",
    "check-types": "turbo run check-types",
    "clean": "turbo run clean",
    "format": "prettier --write ."
  },
  "devDependencies": {
    "turbo": "^2.9.14",
    "prettier": "^3.5.0"
  }
}
```

### dev task setup

The `dev` task runs all app dev servers in parallel. Since they're persistent, they run concurrently:

```jsonc
"dev": {
  "dependsOn": ["^build"],
  "cache": false,
  "persistent": true
}
```

`turbo run dev` starts all three apps' dev servers simultaneously. Each app defines its own dev command:

```jsonc
// apps/backend/package.json
{ "scripts": { "dev": "tsx watch src/index.ts" } }

// apps/frontend/package.json
{ "scripts": { "dev": "next dev -p 3000" } }

// apps/workers/package.json
{ "scripts": { "dev": "tsx watch src/index.ts" } }
```

Use `turbo watch` for dev workflows where tasks should re-run on file changes:

```sh
turbo watch dev
```

Persistent tasks are ignored by `turbo watch` (they have their own HMR/watchers). Non-persistent tasks (lint, test) re-run on matching file changes.

## Antipatterns and common mistakes

1. **Using `pipeline` key in Turborepo 2.x** — the most impactful mistake. Turborepo 2.x will reject `pipeline` with an error. Always use `tasks`.

2. **Missing `outputs` on build tasks** — without `outputs`, Turborepo caches that the task "succeeded" but cannot restore the output files. Downstream tasks that need those files will fail on cache hit.

3. **Wrong `dependsOn` topology** — `"dependsOn": ["build"]` (same package, no `^`) means "build myself before testing myself." `"dependsOn": ["^build"]` (with `^`) means "build my dependencies before building myself." Getting this wrong silently breaks the build order.

4. **Forgetting `passThroughEnv` for runtime env vars** — if your app reads `PORT` or `LOG_LEVEL` at runtime but those are not in `passThroughEnv`, Turborepo strips them in strict mode (default in 2.x). The app crashes at runtime with `undefined`.

5. **Apps importing each other directly** — `/apps/backend` should never import from `/apps/frontend`. Apps are leaf nodes in the dependency graph. Shared code goes in `/packages/*`. If two apps need the same thing, extract it to a package.

6. **Declaring `workspace:*` deps without `save-workspace-protocol`** — running `pnpm add @shared/types` without the config flag saves `"@shared/types": "^1.0.0"` instead of `"workspace:*"`, which may resolve to a registry version instead of the local one.

7. **Putting `moduleResolution` in `tsconfig.base.json`** — this breaks the base config because different apps need different resolution strategies (frontend = `bundler`, backend = `nodenext`). Module resolution belongs in per-app tsconfig files (see D2).

8. **Not including `.npmrc` in the prune output** — if `.npmrc` has settings like `inject-workspace-packages=true` or custom registries, the Docker installer stage won't have it unless it's copied. The `full/` output includes `.npmrc` if it exists at the repo root.

## Audit-flagged gaps — resolution

### 1. pipeline → tasks migration

**Resolved.** `pipeline` renamed to `tasks` in Turborepo 2.0. Also: `outputMode` → `outputLogs`, `globalDotEnv`/`dotEnv` removed (use `inputs`), `turbo` field in `package.json` removed. Automated migration: `npx @turbo/codemod migrate`. Using `pipeline` in 2.x throws an error — it does not silently fail.

Sources: Web search results citing the official Turborepo upgrading guide at turborepo.dev/docs/crafting-your-repository/upgrading.

### 2. dependsOn syntax changes in 2.x

**Resolved.** `^task` and `task` syntax unchanged. `$ENV_VAR` prefix was deprecated in Turborepo **1.5** (not a 2.x change), moved to dedicated `env` key. In 2.x, using `$ENV_VAR` in `dependsOn` is a hard error. `env` is a per-task array of variable name strings. `globalEnv` serves the same purpose globally.

Sources: Web search results citing Turborepo docs on `env`/`globalEnv` configuration, and the `migrate-env-var-dependencies` codemod docs.

### 3. Cache configuration in 2.x

**Resolved.** No fundamental schema change beyond the `pipeline` → `tasks` rename. `outputs`, `inputs`, `cache`, `env` work the same way. The key change is **strict environment mode** being the default — only explicitly listed env vars are available. `globalEnv` and `globalPassThroughEnv` are the new top-level keys. Local cache location moved from `node_modules/.cache` to `.turbo/cache`.

Sources: Web search results citing turborepo.dev/docs/reference/configuration.

### 4. turbo prune --docker exact flow

**Resolved.** Command: `turbo prune <workspace> --docker`. Produces `out/json/` (package.json files + pruned lockfile) and `out/full/` (full source of needed packages). Multi-stage Dockerfile: COPY json → pnpm install → COPY full → turbo build. Verified for pnpm — the `pnpm-workspace.yaml` is included automatically.

**Open issue**: `bin` entries missing in `out/json/` with pnpm (Issue #12610, Apr 2026). Workaround documented above. This is a known gap to monitor.

Sources: Web search results citing turborepo.dev/guides/tools/docker and GitHub discussions #3346 (pnpm + Docker pattern).

### 5. Persistent tasks in Turborepo 2.x

**Resolved.** `"persistent": true` on the task definition. Persistent tasks cannot be in another task's `dependsOn`. `turbo watch` (2.0+) for dependency-aware re-running of non-persistent tasks. `"with"` (2.5+) for running multiple persistent tasks together without `dependsOn`.

Sources: Web search results citing turborepo.dev/docs/reference/watch and the Turborepo 2.5 blog post.

### 6. pnpm catalog feature

**Resolved.** `catalog:` protocol defines shared dependency versions in `pnpm-workspace.yaml`. Default catalog (`catalog:`) or named catalogs (`catalog:react18`). Referenced in `package.json` as `"react": "catalog:"`. On publish, `catalog:` is replaced with the actual version. **Does not conflict with `workspace:*`** — they serve different purposes:
- `workspace:*` — links internal packages (apps depending on `/packages/*`)
- `catalog:` — aligns external dependency versions across the workspace (all apps use the same `react` version)

Sources: Web search results citing pnpm.io/catalogs and RFC #0001.

## Reference snippets

### pnpm-workspace.yaml

```yaml
packages:
  - "apps/*"
  - "packages/*"

catalog:
  typescript: ^5.9.2
  fastify: ^5.3.0
  zod: ^4.0.0
```

### turbo.json (full, Turborepo 2.x)

```jsonc
{
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".next/**", "!.next/cache/**"],
      "inputs": ["src/**", "tsconfig.json"],
      "env": ["NODE_ENV"],
      "cache": true
    },
    "dev": {
      "dependsOn": ["^build"],
      "cache": false,
      "persistent": true,
      "inputs": ["src/**", "tsconfig.json"]
    },
    "test": {
      "dependsOn": ["build"],
      "outputs": ["coverage/**"],
      "inputs": ["src/**", "test/**"],
      "env": ["CI", "NODE_ENV"],
      "cache": true
    },
    "lint": {
      "dependsOn": ["^build"],
      "inputs": ["src/**", "tsconfig.json", "eslint.config.*"],
      "cache": true
    },
    "check-types": {
      "dependsOn": ["^check-types"],
      "inputs": ["src/**", "tsconfig.json"],
      "cache": true
    },
    "clean": {
      "cache": false
    }
  },
  "globalEnv": ["CI", "NODE_ENV"],
  "globalPassThroughEnv": ["NPM_CONFIG_REGISTRY", "npm_config_registry"]
}
```

### Root package.json scripts

```jsonc
{
  "name": "my-monorepo",
  "private": true,
  "packageManager": "pnpm@11.3.0",
  "scripts": {
    "build": "turbo run build",
    "dev": "turbo run dev",
    "test": "turbo run test",
    "lint": "turbo run lint",
    "check-types": "turbo run check-types",
    "clean": "turbo run clean",
    "format": "prettier --write ."
  },
  "devDependencies": {
    "turbo": "^2.9.14",
    "prettier": "^3.5.0"
  }
}
```

### Internal package declaration

```jsonc
// packages/shared-types/package.json
{
  "name": "@shared/types",
  "version": "0.0.0",
  "private": true,
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "default": "./dist/index.js"
    }
  },
  "scripts": {
    "build": "tsc --build",
    "dev": "tsc --build --watch",
    "clean": "rm -rf dist"
  },
  "devDependencies": {
    "typescript": "catalog:"
  }
}
```

### App depending on internal packages

```jsonc
// apps/backend/package.json
{
  "name": "@app/backend",
  "version": "0.0.0",
  "private": true,
  "scripts": {
    "build": "tsup src/index.ts",
    "dev": "tsx watch src/index.ts",
    "test": "vitest run",
    "lint": "eslint src/"
  },
  "dependencies": {
    "@shared/types": "workspace:*",
    "@shared/utils": "workspace:*",
    "fastify": "catalog:",
    "zod": "catalog:"
  },
  "devDependencies": {
    "tsup": "^8.5.0",
    "tsx": "^4.20.0",
    "vitest": "^3.1.0",
    "typescript": "catalog:"
  }
}
```

### turbo prune Dockerfile pattern (pnpm, multi-stage)

```dockerfile
FROM node:24-alpine AS base
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
COPY --from=builder /app/apps/backend/dist ./dist
COPY --from=builder /app/apps/backend/package.json .
COPY --from=builder /app/out/json/ .
RUN pnpm install --frozen-lockfile --prod
CMD ["node", "dist/index.js"]
```

## Sources used

- [Turborepo 2.x upgrading guide](https://turborepo.dev/docs/crafting-your-repository/upgrading) — official
- [Turborepo configuration reference](https://turborepo.dev/docs/reference/configuration) — official
- [Turborepo deploying with Docker](https://turborepo.dev/docs/handbook/deploying-with-docker) — official
- [Turborepo Docker guide](https://turborepo.dev/guides/tools/docker) — official
- [Turborepo watch reference](https://turborepo.dev/docs/reference/watch) — official
- [Turborepo prune reference](https://turborepo.dev/docs/reference/prune) — official
- [Turborepo 2.5 blog post (sidecar tasks)](https://turborepo.dev/blog/turbo-2-5) — official
- [Turborepo GitHub — v2.9.14 release](https://github.com/vercel/turbo/releases) — official
- [Turborepo issue #839 — hard links corrupt tsbuildinfo](https://github.com/vercel/turborepo/issues/839) — official
- [Turborepo issue #12002 — broken pnpm lockfile from prune](https://github.com/vercel/turborepo/issues/12002) — official
- [Turborepo issue #12610 — bin entries missing in prune](https://github.com/vercel/turborepo/issues/12610) — official
- [Turborepo discussion #3346 — pnpm Docker example](https://github.com/vercel/turborepo/discussions/3346) — community
- [pnpm workspaces documentation](https://pnpm.io/workspaces) — official
- [pnpm filtering documentation](https://pnpm.io/filtering) — official
- [pnpm catalog documentation](https://pnpm.io/catalogs) — official
- [pnpm workspace protocol documentation](https://pnpm.io/workspaces#workspace-protocol-workspace) — official
- [pnpm injected dependencies](https://pnpm.io/package_json#dependenciesmetainjected) — official
- [pnpm v11 migration guide](https://pnpm.io/11.x/migration) — official
- [Node.js release schedule](https://github.com/nodejs/Release/blob/main/README.md) — official
- [pnpm catalogs RFC #0001](https://github.com/pnpm/rfcs/blob/main/text/0001-catalogs.md) — official
