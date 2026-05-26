# Skill: TypeScript — D5: Shared Types and Contracts

## Summary

Architectural skill covering the contract layer of a TypeScript monorepo: what belongs in `@shared`, how to structure it, how to co-locate Zod schemas with inferred types, how to design job payload contracts, how to enforce boundaries, and how to evolve shared types without breaking consumers. Knowledge cutoff: 2026-05-25. Audit risk: **Lowest**.

**Dependencies:** D1 (TypeScript Language Core), D3 (Multi-App Monorepo), and D4 (Runtime Validation with Zod) — all completed and verified.

**Propagation lock status: CLEAN.** D1, D3, and D4 have no unresolved critical gaps that affect this skill. One architectural discrepancy between D3 and D4 is reconciled below: D3 structures packages as `@shared/types` + `@shared/utils` (two packages), while D4 implicitly uses `@shared/schemas` with internal subdirectories. This document resolves the decision — single `@shared` package with grouped subdirectories for the current project scale (3 apps, 5 packages total).

## Version landscape

No primary library version — D5 is architectural. Alignment notes:

| Dependency | Version | Source |
|---|---|---|
| TypeScript | 6.0.3 (May 2026) | D1 |
| pnpm | 11.3.0 | D3 |
| Turborepo | 2.9.14 | D3 |
| Zod | 4.4.3 | D4 |
| Workspace protocol | `workspace:*` | D3 |
| `isolatedDeclarations` | TS 5.5+ (enabled in D1 baseline) | D1 |
| `verbatimModuleSyntax` | TS 5.0+ (recommended in D2) | D2 |
| BullMQ | v5.x (for typed job payloads) | D10 (forthcoming) |

## Package structure decision

### Recommendation: single `@shared` package with grouped subdirectories

For the current project (3 apps, ~5 packages total), **one `@packages/shared` package** with internal grouping by concern is the right call. Reasoning:

- The Turborepo documentation recommends "one purpose per package" and warns against mega-packages — but "one purpose" here is "the monorepo contract layer." That is a single coherent purpose.
- Splitting into `@shared/types`, `@shared/schemas`, `@shared/utils` creates three packages that are almost always imported together — every consumer needs types from schemas, schemas produce types, and utils wrap both. Three packages means three `package.json` files, three `tsconfig.json` files, three Turborepo task entries, and three dependency declarations per consumer for no meaningful separation.
- Mitigation against growth: if `@shared` exceeds ~50 exported symbols or a clear domain boundary emerges (e.g., a separate `@shared/jobs` package when job types grow to 20+), split then. Use the "second time" rule — extract when duplication or coupling becomes actual friction, not before.
- Turborepo's own basic example uses a single `@repo/ui` package with internal `src/` structure, not micro-packages.

### When to split (for future reference)

| Signal | Action |
|---|---|
| >50 exports from a single barrel | Split by domain (e.g., `@shared/auth`, `@shared/billing`) |
| Circular dependency forming within `@shared` | Extract the shared dependency into a `@shared/core` or `@shared/common` |
| One subdirectory accumulates heavy runtime deps | Extract so consumers that only need types don't pull runtime code |
| A subdirectory needs independent versioning | Extract to its own package so it can be versioned separately |

### Package name in D3 vs D4

D3 uses `@shared/types` (name field). D4 uses `@shared/schemas` (import path). This document standardizes on **`@shared`** as the package name, with the internal directory structure providing the grouping:

```
packages/shared/
  package.json          ← name: "@shared"
  tsconfig.json
  src/
    schemas/            ← Zod schemas + inferred types (D4 co-location)
    types/              ← Plain TypeScript types/interfaces (no runtime validation needed)
    enums/              ← Constants, string enums, config values
    errors/             ← Error codes, error response shapes
    jobs/               ← Job payload types (D5 ↔ D8 ↔ D10)
    index.ts            ← Barrel export
```

### Annotated package.json

```jsonc
{
  "name": "@shared",
  "version": "0.0.0",
  "private": true,
  "type": "module",
  "exports": {
    ".": {
      "types": "./src/index.ts",
      "default": "./src/index.ts"
    },
    "./schemas": {
      "types": "./src/schemas/index.ts",
      "default": "./src/schemas/index.ts"
    },
    "./types": {
      "types": "./src/types/index.ts",
      "default": "./src/types/index.ts"
    },
    "./errors": {
      "types": "./src/errors/index.ts",
      "default": "./src/errors/index.ts"
    },
    "./jobs": {
      "types": "./src/jobs/index.ts",
      "default": "./src/jobs/index.ts"
    }
  },
  "scripts": {
    "lint": "eslint . --max-warnings 0",
    "check-types": "tsc --noEmit"
  },
  "peerDependencies": {
    "zod": "^4.0.0"
  },
  "devDependencies": {
    "zod": "^4.0.0",
    "typescript": "catalog:"
  }
}
```

