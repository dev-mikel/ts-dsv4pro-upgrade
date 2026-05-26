# Skill: TypeScript — D7: Linting and Formatting

## Summary

Covers ESLint flat config, typescript-eslint, Prettier integration, and architectural enforcement rules for a TypeScript monorepo. The lint layer is the last defense before a developer accidentally imports from one app into another or introduces a circular dependency in `@shared`. Knowledge cutoff: 2026-05-25. Audit risk: **TIER 2 HIGH** — all 6 flagged gaps resolved during research.

**Dependencies**: D1 (TypeScript Language Core), D2 (Compiler and Tooling), D3 (Multi-App Monorepo), D5 (Shared Types and Contracts) — all completed and verified.

**Critical version finding**: The audit assumed ESLint v9.x. As of May 2026, ESLint is at **v10.4.0** (released Feb 2026). ESLint v9.x is EOL August 2026. The legacy `.eslintrc` format is **fully removed** in v10 — flat config (`eslint.config.js`) is the only format. This document documents v10.x only.

## Version landscape

| Tool | Version | Status | Notes |
|------|---------|--------|-------|
| **ESLint** | 10.4.0 (May 2026) | Stable | v10 released Feb 2026. Flat config ONLY. `.eslintrc` fully removed. |
| **typescript-eslint** | 8.59.4 (May 2026) | Stable | Still v8.x major. No v9 yet. `tseslint.config()` deprecated in favor of `defineConfig()`. |
| **Prettier** | 3.8.3 (Apr 2026) | Stable | v4.0.0 in alpha. No stable v4 yet. |
| **eslint-config-prettier** | 9.1.x | Stable | Use `/flat` entry point for flat config. |
| **eslint-plugin-import-x** | 4.16.2 (Mar 2026) | Stable | **Use this, not `eslint-plugin-import`.** The original (`import-js/eslint-plugin-import`) v2.32.0 does not support ESLint v10. |
| **TypeScript** | 6.0.3 (May 2026) | D1 | typescript-eslint v8.58.0+ supports TS 6. |

**Why eslint-plugin-import-x instead of eslint-plugin-import:**

The original `eslint-plugin-import` (v2.32.0, June 2025) has flat config support via `flatConfigs` but has not shipped an ESLint v10-compatible release. ESLint v10 removed APIs (`context.parserPath`, `FileEnumerator`) that the plugin depends on. The maintainer has committed v10 support to `main` (Feb 2026) but no release has been published. The community has largely moved to `eslint-plugin-import-x`, which:
- Shipped ESLint v10 peer-dep support in v4.16.2 (March 2026)
- Uses a Rust-based resolver (`unrs-resolver`) for better performance and `exports` field support
- Has 16 dependencies (vs 117 for the original)
- Releases multiple times per month (not once per year)

Migration from `eslint-plugin-import` to `eslint-plugin-import-x` takes about 10 minutes: rename rule prefixes from `import/` to `import-x/`.

**`tseslint.config()` deprecation note:**

The `tseslint.config()` helper was deprecated in favor of ESLint's built-in `defineConfig()` from `'eslint/config'` (available since ESLint v9.22.0). However, `defineConfig()` has known type compatibility issues with `eslint-plugin-import-x`'s flat config objects. **Practical recommendation: continue using `tseslint.config()` for now** — it still works in typescript-eslint v8 and has better compatibility with third-party flat configs. When typescript-eslint ships a v9 major, `tseslint.config()` will be removed and the ecosystem type issues should be resolved.

## ESLint v10 flat config structure

### Config object anatomy

```js
// eslint.config.js
export default [
  // Each element is a config object:
  {
    name: 'my-config',                    // Optional: identifier for Config Inspector
    files: ['**/*.ts', '**/*.tsx'],       // Glob patterns — which files this config applies to
    ignores: ['**/dist/**'],              // Glob patterns — which files to skip (can also be top-level)
    languageOptions: {
      ecmaVersion: 'latest',
      sourceType: 'module',
      parser: tsParser,                   // Custom parser (e.g., @typescript-eslint/parser)
      parserOptions: {
        projectService: true,             // Type-aware linting (see below)
        tsconfigRootDir: import.meta.dirname,
      },
      globals: {                          // Global variables (replaces .eslintrc env)
        myGlobal: 'readonly',
      },
    },
    plugins: {
      '@typescript-eslint': tseslintPlugin,  // Plugin name → plugin object
      'import-x': importX,
    },
    rules: {
      'no-console': 'warn',
    },
    settings: {                           // Shared settings for rules
      'import-x/resolver': {
        typescript: true,
      },
    },
    linterOptions: {
      reportUnusedDisableDirectives: 'error',  // Warn on unnecessary // eslint-disable
    },
  },
];
```

