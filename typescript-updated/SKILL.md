# TypeScript Stack — Skill Index

## Architecture
Multi-app monorepo with independently deployable units: `/apps/frontend`, `/apps/backend`, `/apps/workers`.
Shared code in `/packages/shared-types` and `/packages/db`.
Stack: TypeScript 5.5+ · Next.js (App Router) · Fastify · BullMQ · Drizzle ORM 0.45.x · Zod · Vitest · Turborepo 2.x · pnpm workspaces · Docker.

Read the relevant skill file(s) before producing code or config for any domain below.
Skills are organized bottom-up: foundational skills (D1–D3) inform all others.

---

## Skill Registry

### Layer 1 — Foundational
| ID  | File                                                 | Read when the task involves...                                                                      |
|-----|------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| D1  | typescript-updated/D1-typescript-language-core.md    | Generics, conditional/mapped types, utility types, `satisfies`, type narrowing, inferred predicates, `.d.ts` |
| D2  | typescript-updated/D2-typescript-compiler-tooling.md | `tsconfig.json`, `moduleResolution`, project references, path aliases, `tsup`, `esbuild`, `swc`, build output |
| D3  | typescript-updated/D3-multi-app-monorepo.md          | `turbo.json`, Turborepo tasks, `pnpm-workspace.yaml`, `turbo prune`, monorepo scripts, workspace deps |

### Layer 2 — Cross-cutting
| ID  | File                                                 | Read when the task involves...                                                                      |
|-----|------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| D4  | typescript-updated/D4-zod.md                         | Zod schemas, `z.infer`, `z.input`, `z.output`, refinements, transforms, Zod in `@shared`           |
| D5  | typescript-updated/D5-shared-types-contracts.md      | `@packages/shared-types` design, DTOs, job payload types, barrel exports, type contracts between apps |
| D6  | typescript-updated/D6-testing.md                     | Vitest workspace config, testing Fastify routes (`inject`), testing job handlers, `expectTypeOf`, fixtures |
| D7  | typescript-updated/D7-linting-formatting.md          | `eslint.config.js` flat config, `typescript-eslint` v8, Prettier, `no-restricted-imports`, `import/no-cycle` |

### Layer 3 — Apps
| ID  | File                                                 | Read when the task involves...                                                                      |
|-----|------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| D8  | typescript-updated/D8-backend-app-fastify.md         | Fastify routes, plugins, `fp()`, typed decorators, Zod type provider, error handler, graceful shutdown, env validation |
| D9  | typescript-updated/D9-frontend-app.md                | Next.js App Router, Server Components, Server Actions, `useActionState`, React 19, caching model, Vite SPA, RHF |
| D10 | typescript-updated/D10-workers-app.md                | BullMQ queues, typed job handlers, `Queue`/`Worker` generics, graceful shutdown, dead-letter, node-cron |

### Support
| ID  | File                                                 | Read when the task involves...                                                                      |
|-----|------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| S1  | typescript-updated/S1-data-layer-drizzle-orm.md      | Drizzle ORM schema, `pgTable`, `InferSelectModel`, queries, relations, transactions, `drizzle-kit`, repositories |
| S2  | typescript-updated/S2-containerization.md            | Dockerfiles, `turbo prune` in Docker, Next.js standalone output, `docker-compose.yml`, healthchecks, migration container |

---

## Multi-skill Rules

Read more than one skill when a task crosses domain boundaries:

  Typed Fastify route using @shared Zod schema       → D4 + D8
  Server Action with Zod validation                  → D4 + D9
  Job payload type shared between backend/workers    → D5 + D8 + D10
  New shared type in @packages/shared-types          → D4 + D5
  Drizzle schema + repository for a new domain       → S1 (+ D5 if types go to @shared)
  New Dockerfile or docker-compose change            → S2 (+ D3 if turbo prune is involved)
  Monorepo-wide ESLint rule change                   → D7 (+ D3 if turbo lint task is affected)
  Test for a Fastify route                           → D6 + D8
  Test for a job handler                             → D6 + D10
  New workspace package in /packages/*               → D3 + D5

## Priority Rule

If a task involves advanced TypeScript patterns (generics, type inference, narrowing) that
underpin another domain, read D1 first — even if the primary domain is D8, D9 or D10.
D1 is the foundation; the other skills assume it.

## Version Reference

  TypeScript        5.5+           D1, D2
  Turborepo         2.x            D3, S2
  Zod               see D4         D4
  Vitest            2.x            D6
  ESLint            v9 flat config D7
  typescript-eslint v8             D7
  Fastify           5.x            D8
  Next.js           15/16          D9
  React             19             D9
  BullMQ            current        D10
  Drizzle ORM       0.45.x stable  S1
  drizzle-kit       0.45.x stable  S1
  Node.js           LTS            S2
  pnpm              9.x            D3, S2