**Key decisions in this package.json:**

- **JIT package (no build step):** `exports` points directly to `.ts` source. The consuming app's bundler compiles it. No `dist/`, no `tsc` build, no Turborepo `outputs`. This is the simplest setup for an internal-only package. Source: Turborepo docs on internal packages (JIT strategy).
- **`private: true`:** This package is never published to npm. The `workspace:*` protocol resolves it locally.
- **Subpath exports:** `@shared/schemas`, `@shared/types`, `@shared/errors`, `@shared/jobs` allow consumers to import only what they need without dragging in the entire barrel. This is the primary defense against barrel bloat — consumers opt into the subpackage they need.
- **No `main` field:** With `"type": "module"` and an `exports` map, `main` is redundant and can cause resolution conflicts. `exports` is the sole source of truth for entry points.
- **Zod as peer + dev dependency:** Consumers bring their own Zod (peer). `@shared` uses it internally (dev). This prevents duplicate Zod instances. Source: Zod Library Authors guide.

**Why JIT (not compiled) for this package:**

- `@shared` is internal-only, consumed only by apps in the monorepo
- All three apps (frontend, backend, workers) use bundlers or runtimes that understand TypeScript
- No Turborepo cache needed — type-checking is handled by `check-types` script, and the package has no build artifacts
- Simplest setup — no `dist/` directory, no `tsc` output to manage, no stale `.d.ts` files

### Annotated tsconfig.json

```jsonc
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "${configDir}/dist",
    "rootDir": "${configDir}/src",
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "declaration": true,
    "declarationMap": true,
    "isolatedDeclarations": true,
    "verbatimModuleSyntax": true
  },
  "include": ["src"],
  "exclude": ["node_modules", "dist"]
}
```

**Key decisions in this tsconfig.json:**

- **`moduleResolution: "nodenext"`:** The safe common denominator (D2). Code that resolves under `nodenext` works in Node.js (backend, workers) and in bundlers (frontend). The reverse is not true — `bundler` resolution allows extensionless imports that fail under `nodenext`. Source: D2 skill document, section "moduleResolution choice per app type."
- **`isolatedDeclarations: true`:** Consistent with D1. Every export must have an explicit type annotation. This enables parallel `.d.ts` generation and forces discipline in shared types.
- **`verbatimModuleSyntax: true`:** Consistent with D2. Enforces `import type` / `export type` for type-only imports/exports. Prevents silent runtime failures from type-only imports being emitted as real imports.
- **`declaration: true` + `declarationMap: true`:** Declaration maps enable go-to-definition across package boundaries in the IDE. Even as a JIT package, declaration maps are useful when the consumer's TypeScript server resolves types.
- **No `composite: true`:** D2 and D3 both warn against project references when using Turborepo. Turborepo's task graph already handles build ordering — project references add redundant configuration.

## What belongs in @shared — decision framework

The rule: **if two or more apps need it, and it's a contract (type, schema, constant), it belongs in `@shared`.** If it has app-specific runtime code, component props, or data-layer types, it does not.

| Category | In @shared? | Location | Rationale |
|---|---|---|---|
| DTOs (request/response shapes) | YES | `src/types/` or derived from schemas | API contract between frontend and backend. Derive from Zod schema when validation is needed (→ `src/schemas/`); write as plain interface when compile-time typing suffices. |
| Domain entities | YES | `src/types/` | Shared representation of core business objects. App-specific projections (e.g., `UserWithOrders`) stay in the app. |
| Enums and constants | YES | `src/enums/` | String literal unions preferred for enums (discriminated unions work better with TypeScript). Use `z.enum()` for runtime-validated enums. |
| Job payload types | YES | `src/jobs/` | Backend enqueues, workers dequeue — must be identical. Non-negotiable. |
| API error types | YES | `src/errors/` | Error codes, error response shape consumed by frontend and produced by backend. |
| Utility/helper types | YES | `src/types/` | `Paginated<T>`, `ApiResponse<T>`, `ApiError`, `Nullable<T>`, etc. Shared generic wrappers. |
| Zod schemas | YES | `src/schemas/` | Co-located with inferred types per D4 pattern. |
| Prisma types | NO | Backend only | Prisma's generated types leak database structure. Wrap in DTOs before exposing to other apps. |
| Component props | NO | Frontend only | React component types have no meaning outside the frontend. |
| Internal service types | NO | Respective app | Types for service-internal logic, DI containers, middleware config — not shared. |
| Fastify route schemas | NO | Backend only | Route definitions are backend infrastructure. Share only the DTO types they validate. |
| React Hook Form types | NO | Frontend only | Form state management is a frontend concern. Share only the Zod schemas. |