### Global ignores (replaces .eslintignore)

```js
export default [
  // Global ignores — must be a standalone config object with ONLY `ignores`
  {
    ignores: [
      '**/node_modules/**',
      '**/dist/**',
      '**/.turbo/**',
      '**/.next/**',
      '**/build/**',
      '**/coverage/**',
      '**/*.min.js',
    ],
  },
  // ... other configs
];
```

Global ignores are a standalone config object containing only `ignores` (no other keys). They apply to all files regardless of `files` patterns in other configs. This replaces both `.eslintignore` and the old `ignorePatterns` in `.eslintrc`.

### Running ESLint

```bash
# Lint all files from CWD
npx eslint .

# Lint specific directory
npx eslint apps/backend/

# Lint with auto-fix
npx eslint . --fix

# Lint with cache (--cache always recommended in CI)
npx eslint . --cache --cache-location .eslintcache

# Inspect the resolved config (debugging)
npx eslint --inspect-config
```

ESLint v10 looks up `eslint.config.*` starting from the directory of each linted file, not CWD. This means different packages in a monorepo can have their own `eslint.config.js` and ESLint finds them automatically. For a root-level shared config, place `eslint.config.js` at the repo root.

## typescript-eslint setup

### Installation

```bash
pnpm add -D eslint @eslint/js typescript typescript-eslint
```

All four packages are required:
- `eslint` — the linter itself
- `@eslint/js` — provides `js.configs.recommended` (core ESLint recommended rules, flat config format)
- `typescript` — the TypeScript compiler (used by the parser for type-aware rules)
- `typescript-eslint` — provides `tseslint.config()`, all `configs.*`, and the TypeScript parser/plugin

### Basic setup with tseslint.config()

Despite the deprecation warning, `tseslint.config()` is the most compatible wrapper for the current ecosystem:

```js
// @ts-check
import js from '@eslint/js';
import tseslint from 'typescript-eslint';
import { importX } from 'eslint-plugin-import-x';
import eslintConfigPrettier from 'eslint-config-prettier/flat';

export default tseslint.config(
  // ── Global ignores ──────────────────────────────────────
  {
    ignores: [
      '**/node_modules/**',
      '**/dist/**',
      '**/.turbo/**',
      '**/.next/**',
      '**/build/**',
      '**/coverage/**',
    ],
  },

  // ── Base recommended configs ────────────────────────────
  js.configs.recommended,
  tseslint.configs.recommendedTypeChecked,
  tseslint.configs.stylisticTypeChecked,

  // ── Type-aware linting config ───────────────────────────
  {
    languageOptions: {
      parserOptions: {
        projectService: true,
        tsconfigRootDir: import.meta.dirname,
      },
    },
  },

  // ── Import rules ────────────────────────────────────────
  importX.flatConfigs.recommended,
  importX.flatConfigs.typescript,
  {
    settings: {
      'import-x/resolver': {
        typescript: true,               // Uses eslint-import-resolver-typescript
      },
    },
  },

  // ── Prettier — MUST be last ─────────────────────────────
  eslintConfigPrettier,
);
```

### Type-aware linting: projectService vs project array

`projectService: true` is the **current recommended default**. It uses the same TypeScript Project Service APIs as VS Code, auto-discovers the nearest `tsconfig.json` per file, and supports project references out of the box.

```js
// Recommended: projectService (simpler, editor-consistent)
languageOptions: {
  parserOptions: {
    projectService: true,
    tsconfigRootDir: import.meta.dirname,
  },
}
```