### DTOs: schema-derived vs plain interface

```ts
// Use schema-derived when: validation runs at a boundary (API request/response)
// → @shared/src/schemas/user.ts (D4 co-location pattern)
export const CreateUserSchema = z.object({
  name: z.string().min(1),
  email: z.email(),
});
export type CreateUserInput = z.input<typeof CreateUserSchema>;

// Use plain interface when: compile-time typing only, no runtime validation needed
// → @shared/src/types/navigation.ts
export interface Breadcrumb {
  label: string;
  href: string;
  current?: boolean;
}
```

### Enums: const object vs Zod enum — consistency rule

In `@shared`, when a value set is used for both runtime checks and TypeScript types, define the schema with `z.enum()` and infer the type. When only compile-time discrimination is needed, use a string literal union. **Avoid TypeScript `enum`** — they emit runtime code, don't tree-shake well, and don't compose with discriminated unions.

```ts
// Runtime-validated enum: schema + inferred type
// @shared/src/schemas/user.ts
export const UserRoleSchema = z.enum(["admin", "user", "guest"]);
export type UserRole = z.infer<typeof UserRoleSchema>;

// Compile-time only: string literal union
// @shared/src/types/status.ts
export type TaskStatus = "pending" | "running" | "completed" | "failed";
```

Source for TypeScript `enum` avoidance: D1 pattern (const objects and string unions). Source for Zod enum: D4.

## Zod co-location pattern (building on D4)

D4 established the pattern: every domain file exports the schema `const` and its inferred `type` together. D5 extends this with export strategy and peer dependency decisions.

### Schema + inferred type in the same file

```ts
// @shared/src/schemas/user.ts
import { z } from "zod/v4";

export const UserSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1),
  email: z.email(),
  role: z.enum(["admin", "user", "guest"]),
  createdAt: z.date(),
});
export type User = z.infer<typeof UserSchema>;

export const CreateUserSchema = z.object({
  name: z.string().min(1),
  email: z.email(),
  role: z.enum(["admin", "user", "guest"]).default("user"),
});
export type CreateUserInput = z.input<typeof CreateUserSchema>;
export type CreateUserOutput = z.output<typeof CreateUserSchema>;
```

### Export strategy: schema for runtime, type for compile-time

Consumers import what they need:

```ts
// Frontend — needs both schema (for React Hook Form) and type (for state)
import { CreateUserSchema, type CreateUserInput } from "@shared/schemas";

// Backend — needs schema (for Fastify type provider) and type (for service layer)
import { CreateUserSchema, type CreateUserInput } from "@shared/schemas";

// Worker — needs schema (for job payload validation) and type (for processing)
import { UserSchema, type User } from "@shared/schemas";
```

### Zod as peer dependency

**Recommendation: `@shared` declares Zod in `peerDependencies` + `devDependencies`. Consumers install Zod themselves.** Do not re-export Zod from `@shared`.

Reasoning:
- Re-exporting Zod from `@shared` couples the Zod version to the shared package version. If a consumer needs a Zod feature from a newer minor version, it must wait for `@shared` to update.
- Peer dependency makes the constraint explicit: "you must have Zod installed to use these schemas." The consumer controls the version within the allowed range.
- pnpm's strict resolution already prevents duplicate instances — `pnpm` catalogs or overrides enforce a single Zod version across the workspace.

```jsonc
// @shared/package.json
{
  "peerDependencies": {
    "zod": "^4.0.0"
  },
  "devDependencies": {
    "zod": "^4.0.0"
  }
}
```

```yaml
# pnpm-workspace.yaml (root) — enforce single Zod version
catalog:
  zod: ^4.4.3
```

```jsonc
// Any consumer's package.json
{
  "dependencies": {
    "@shared": "workspace:*",
    "zod": "catalog:"
  }
}
```

### Import path: `zod/v4`

D4 uses `zod/v4` (not `zod/v4/core`). While `zod/v4/core` offers broader compatibility (works with both Zod Classic and Mini), `zod/v4` is the project's established convention from D4. D5 inherits this. If the project later adopts Zod Mini, a single find-and-replace across `@shared` is sufficient.

## Barrel export architecture

### Recommended: explicit named re-exports with `export type`

```ts
// @shared/src/index.ts — root barrel
export {
  UserSchema, type User,
  CreateUserSchema, type CreateUserInput, type CreateUserOutput,
  UpdateUserSchema, type UpdateUserInput,
} from "./schemas/user";

export {
  PostSchema, type Post,
  CreatePostSchema, type CreatePostInput, type CreatePostOutput,
} from "./schemas/post";

export {
  ApiResponseSchema, type ApiResponse,
  PaginatedResponseSchema, type PaginatedResponse,
} from "./schemas/api";

export { type Paginated, type ApiError, type Nullable } from "./types/utility";

export { type Breadcrumb, type NavItem } from "./types/navigation";

export { type UserRole, type TaskStatus } from "./enums";

export { type ErrorCode, type ErrorResponse } from "./errors";

export { type EmailJobPayload, type NotificationJobPayload, type JobName } from "./jobs";
```

### Why explicit named re-exports (not `export *`)

| Concern | `export *` | Explicit named |
|---|---|---|
| Tree-shaking | Poor — bundler can't be sure what's used | Good — only imported symbols are bundled |
| API clarity | Invisible — need tooling to see public surface | Self-documenting — barrel is the public API |
| Name collisions | Silent — last export wins | Compile error — forces resolution |
| `verbatimModuleSyntax` | Works but can silently export types as values | Each type export is explicitly `export type` |
| `isolatedDeclarations` | Works but loses per-symbol control | Each type-only export is marked, enabling fast `.d.ts` emit |

### Grouping strategy: subpath exports

The root barrel (`@shared`) re-exports everything. Subpath exports (`@shared/schemas`, `@shared/types`, `@shared/errors`, `@shared/jobs`) let consumers import only the group they need:

```ts
// Consumer that only needs schemas
import { UserSchema, type User } from "@shared/schemas";

// Consumer that only needs error types
import { type ErrorCode, type ErrorResponse } from "@shared/errors";

// Consumer that needs everything (rare, but valid)
import { UserSchema, type User, type ApiResponse } from "@shared";
```

**Trade-off:** Subpath exports require entries in `package.json` `exports` field. Without them, consumers must use deep imports (`@shared/src/schemas/user`), which bypass the public API. The `exports` field is the enforcement mechanism — if a path isn't in `exports`, it's not importable. Set up `exports` entries for each group at creation time; it's ~3 lines per group.

### Type-only exports

Every type-only re-export in a barrel must use `export type`. This is enforced by `verbatimModuleSyntax` (D2) and `isolatedDeclarations` (D1):

```ts
// ✅ Correct
export type { User } from "./schemas/user";
export { UserSchema, type User } from "./schemas/user";  // mixed: value + type

// ❌ Wrong with verbatimModuleSyntax
export { User } from "./schemas/user";  // User is a type — errors at compile time
```

## Job payload type pattern (critical intersection D5 ↔ D8 ↔ D10)

The contract between backend (enqueues jobs) and workers (dequeues and processes jobs) lives in `@shared`. Both sides must agree on the payload shape. The type defined in `@shared` is the canonical source.

### Typed payload in @shared

BullMQ v5 supports generics: `Queue<DataType, ResultType, NameType>`. Define the shared types, then parameterize:

```ts
// @shared/src/jobs/index.ts

// ── Job name discriminated union ──
export const JOB_NAMES = ["send-email", "send-notification"] as const;
export type JobName = (typeof JOB_NAMES)[number];

// ── Per-job payload types ──
export interface EmailJobPayload {
  version: 1;
  jobName: "send-email";
  to: string;
  subject: string;
  body: string;
  templateId?: string;
}

export interface NotificationJobPayload {
  version: 1;
  jobName: "send-notification";
  userId: string;
  title: string;
  message: string;
  channel: "push" | "in-app";
}

// ── Discriminated union for workers ──
export type JobPayload = EmailJobPayload | NotificationJobPayload;

// ── Zod schemas for runtime validation at worker boundary ──
export const EmailJobPayloadSchema = z.object({
  version: z.literal(1),
  jobName: z.literal("send-email"),
  to: z.string().email(),
  subject: z.string().min(1),
  body: z.string().min(1),
  templateId: z.string().optional(),
});

export const NotificationJobPayloadSchema = z.object({
  version: z.literal(1),
  jobName: z.literal("send-notification"),
  userId: z.string().uuid(),
  title: z.string().min(1),
  message: z.string().min(1),
  channel: z.enum(["push", "in-app"]),
});
```

### Backend enqueues, workers dequeue — same type