**However**, `projectService` has a known performance regression in large monorepos. The typescript-eslint team tracks this in [issue #9571](https://github.com/typescript-eslint/typescript-eslint/issues/9571) and their performance repo states bluntly: *"Right now, `parserOptions.project` with single-run inference outperforms `parserOptions.projectService`."* One user reported 3m25s vs 16m23s on a large monorepo.

For this project (3 apps, 2 packages), `projectService: true` is fine. If lint becomes slow as the repo grows, fall back to the explicit `project` array:

```js
// Fallback: explicit project array (faster for large monorepos)
languageOptions: {
  parserOptions: {
    project: [
      './apps/backend/tsconfig.json',
      './apps/frontend/tsconfig.json',
      './apps/workers/tsconfig.json',
      './packages/shared/tsconfig.json',
      './packages/utils/tsconfig.json',
    ],
    tsconfigRootDir: import.meta.dirname,
  },
}
```

Key notes:
- `projectService: true` replaces `project: true` (the heuristic variant). `project: true` may be deprecated.
- `project: string[]` (explicit array) is **not** being deprecated and remains the fallback.
- `allowDefaultProject` should be used sparingly — each matched file incurs overhead.
- For JS files, disable type-aware linting: `{ files: ['**/*.js'], extends: [tseslint.configs.disableTypeChecked] }`

### Available rule sets

| Config | Description | Requires type info? |
|--------|-------------|---------------------|
| `tseslint.configs.recommended` | Core correctness rules | No |
| `tseslint.configs.recommendedTypeChecked` | `recommended` + type-aware rules | Yes |
| `tseslint.configs.strict` | `recommended` + more opinionated bug-catching | No |
| `tseslint.configs.strictTypeChecked` | `strict` + type-aware strict rules | Yes |
| `tseslint.configs.stylistic` | Consistent code style | No |
| `tseslint.configs.stylisticTypeChecked` | Stylistic + type-aware stylistic rules | Yes |

**Recommendation for this project:** `recommendedTypeChecked` + `stylisticTypeChecked`. This gives the type-aware correctness rules (`no-floating-promises`, `no-misused-promises`) plus consistent style, without the more aggressive `strict` rules that may generate noise. If the team is TypeScript-savvy, upgrade to `strictTypeChecked` instead of `recommendedTypeChecked`.

### Key type-aware rules that ship enabled

Both `recommendedTypeChecked` and `strictTypeChecked` include:

| Rule | What it catches |
|------|----------------|
| `@typescript-eslint/no-floating-promises` | Unhandled Promise statements (missing `await`) |
| `@typescript-eslint/no-misused-promises` | Promises in conditionals, spreads, void returns |
| `@typescript-eslint/await-thenable` | `await` on non-Promise values |
| `@typescript-eslint/no-unnecessary-type-assertion` | `as string` when the type is already `string` |
| `@typescript-eslint/no-unsafe-assignment` | Assigning `any` to a typed variable |
| `@typescript-eslint/no-unsafe-member-access` | Accessing properties on `any` |
| `@typescript-eslint/no-unsafe-call` | Calling `any`-typed values as functions |
| `@typescript-eslint/no-unsafe-return` | Returning `any` from a typed function |
| `@typescript-eslint/restrict-template-expressions` | `${someObj}` in template strings |
| `@typescript-eslint/only-throw-error` | `throw "string"` instead of `throw new Error()` |
| `@typescript-eslint/require-await` | `async` functions with no `await` |

`strictTypeChecked` additionally enables `no-unnecessary-condition`, `no-explicit-any`, `no-non-null-assertion`, `no-deprecated`, and ~30 more rules.

## Prettier integration

### prettier.config.mjs (recommended format)

Use `.mjs` for explicit ESM — no ambiguity with `"type"` in `package.json`:

```js
// prettier.config.mjs
/** @type {import("prettier").Config} */
const config = {
  semi: true,
  singleQuote: true,
  trailingComma: 'all',
  printWidth: 100,
  tabWidth: 2,
  overrides: [
    {
      files: '*.json',
      options: { printWidth: 200 },
    },
  ],
};

export default config;
```

`.prettierrc` (plain JSON) is also valid for simpler configs, but `.mjs` enables comments, logic, and type-checking via the JSDoc annotation.

### .prettierignore

```
# .prettierignore — files Prettier must not touch
node_modules/
dist/
.turbo/
.next/
build/
coverage/
pnpm-lock.yaml
*.min.js
*.min.css
```

Prettier reads `.prettierignore` (gitignore syntax) to exclude files. This is separate from `.gitignore` and ESLint's `ignores` — all three must be kept in sync manually.

### eslint-config-prettier in flat config

`eslint-config-prettier` disables ESLint rules that would conflict with Prettier's formatting. In flat config, import the `/flat` entry point and place it **last**:

```js
import eslintConfigPrettier from 'eslint-config-prettier/flat';

export default tseslint.config(
  // ... all other configs ...
  eslintConfigPrettier,  // MUST be last — disables conflicting rules
);
```

The `/flat` import includes a `name` property for Config Inspector compatibility.

### eslint-plugin-prettier: trade-off discussion

There are two ways to run Prettier in a TypeScript project:

| Approach | How it works | Pros | Cons |
|----------|-------------|------|------|
| **Standalone Prettier** | `prettier --check .` as separate command | Fast (native), no ESLint coupling, editor format-on-save uses Prettier directly | Two CLI commands instead of one |
| **eslint-plugin-prettier** | Runs Prettier inside ESLint as a rule | Single command (`eslint --fix` formats too) | Slower (ESLint → Prettier), adds 1 dependency, can conflict with editor format-on-save |

**Recommendation: standalone Prettier.** Reasons:
1. Editor format-on-save uses Prettier directly — coupling it to ESLint means the editor path and the CLI path use different formatters, which causes inconsistent results.
2. Prettier is fast. `eslint-plugin-prettier` runs Prettier inside ESLint's rule loop, making lint ~2-3x slower.
3. The Turborepo lint task can run `prettier --check` as a separate step in the same pipeline.

If the team strongly prefers a single command, use `eslint-plugin-prettier` but be aware of the perf cost. Never use both approaches simultaneously.

### Editor format-on-save (VS Code)

`.vscode/settings.json`:
```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit"
  },
  "eslint.useFlatConfig": true
}
```

`eslint.useFlatConfig` must be `true` for the VS Code ESLint extension (v3.0+) to use `eslint.config.js`.

## Monorepo config architecture

### Root eslint.config.js as single source of truth

This project uses **one root-level `eslint.config.js`** with per-app overrides scoped via `files` globs:

```
/
├── eslint.config.js        ← Single config, per-app overrides
├── prettier.config.mjs     ← Single Prettier config
├── .prettierignore
└── apps/
    ├── frontend/           ← No local eslint.config.js
    ├── backend/            ← No local eslint.config.js
    └── workers/            ← No local eslint.config.js
```

**Why not per-app configs?** With 3 apps and 2 packages, a single root config is simpler to maintain. Per-app configs would require each to re-apply base settings, creating drift. ESLint v10's file-based config lookup (starting from each linted file's directory) theoretically supports per-package configs, but for a repo this size the root config with `files` scoping is the right call.