```ts
// @apps/backend — enqueuing
import { Queue } from "bullmq";
import type { EmailJobPayload, JobPayload, JobName } from "@shared/jobs";

const emailQueue = new Queue<JobPayload, void, JobName>("email", {
  connection,
});

await emailQueue.add("send-email", {
  version: 1,
  jobName: "send-email",
  to: "user@example.com",
  subject: "Welcome",
  body: "Hello!",
});

// @apps/workers — dequeuing and processing
import { Worker } from "bullmq";
import { EmailJobPayloadSchema } from "@shared/jobs";
import type { JobPayload, JobName } from "@shared/jobs";

const worker = new Worker<JobPayload, void, JobName>(
  "email",
  async (job) => {
    // Runtime validation with shared schema
    const parsed = EmailJobPayloadSchema.safeParse(job.data);
    if (!parsed.success) {
      throw new Error(`Invalid payload: ${parsed.error.message}`);
    }

    switch (job.name) {
      case "send-email":
        // parsed.data is fully typed as EmailJobPayload
        await sendEmail(parsed.data);
        break;
    }
  },
  { connection }
);
```

### Versioning strategy for evolving payloads

**Rule: never break backward compatibility.** Add optional fields only. Never remove, rename, or change the type of an existing field.

```ts
// v1 — initial
export interface EmailJobPayload {
  version: 1;
  jobName: "send-email";
  to: string;
  subject: string;
  body: string;
}

// v2 — add optional field (backward compatible: v1 workers ignore it)
export interface EmailJobPayloadV2 {
  version: 2;
  jobName: "send-email";
  to: string;
  subject: string;
  body: string;
  templateId?: string;       // NEW — optional
  replyTo?: string;           // NEW — optional
}

// Discriminated union across versions
export type EmailJobPayload = EmailJobPayload | EmailJobPayloadV2;
```

**For unavoidable breaking changes** (e.g., renaming a field, changing a type, splitting a payload):
1. Create a **new job name** (e.g., `"send-email-v2"`)
2. Deploy workers that handle **both** `"send-email"` and `"send-email-v2"`
3. Switch the backend to enqueue `"send-email-v2"`
4. Wait for the old job queue to drain (workers still processing `"send-email"`)
5. Remove the old handler and type

This is the "dual-write, dual-read, then deprecate" pattern. It requires discipline but avoids downtime during rolling deployments.

### Discriminated union for multiple job types in one queue

The `jobName` field serves as the discriminant. The worker switches on it:

```ts
// Worker processor with fully typed discriminated union
const worker = new Worker<JobPayload, void, JobName>("notifications", async (job) => {
  switch (job.data.jobName) {
    case "send-email": {
      // job.data narrowed to EmailJobPayload
      break;
    }
    case "send-notification": {
      // job.data narrowed to NotificationJobPayload
      break;
    }
  }
}, { connection });
```

### Should the worker validate the payload at runtime?

**Yes.** Even though the backend uses the same type, runtime validation at the worker boundary catches:
- Serialization/deserialization issues (dates become strings, BigInts are lost)
- Version mismatches during rolling deployments (old worker receives new payload field with unexpected value)
- Queue corruption or manual queue manipulation

The Zod schema defined in `@shared/jobs` validates at consumption:

```ts
// The shared schema validates at the worker boundary
const result = EmailJobPayloadSchema.safeParse(job.data);
if (!result.success) {
  logger.error("Invalid job payload", { errors: result.error.flatten() });
  throw new Error(`Invalid payload for job ${job.id}`);
}
// result.data is fully typed — no casting needed
```

## Avoiding coupling and circular dependencies

### The rule

- Apps import **from** `@shared` — never from each other.
- `@shared` imports **from no app** — zero runtime dependencies on `/apps/*`.
- This is enforced at two levels:
  1. **ESLint `no-restricted-imports`** (D7): configured per-package to block cross-app imports
  2. **Turborepo task graph** (D3): `@shared#build` runs first, apps depend on it via `"^build"`

### Detecting violations

**madge** detects circular dependencies across the workspace:

```bash
# Run from workspace root — checks all packages
npx madge --circular --extensions ts --ts-config tsconfig.base.json packages/ apps/
```

CI integration: add a root script that runs `madge --circular` and exits non-zero if cycles exist.

**ESLint `no-restricted-imports`** (configured in D7) blocks cross-app imports at lint time:

```jsonc
// .eslintrc for @app/frontend
{
  "rules": {
    "no-restricted-imports": ["error", {
      "patterns": [{
        "group": ["@app/backend/*", "@app/workers/*"],
        "message": "Apps must import from @shared, not from other apps directly."
      }]
    }]
  }
}
```

### Dependency inversion: when a type seems to need app code

If a type in `@shared` feels like it needs a type from `/apps/backend`, the dependency is inverted. The fix depends on what's being shared:

| Scenario | Fix |
|---|---|
| Shared type references a Prisma model | Extract a plain interface for the shared fields. Backend maps Prisma ↔ shared type at the boundary. |
| Shared type references a Fastify type | Extract the relevant subset. Fastify plugins/types stay in backend. |
| Two apps need the same derived type | Lift the derivation into `@shared`. Both apps consume the shared version. |
| `@shared` would need to import a library only backend uses | Move the library-specific wrapper to a backend utility. Shared keeps the generic type. |

### Granular packages as escape hatch

If `@shared` grows unwieldy (50+ exports, subdirectories with conflicting concerns), split reactively:

```
packages/
  shared/          ← core: schemas, DTOs, enums, utility types
  shared-jobs/     ← job payloads (if job types become complex enough to warrant independence)
```

The split should be motivated by real friction, not hypothetical future growth.

## Type evolution and backward compatibility

### Adding optional fields vs versioning types

| Change | Strategy |
|---|---|
| Add optional field | Add to existing type. Backward compatible — old consumers ignore unknown fields, new consumers handle the optional. |
| Add required field with default | Add as optional first, then make required in a coordinated update after all consumers handle it. |
| Remove a field | Deprecate with `@deprecated` first. Remove in a coordinated update after all consumers stop reading it. |
| Rename a field | Add new field name, deprecate old. Dual-write both during transition. Remove old after consumers migrate. |
| Change field type | Never. Create a new type (or a new version of the payload) and migrate consumers. |
| Add a new job type | Add to the discriminated union. No breaking change — workers that don't handle the new name get a default/error case. |

### Semver for internal packages

Internal packages use `workspace:*` so semver ranges don't apply at install time. However, version the `@shared` package semantically for changelog purposes:

- **MAJOR (1.0.0 → 2.0.0):** Breaking type change that requires consumer code changes (field removed, field renamed, type changed, function signature changed)
- **MINOR (1.0.0 → 1.1.0):** New types, new optional fields, new schema added. Backward compatible.
- **PATCH (1.1.0 → 1.1.1):** Doc fixes, type comment updates, internal reorganization with no public API change.

### Deprecation pattern

```ts
// @shared/src/types/user.ts

/** @deprecated Use `UserProfile` instead — `User` will be removed in v2.0.0 */
export interface User {
  id: string;
  name: string;
  /** @deprecated Use `UserProfile.email` instead */
  email: string;
}

export interface UserProfile {
  id: string;
  name: string;
  email: string;
  avatarUrl?: string;
}
```

TSDoc `@deprecated` triggers strikethrough in VS Code and warnings in most TypeScript language servers. It's the universal signal for "migrate away from this."

### Breaking change strategy across apps

When a breaking type change is unavoidable:

1. **Create the new type** (e.g., `UserProfileV2`) alongside the old one in `@shared`
2. **Update one consumer at a time** — backend, then frontend, then workers
3. **Each consumer deploys independently** — the old type still exists for consumers that haven't updated
4. **Remove the old type** only after all consumers have migrated and deployed

This works because `@shared` is JIT-compiled by each consumer — there's no shared `dist/` that gets overwritten. Each app compiles `@shared` at its own cadence.

## Antipatterns and common mistakes

| Antipattern | Why it's wrong | Fix |
|---|---|---|
| Apps importing from each other (`@app/frontend` imports from `@app/backend`) | Creates hidden coupling. Breaks Turborepo task graph. Makes deployment ordering fragile. | Move the shared type to `@shared`. Both apps import from it. |
| `@shared` importing from any app | Circular dependency. `@shared` is the leaf — it has zero app dependencies. | Invert the dependency. If the type needs app code, lift the abstraction. |
| Exporting Prisma types from `@shared` | Leaks database schema to all consumers. Frontend shouldn't know about database columns. | Map Prisma types to DTOs in backend. Export the DTO from `@shared`. |
| `export * from "./module"` in barrel | Invisible public API. Name collisions. Poor tree-shaking. | Explicit named re-exports with `export type` for type-only. |
| Duplicating types across apps instead of sharing | Divergent types. Frontend assumes one shape, backend validates another. | Move to `@shared`. One source of truth. |
| Putting runtime utilities in `@shared` instead of types | `@shared` becomes a utility dumping ground. Consumers pull heavy runtime code for a single type. | Types and light constants only. Heavy runtime logic goes in `@shared/utils` (separate package, only import if needed). |
| Schema without version field on job payloads | Worker can't detect or handle version mismatches during rolling deploys. | Every job payload type includes `version: number`. |
| Re-exporting Zod from `@shared` instead of peer dep | Couples Zod version to `@shared` version. Consumer can't upgrade Zod independently. | Zod is a peer dependency. Consumer installs its own. |
| TypeScript `enum` instead of string union or Zod enum | Emits runtime JS. Doesn't tree-shake. Can't compose with discriminated unions. | String literal union for compile-time; `z.enum()` for runtime validation. |
| Wildcard `import *` from `@shared` | Pulls every schema and type into the consumer's bundle. | Named imports from subpath exports: `import { UserSchema } from "@shared/schemas"`. |