### Whether to extract to @packages/eslint-config

**Not recommended for this project.** Extracting lint config into a shared package (`@packages/eslint-config`) has benefits for large orgs with many repos sharing the same lint rules. For a single monorepo with 3 apps, it adds indirection without benefit. The root `eslint.config.js` is already a single source of truth.

Extract only if:
- Multiple repos need the same lint config (not the case here)
- The config grows so large it hurts readability (>200 lines — not expected)
- A team split requires independent lint rule evolution per domain

### Turborepo lint task integration

From D3, the project uses Turborepo 2.x (`tasks`, not `pipeline`). The lint task in `turbo.json`:

```jsonc
{
  "tasks": {
    "transit": { "dependsOn": ["^transit"] },
    "lint": {
      "dependsOn": ["transit"],
      "inputs": [
        "$TURBO_DEFAULT$",
        "$TURBO_ROOT$/eslint.config.js",
        "$TURBO_ROOT$/prettier.config.mjs",
        "$TURBO_ROOT$/.prettierignore"
      ],
      "outputs": [],
      "cache": true
    }
  }
}
```

Key points:
- **`dependsOn: ["transit"]`** — uses the transit node pattern (empty task, no script) so lint runs in parallel across packages but cache invalidates correctly when shared configs change. Do NOT use `dependsOn: ["^lint"]` — that forces sequential execution.
- **`inputs`** — explicitly includes root config files so that ESLint/Prettier config changes invalidate the cache. Without this, changing a lint rule would hit stale cached results.
- **`outputs: []`** — lint produces no build artifacts, so no outputs.
- **`cache: true`** — enables Turborepo caching (incremental lint is fast).

Per-package `lint` script in each `package.json`:
```json
{
  "scripts": {
    "lint": "eslint . --cache --cache-location .eslintcache && prettier --check ."
  }
}
```

Root `package.json` can also have:
```json
{
  "scripts": {
    "lint": "turbo run lint",
    "format": "prettier --write .",
    "format:check": "prettier --check ."
  }
}
```

## Architectural enforcement

These rules are the lint-time enforcement of D3 (monorepo boundaries) and D5 (no cross-app imports). They are the most critical rules in this config — everything else is code quality.

### no-restricted-imports: preventing cross-app imports

Use `@typescript-eslint/no-restricted-imports` (the TypeScript-aware extension) with per-app `files` scoping. Each app directory gets a block that prevents importing from other app directories:

```js
export default tseslint.config(
  // ... base configs ...

  // ═══════════════════════════════════════════════════════
  // Architectural boundaries — apps must NOT import each other
  // ═══════════════════════════════════════════════════════

  // Block frontend from importing backend or workers
  {
    name: 'boundary-frontend',
    files: ['apps/frontend/**/*.{ts,tsx}'],
    rules: {
      'no-restricted-imports': 'off',
      '@typescript-eslint/no-restricted-imports': ['error', {
        patterns: [
          {
            group: ['@app/backend', '@app/backend/**'],
            message: '❌ apps/frontend cannot import from @app/backend. Use @shared for shared types.',
            allowTypeImports: false,
          },
          {
            group: ['@app/workers', '@app/workers/**'],
            message: '❌ apps/frontend cannot import from @app/workers. Use @shared for shared types.',
            allowTypeImports: false,
          },
        ],
      }],
    },
  },

  // Block backend from importing frontend or workers
  {
    name: 'boundary-backend',
    files: ['apps/backend/**/*.{ts,tsx}'],
    rules: {
      'no-restricted-imports': 'off',
      '@typescript-eslint/no-restricted-imports': ['error', {
        patterns: [
          {
            group: ['@app/frontend', '@app/frontend/**'],
            message: '❌ apps/backend cannot import from @app/frontend. Use @shared for shared types.',
            allowTypeImports: false,
          },
          {
            group: ['@app/workers', '@app/workers/**'],
            message: '❌ apps/backend cannot import from @app/workers. Use @shared for shared types.',
            allowTypeImports: false,
          },
        ],
      }],
    },
  },

  // Block workers from importing frontend or backend
  {
    name: 'boundary-workers',
    files: ['apps/workers/**/*.{ts,tsx}'],
    rules: {
      'no-restricted-imports': 'off',
      '@typescript-eslint/no-restricted-imports': ['error', {
        patterns: [
          {
            group: ['@app/frontend', '@app/frontend/**'],
            message: '❌ apps/workers cannot import from @app/frontend. Use @shared for shared types.',
            allowTypeImports: false,
          },
          {
            group: ['@app/backend', '@app/backend/**'],
            message: '❌ apps/workers cannot import from @app/backend. Use @shared for shared types.',
            allowTypeImports: false,
          },
        ],
      }],
    },
  },
);
```