## Audit-flagged gaps — resolution

### 1. One @shared package vs multiple — RESOLVED

**Finding:** Turborepo recommends "one purpose per package" but "contract layer" is a single purpose. The official `basic` example uses a single `@repo/ui` package with internal structure, not micro-packages. The "second time" rule applies: extract only when duplication or coupling friction is real.

**Source:** Turborepo docs — internal packages guide, `github.com/vercel/turborepo/tree/main/examples/basic`

### 2. Zod as peer dependency — RESOLVED

**Finding:** Best practice is Zod in both `peerDependencies` and `devDependencies` of `@shared`. Consumers install Zod themselves. Use pnpm catalogs to enforce a single version across the workspace. Do NOT re-export Zod from `@shared` — it couples the Zod version to the shared package.

**Source:** Zod Library Authors guide (`zod.dev/library-authors`), Zod v4 release notes

### 3. Conditional exports for TypeScript-only internal packages — RESOLVED

**Finding:** For JIT (internal-only) packages, the `exports` field pointing to `.ts` source is sufficient and simpler than compiled conditional exports. The `exports` field is still necessary for: (a) `moduleResolution: "nodenext"`, (b) subpath exports (`@shared/schemas`). For compiled packages, use `{ "types": "./src/index.ts", "default": "./dist/index.js" }` — the `types` condition can still point to source for IDE support.

**Source:** Turborepo internal packages documentation, TypeScript handbook on `exports` field

### 4. Job payload versioning — RESOLVED

**Finding:** The dominant pattern is: never break backward compatibility. Add optional fields only. Use a `version` field in JSON payloads for version detection. For unavoidable breaking changes: create a new job name, deploy workers that handle both old and new, switch the backend, drain the old queue, then remove the old handler. Schema validation at the worker boundary catches version mismatches during rolling deployments.

**Source:** Message queue community patterns (schema evolution best practices), BullMQ v5 API docs

### 5. Type-only imports in barrel exports — RESOLVED

**Finding:** `export type { ... }` is required for type-only re-exports with `verbatimModuleSyntax` and `isolatedDeclarations`. For mixed re-exports: `export { SomeFn, type SomeType }`. This is the 2025-2026 standard enforced by ESLint rules `@typescript-eslint/consistent-type-exports` and `consistent-type-imports`.

**Source:** TypeScript 5.5 release notes (`isolatedDeclarations`), `verbatimModuleSyntax` documentation, ESLint `@typescript-eslint` plugin docs

## Reference snippets

### @shared/package.json (JIT, annotated)

```jsonc
{
  "name": "@shared",
  "version": "0.0.0",
  "private": true,
  "type": "module",
  "exports": {
    ".": { "types": "./src/index.ts", "default": "./src/index.ts" },
    "./schemas": { "types": "./src/schemas/index.ts", "default": "./src/schemas/index.ts" },
    "./types": { "types": "./src/types/index.ts", "default": "./src/types/index.ts" },
    "./errors": { "types": "./src/errors/index.ts", "default": "./src/errors/index.ts" },
    "./jobs": { "types": "./src/jobs/index.ts", "default": "./src/jobs/index.ts" }
  },
  "peerDependencies": { "zod": "^4.0.0" },
  "devDependencies": { "zod": "^4.0.0", "typescript": "catalog:" }
}
```

### @shared/tsconfig.json

```jsonc
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "${configDir}/dist",
    "rootDir": "${configDir}/src",
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "declaration": true,
    "declarationMap": true,
    "isolatedDeclarations": true,
    "verbatimModuleSyntax": true
  },
  "include": ["src"],
  "exclude": ["node_modules", "dist"]
}
```

### Schema + type co-location (@shared/src/schemas/user.ts)

```ts
import { z } from "zod/v4";

export const UserSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1),
  email: z.email(),
  role: z.enum(["admin", "user", "guest"]),
  createdAt: z.date(),
});
export type User = z.infer<typeof UserSchema>;

export const CreateUserSchema = z.object({
  name: z.string().min(1),
  email: z.email(),
  role: z.enum(["admin", "user", "guest"]).default("user"),
});
export type CreateUserInput = z.input<typeof CreateUserSchema>;
export type CreateUserOutput = z.output<typeof CreateUserSchema>;

export const UpdateUserSchema = z.object({
  name: z.string().min(1).optional(),
  email: z.email().optional(),
  role: z.enum(["admin", "user", "guest"]).optional(),
});
export type UpdateUserInput = z.input<typeof UpdateUserSchema>;
```

### Job payload types with discriminated union (@shared/src/jobs/index.ts)