This pattern:
- Uses the package names from D3 (`@app/frontend`, `@app/backend`, `@app/workers`) — not filesystem paths, because the `workspace:*` protocol means imports resolve by package name
- Disables the base ESLint `no-restricted-imports` rule (required — the base rule doesn't understand TypeScript)
- Sets `allowTypeImports: false` — even `import type` from another app is forbidden; type sharing must go through `@shared`
- Each block has a `name` for Config Inspector debugging

**Alternative with relative-path patterns** (if apps don't use package names for imports):
```js
{
  group: ['**/apps/backend/**'],
  message: 'Cross-app imports forbidden. Use @shared.',
}
```

Glob patterns follow gitignore semantics. `**/apps/backend/**` matches any import path containing `apps/backend/` at any depth, regardless of whether the import is by package name or relative path.

### import/no-cycle: circular dependency detection

`import-x/no-cycle` is enabled via `importX.flatConfigs.recommended`. For monorepo use, configure it explicitly to set a reasonable `maxDepth`:

```js
{
  rules: {
    'import-x/no-cycle': ['error', {
      maxDepth: 10,             // Limit traversal depth (default: Infinity)
      ignoreExternal: false,    // Check cycles into node_modules
    }],
  },
}
```

This rule is computationally expensive — it walks the full import graph. On large repos, set `maxDepth` lower (e.g., `5`). For CI, `maxDepth: 10` provides good coverage without excessive runtime.

The rule catches:
- `A.ts` imports `B.ts` which imports `A.ts` (direct cycle)
- `A.ts` → `B.ts` → `C.ts` → `A.ts` (indirect cycle, caught up to `maxDepth`)
- Cycles through barrel files (the most common source — `index.ts` re-exports everything)

### import/order: import grouping

Enforce consistent import grouping that distinguishes external deps, internal `@shared/*` imports, and relative imports:

```js
{
  rules: {
    'import-x/order': ['error', {
      groups: [
        'builtin',                        // node:fs, node:path
        'external',                        // react, zod, fastify
        'internal',                        // @shared, @shared/*
        ['parent', 'sibling'],             // ../, ./
        'index',                           // ./index
        'type',                            // import type { ... }
      ],
      pathGroups: [
        {
          pattern: '@shared/**',
          group: 'internal',
          position: 'before',
        },
      ],
      pathGroupsExcludedImportTypes: ['builtin', 'external'],
      'newlines-between': 'always',
      alphabetize: { order: 'asc', caseInsensitive: true },
    }],
  },
}
```

This produces import ordering like:
```ts
import { readFile } from 'node:fs';           // builtin
import { z } from 'zod';                      // external
import { FastifyInstance } from 'fastify';    // external
import { apiResponseSchema } from '@shared/schemas';  // internal
import { createUser } from '@shared/utils';   // internal
import { config } from '../config';           // parent
import { helper } from './helper';            // sibling
import type { User } from '@shared/types';    // type
```

### consistent-type-imports: enforcing type-only imports

This rule auto-fixes imports to use `import type` for type-only imports, aligning with D1's `isolatedDeclarations` and D2's `verbatimModuleSyntax`:

```js
{
  rules: {
    '@typescript-eslint/consistent-type-imports': ['error', {
      prefer: 'type-imports',
      fixStyle: 'separate-type-imports',    // import { X } → import { type X } on separate line
    }],
  },
}
```

With `fixStyle: 'separate-type-imports'`:
```ts
// Before --fix
import { User, createUser } from '@shared';

// After --fix
import { createUser } from '@shared';
import type { User } from '@shared';
```

**Interaction with `verbatimModuleSyntax`:** D2 enables `verbatimModuleSyntax: true` in tsconfig. When `verbatimModuleSyntax` is on, TypeScript enforces WYSIWYG module syntax at build time. The ESLint `consistent-type-imports` rule is complementary — it auto-fixes to the correct style. Only use one or the other for enforcement; using both is redundant. Since D2 already has `verbatimModuleSyntax`, `consistent-type-imports` with `--fix` acts as a migration helper but is not strictly necessary for enforcement. Keep it as `'warn'` or `'error'` for developer convenience (auto-fix on save).

### Additional useful rules for this stack

```js
{
  rules: {
    // no-explicit-any — enforce strict typing
    '@typescript-eslint/no-explicit-any': 'error',

    // explicit-function-return-type — disable for apps, enable for @shared
    // Rationale: app code has contextual inference; shared exports need explicit types
    '@typescript-eslint/explicit-function-return-type': 'off',

    // no-console — per-app config
    // Workers: allow console.log/error for operational logging
    // Frontend/Backend: warn (use structured logger in backend, debug in frontend)
  },
},

// no-console: allow in workers
{
  name: 'workers-console',
  files: ['apps/workers/**/*.ts'],
  rules: {
    'no-console': 'off',
  },
},

// no-console: warn in frontend/backend
{
  name: 'apps-console',
  files: ['apps/{frontend,backend}/**/*.ts'],
  rules: {
    'no-console': 'warn',
  },
},
```

## Antipatterns and common mistakes

1. **Using `.eslintrc` with ESLint v10** — The legacy format is fully removed. ESLint v10 will not read `.eslintrc` files. There is no `ESLINT_USE_FLAT_CONFIG` flag (it's always true). If a `.eslintrc` file exists alongside `eslint.config.js`, ESLint ignores it silently — no error, no warning. Delete all `.eslintrc.*` files.

2. **Type-aware rules without parserOptions** — `recommendedTypeChecked` and `strictTypeChecked` require `languageOptions.parserOptions.projectService` (or `project`). Without it, type-aware rules are silently disabled. No error is thrown. Verify with `npx eslint --inspect-config` that `parserOptions` is resolved.

3. **Forgetting eslint.config.js in Turborepo cache inputs** — If `eslint.config.js` is not in the lint task's `inputs`, changing a lint rule hits the Turborepo cache and old results are served. Always include root config files in `inputs`.

4. **Prettier and ESLint conflicting on formatting rules** — Without `eslint-config-prettier` (placed last), ESLint rules like `@typescript-eslint/indent` or `max-len` can conflict with Prettier. Each edit → format → lint cycle produces a different result. Always use `eslint-config-prettier/flat` as the last config element.

5. **no-restricted-imports pattern too broad or too narrow** — Using `**/apps/**` blocks everything including the current app importing itself (harmless). Using only the package name without `/**` blocks `@app/backend` but not `@app/backend/some/deep/path`. Always include both: `['@app/backend', '@app/backend/**']`.

6. **Using the wrong `no-restricted-imports`** — ESLint's built-in `no-restricted-imports` does not understand TypeScript syntax (`import type`, `export type`). Always use `@typescript-eslint/no-restricted-imports` and disable the base rule.

7. **eslint-plugin-import on ESLint v10** — The original plugin has not shipped an ESLint v10-compatible release. If you install it, rules may silently fail or throw runtime errors (removed APIs: `context.parserPath`, `FileEnumerator`). Use `eslint-plugin-import-x` instead.

8. **Mixing `tseslint.config()` wrapper styles** — Don't mix `defineConfig()` and bare arrays. Pick one wrapper for the whole config. If using `tseslint.config()`, pass all configs as arguments (not a single array).

## Audit-flagged gaps — resolution

All 6 flagged items resolved:

1. **ESLint v9 flat config exact syntax** → **Resolved.** ESLint is at v10.4.0 (not v9). Flat config is the only format — no legacy fallback. Complete annotated `eslint.config.js` in Reference Snippets below. Source: eslint.org/docs/latest/use/configure/configuration-files, ESLint v10.0.0 release notes.

2. **parserOptions.project equivalent in flat config** → **Resolved.** `languageOptions.parserOptions.projectService: true` is the current recommendation. `project: ['./path/to/tsconfig.json', ...]` is the explicit fallback. The old `parserOptions.project` key moves into `languageOptions.parserOptions` unchanged — the nesting is the only difference from `.eslintrc`. Source: typescript-eslint.io docs, typescript-eslint performance repo.

3. **projectService vs project array** → **Resolved.** `projectService` is recommended by default (simpler, editor-consistent). `project: string[]` is the documented fallback for large repos where `projectService` has known performance issues. For this 3-app monorepo, `projectService: true` is the right choice. Source: typescript-eslint issue #9571, typescript-eslint/performance repo.

4. **eslint-plugin-import flat config status** → **Resolved.** Original `eslint-plugin-import` v2.32.0 has flat config support (`flatConfigs`) but does NOT support ESLint v10 (no release with v10 API compatibility). Use `eslint-plugin-import-x` v4.16.2, which shipped ESLint v10 support in March 2026 and is actively maintained. Source: github.com/import-js/eslint-plugin-import/issues/3227, github.com/un-ts/eslint-plugin-import-x.

5. **no-restricted-imports exact config** → **Resolved.** Use `@typescript-eslint/no-restricted-imports` with per-app `files` blocks. Exact config in Architectural Enforcement section above. Pattern matched to D3 package names (`@app/frontend`, `@app/backend`, `@app/workers`) and D5 boundary rules (all cross-app sharing through `@shared`). Source: typescript-eslint docs, ESLint no-restricted-imports docs, community monorepo patterns.

6. **eslint-config-prettier v9 compatibility** → **Resolved.** Compatible with ESLint v10. Import from `eslint-config-prettier/flat` and place as the last element in the config array. The `/flat` entry point includes a `name` for Config Inspector. Source: github.com/prettier/eslint-config-prettier README.

## Reference snippets

### eslint.config.js (root, full monorepo setup)

```js
// @ts-check
import js from '@eslint/js';
import tseslint from 'typescript-eslint';
import { importX } from 'eslint-plugin-import-x';
import eslintConfigPrettier from 'eslint-config-prettier/flat';

export default tseslint.config(
  // ═══════════════════════════════════════════════════════
  // Global ignores (replaces .eslintignore)
  // ═══════════════════════════════════════════════════════
  {
    ignores: [
      '**/node_modules/**',
      '**/dist/**',
      '**/.turbo/**',
      '**/.next/**',
      '**/build/**',
      '**/coverage/**',
    ],
  },

  // ═══════════════════════════════════════════════════════
  // Base recommended configs
  // ═══════════════════════════════════════════════════════
  js.configs.recommended,
  tseslint.configs.recommendedTypeChecked,
  tseslint.configs.stylisticTypeChecked,

  // ═══════════════════════════════════════════════════════
  // Type-aware linting — global config
  // ═══════════════════════════════════════════════════════
  {
    languageOptions: {
      parserOptions: {
        projectService: true,
        tsconfigRootDir: import.meta.dirname,
      },
    },
  },

  // ═══════════════════════════════════════════════════════
  // Import rules — cycle detection and import ordering
  // ═══════════════════════════════════════════════════════
  importX.flatConfigs.recommended,
  importX.flatConfigs.typescript,
  {
    settings: {
      'import-x/resolver': {
        typescript: true,
      },
    },
    rules: {
      'import-x/no-cycle': ['error', { maxDepth: 10 }],
      'import-x/order': ['error', {
        groups: [
          'builtin',
          'external',
          'internal',
          ['parent', 'sibling'],
          'index',
          'type',
        ],
        pathGroups: [
          {
            pattern: '@shared/**',
            group: 'internal',
            position: 'before',
          },
        ],
        pathGroupsExcludedImportTypes: ['builtin', 'external'],
        'newlines-between': 'always',
        alphabetize: { order: 'asc', caseInsensitive: true },
      }],
    },
  },

  // ═══════════════════════════════════════════════════════
  // Core TypeScript rules
  // ═══════════════════════════════════════════════════════
  {
    rules: {
      '@typescript-eslint/consistent-type-imports': ['error', {
        prefer: 'type-imports',
        fixStyle: 'separate-type-imports',
      }],
      '@typescript-eslint/no-explicit-any': 'error',
    },
  },

  // ═══════════════════════════════════════════════════════
  // Architectural boundaries — apps must NOT import each other
  // ═══════════════════════════════════════════════════════
  {
    name: 'boundary-frontend',
    files: ['apps/frontend/**/*.{ts,tsx}'],
    rules: {
      'no-restricted-imports': 'off',
      '@typescript-eslint/no-restricted-imports': ['error', {
        patterns: [
          {
            group: ['@app/backend', '@app/backend/**'],
            message: '❌ apps/frontend cannot import from @app/backend. Use @shared.',
            allowTypeImports: false,
          },
          {
            group: ['@app/workers', '@app/workers/**'],
            message: '❌ apps/frontend cannot import from @app/workers. Use @shared.',
            allowTypeImports: false,
          },
        ],
      }],
    },
  },
  {
    name: 'boundary-backend',
    files: ['apps/backend/**/*.ts'],
    rules: {
      'no-restricted-imports': 'off',
      '@typescript-eslint/no-restricted-imports': ['error', {
        patterns: [
          {
            group: ['@app/frontend', '@app/frontend/**'],
            message: '❌ apps/backend cannot import from @app/frontend. Use @shared.',
            allowTypeImports: false,
          },
          {
            group: ['@app/workers', '@app/workers/**'],
            message: '❌ apps/backend cannot import from @app/workers. Use @shared.',
            allowTypeImports: false,
          },
        ],
      }],
    },
  },
  {
    name: 'boundary-workers',
    files: ['apps/workers/**/*.ts'],
    rules: {
      'no-restricted-imports': 'off',
      '@typescript-eslint/no-restricted-imports': ['error', {
        patterns: [
          {
            group: ['@app/frontend', '@app/frontend/**'],
            message: '❌ apps/workers cannot import from @app/frontend. Use @shared.',
            allowTypeImports: false,
          },
          {
            group: ['@app/backend', '@app/backend/**'],
            message: '❌ apps/workers cannot import from @app/backend. Use @shared.',
            allowTypeImports: false,
          },
        ],
      }],
    },
  },

  // ═══════════════════════════════════════════════════════
  // Per-app console rules
  // ═══════════════════════════════════════════════════════
  {
    name: 'workers-console',
    files: ['apps/workers/**/*.ts'],
    rules: { 'no-console': 'off' },
  },
  {
    name: 'apps-console',
    files: ['apps/{frontend,backend}/**/*.{ts,tsx}'],
    rules: { 'no-console': 'warn' },
  },

  // ═══════════════════════════════════════════════════════
  // Prettier — MUST be last
  // ═══════════════════════════════════════════════════════
  eslintConfigPrettier,
);
```

### prettier.config.mjs

```js
// prettier.config.mjs
/** @type {import("prettier").Config} */
const config = {
  semi: true,
  singleQuote: true,
  trailingComma: 'all',
  printWidth: 100,
  tabWidth: 2,
};

export default config;
```

### .prettierignore

```
node_modules/
dist/
.turbo/
.next/
build/
coverage/
pnpm-lock.yaml
*.min.js
```

### Turborepo lint task (turbo.json fragment)

```jsonc
{
  "tasks": {
    "transit": { "dependsOn": ["^transit"] },
    "lint": {
      "dependsOn": ["transit"],
      "inputs": [
        "$TURBO_DEFAULT$",
        "$TURBO_ROOT$/eslint.config.js",
        "$TURBO_ROOT$/prettier.config.mjs",
        "$TURBO_ROOT$/.prettierignore"
      ],
      "outputs": [],
      "cache": true
    }
  }
}
```

## Sources used

- [ESLint v10.0.0 Release Notes](https://eslint.org/blog/2026/02/eslint-v10.0.0-released/) — official (v10 breaking changes, eslintrc removal, new config lookup)
- [ESLint Configuration Files](https://eslint.org/docs/latest/use/configure/configuration-files) — official (flat config reference)
- [typescript-eslint Getting Started](https://typescript-eslint.io/getting-started/) — official (installation, recommended configs)
- [typescript-eslint Package API](https://typescript-eslint.io/packages/typescript-eslint/) — official (config() helper, configs list, projectService)
- [typescript-eslint Performance Repo](https://github.com/typescript-eslint/performance) — official (projectService vs project benchmarking)
- [typescript-eslint Issue #9571](https://github.com/typescript-eslint/typescript-eslint/issues/9571) — official (projectService performance tracking)
- [eslint-plugin-import Issue #3227](https://github.com/import-js/eslint-plugin-import/issues/3227) — official (ESLint v10 compatibility status)
- [eslint-plugin-import-x on GitHub](https://github.com/un-ts/eslint-plugin-import-x) — official (flat config, ESLint v10 support, migration docs)
- [eslint-config-prettier README](https://github.com/prettier/eslint-config-prettier) — official (flat config usage, `/flat` entry point)
- [Prettier Configuration Docs](https://prettier.io/docs/en/configuration.html) — official (config file formats, options)
- [Prettier Install Docs](https://prettier.io/docs/en/install.html) — official (editor integration, ignore files)
- [Turborepo Docs — Tasks](https://turbo.build/repo/docs/reference/configuration#tasks) — official (tasks config, inputs micro-syntax, transit node pattern)
- [ESLint no-restricted-imports Rule](https://eslint.org/docs/latest/rules/no-restricted-imports) — official (patterns, group, glob syntax)
- [typescript-eslint no-restricted-imports Rule](https://typescript-eslint.io/rules/no-restricted-imports/) — official (TypeScript extension, regex, allowTypeImports)
- [typescript-eslint consistent-type-imports Rule](https://typescript-eslint.io/rules/consistent-type-imports/) — official (fixStyle, verbatimModuleSyntax interaction)