```ts
import { z } from "zod/v4";

// ── Job name discriminated union ──
export const JOB_NAMES = ["send-email", "send-notification"] as const;
export type JobName = (typeof JOB_NAMES)[number];

// ── Payload types ──
export interface EmailJobPayload {
  version: 1;
  jobName: "send-email";
  to: string;
  subject: string;
  body: string;
  templateId?: string;
}

export interface NotificationJobPayload {
  version: 1;
  jobName: "send-notification";
  userId: string;
  title: string;
  message: string;
  channel: "push" | "in-app";
}

export type JobPayload = EmailJobPayload | NotificationJobPayload;

// ── Validation schemas ──
export const EmailJobPayloadSchema = z.object({
  version: z.literal(1),
  jobName: z.literal("send-email"),
  to: z.string().email(),
  subject: z.string().min(1),
  body: z.string().min(1),
  templateId: z.string().optional(),
});

export const NotificationJobPayloadSchema = z.object({
  version: z.literal(1),
  jobName: z.literal("send-notification"),
  userId: z.string().uuid(),
  title: z.string().min(1),
  message: z.string().min(1),
  channel: z.enum(["push", "in-app"]),
});
```

### Utility type wrappers (@shared/src/types/utility.ts)

```ts
export interface Paginated<T> {
  items: T[];
  total: number;
  page: number;
  pageSize: number;
  hasMore: boolean;
}

export interface ApiResponse<T> {
  success: true;
  data: T;
}

export interface ApiError {
  success: false;
  error: {
    code: string;
    message: string;
    details?: unknown;
  };
}

export type ApiResult<T> = ApiResponse<T> | ApiError;

export type Nullable<T> = T | null;

export type DeepPartial<T> = {
  [P in keyof T]?: T[P] extends object ? DeepPartial<T[P]> : T[P];
};
```

### Barrel export (@shared/src/index.ts)

```ts
// Schemas — mixed value + type exports
export {
  UserSchema, type User,
  CreateUserSchema, type CreateUserInput, type CreateUserOutput,
  UpdateUserSchema, type UpdateUserInput,
} from "./schemas/user";

export {
  PostSchema, type Post,
  CreatePostSchema, type CreatePostInput, type CreatePostOutput,
} from "./schemas/post";

export {
  ApiResponseSchema, type ApiResponse,
  PaginatedResponseSchema, type PaginatedResponse,
} from "./schemas/api";

// Types — type-only exports (no runtime value)
export type { Paginated, ApiError, ApiResult, Nullable, DeepPartial } from "./types/utility";
export type { Breadcrumb, NavItem } from "./types/navigation";

// Enums — type-only exports (string literal unions)
export type { UserRole } from "./schemas/user";
export type { TaskStatus } from "./enums";

// Errors — type-only exports
export type { ErrorCode, ErrorResponse } from "./errors";

// Jobs — type-only exports for payload types, value exports for schemas
export {
  EmailJobPayloadSchema,
  NotificationJobPayloadSchema,
  JOB_NAMES,
} from "./jobs";
export type { EmailJobPayload, NotificationJobPayload, JobPayload, JobName } from "./jobs";
```

## Sources used

- [Turborepo — Internal Packages](https://turborepo.dev/docs/core-concepts/internal-packages) — official
- [Turborepo — basic example](https://github.com/vercel/turborepo/tree/main/examples/basic) — official
- [Zod Library Authors guide](https://zod.dev/library-authors) — official
- [Node.js — Package entry points (exports field)](https://nodejs.org/api/packages.html#package-entry-points) — official
- [TypeScript — `isolatedDeclarations` (TS 5.5 release notes)](https://devblogs.microsoft.com/typescript/announcing-typescript-5-5/#isolated-declarations) — official
- [TypeScript — `verbatimModuleSyntax`](https://www.typescriptlang.org/tsconfig/#verbatimModuleSyntax) — official
- [BullMQ v5 — Queue typed generics](https://api.docs.bullmq.io/classes/v5.Queue.html) — official
- [BullMQ — Advanced typed queue pattern with discriminated unions](https://github.com/taskforcesh/bullmq/discussions/3509) — community
- [Turborepo — "You might not need TypeScript project references"](https://turborepo.dev/docs/guides/you-might-not-need-typescript-project-references) — official
- [pnpm — Catalog protocol](https://pnpm.io/catalogs) — official
- [madge — circular dependency detection](https://github.com/pahen/madge) — tool
- [ESLint — `no-restricted-imports`](https://eslint.org/docs/latest/rules/no-restricted-imports) — official
- [TypeScript — `export type` for type-only re-exports](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-8.html#type-only-exports-and-imports) — official
