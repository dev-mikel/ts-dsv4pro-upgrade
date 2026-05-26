# Skill: TypeScript — D6: Testing per App

## Summary

Covers Vitest 4.x workspace (projects) configuration, per-app test environments, mocking strategies in a pnpm monorepo, Fastify `inject()` route testing patterns, BullMQ job handler isolation, type-level testing with `expect-type`, and shared typed fixtures.

**Vitest version documented:** 4.1.7 (stable, May 2026). Vitest 5.0.0 beta exists but is not yet stable.
**Audit risk:** Moderate (resolved to Low — all version-sensitive configs verified against 4.x docs).
**Dependencies:** D1 (TypeScript), D2 (tsconfig/paths), D3 (Turborepo), D4 (Zod), D5 (@shared types) — all completed.
**Knowledge cutoff:** May 2026.

---

## Version landscape

| Tool | Version | Notes |
|---|---|---|
| Vitest | **4.1.7** (stable) | 5.0.0-beta.3 available. 4.0 removed `workspace`/`defineWorkspace` (deprecated in 3.2). |
| expect-type | **1.3.0** | `.toMatchTypeOf` deprecated → use `.toExtend`. Added `.map` in 1.2.0. |
| TypeScript | 5.5+ | Consistent with D1. `${configDir}` macro available for path resolution in nested tsconfigs. |
| Node.js | ≥20.0.0 | Required by Vitest 4.x. |

### Breaking changes from Vitest 3.x → 4.x

1. **`workspace` removed.** `vitest.workspace.ts` files and `defineWorkspace()` no longer exist. Use `test.projects` in the root `vitest.config.ts`.
2. **Only `vitest.config.*` / `vite.config.*` filenames** are recognized as project configs.
3. **AST-based coverage remapping** is now the default (replaces `v8-to-istanbul`). `coverage.ignoreEmptyLines` removed.
4. **Browser mode** graduated from experimental to stable. Providers are separate packages (`@vitest/browser-playwright`, etc.).

---

## Vitest workspace (projects) setup

### Root config: `vitest.config.ts`

In 4.x, the projects array lives directly in the root config. There is no separate workspace file.

```ts
// vitest.config.ts (repo root)
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    // Global settings (not per-project) — reporters, coverage config go here
    reporters: process.env.CI ? ['json', 'html'] : ['default'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
      reportsDirectory: './coverage',
    },

    projects: [
      // Glob: auto-discovers vitest.config.ts in each matched directory
      'apps/backend',
      'apps/frontend',
      'apps/workers',
      'packages/*',

      // Inline project configs also work:
      // {
      //   extends: true,
      //   test: {
      //     name: 'typecheck',
      //     include: ['**/*.test-d.ts'],
      //   },
      // },
    ],
  },
})
```

### Per-project config: `vitest.config.ts`

Each app/package uses `defineProject` (not `defineConfig`) for type safety — it flags root-only options like `reporters` and `coverage` as errors.

```ts
// apps/backend/vitest.config.ts
import { defineProject } from 'vitest/config'
import path from 'node:path'

export default defineProject({
  resolve: {
    alias: {
      '@shared': path.resolve(__dirname, '../../packages/shared/src'),
    },
  },
  test: {
    name: 'backend',
    environment: 'node',
    globals: true,
    setupFiles: ['./src/test/setup.ts'],
    include: ['src/**/*.{test,spec}.ts'],
    // Pool: 'threads' is default for node; use 'forks' if native modules need isolation
  },
})
```

```ts
// apps/frontend/vitest.config.ts
import { defineProject } from 'vitest/config'
import path from 'node:path'

export default defineProject({
  resolve: {
    alias: {
      '@shared': path.resolve(__dirname, '../../packages/shared/src'),
    },
  },
  test: {
    name: 'frontend',
    environment: 'jsdom',          // DOM environment for component tests
    globals: true,
    setupFiles: ['./src/test/setup.ts'],
    include: ['src/**/*.{test,spec}.{ts,tsx}'],
    css: true,                     // process CSS imports (or false to skip)
  },
})
```

```ts
// apps/workers/vitest.config.ts
import { defineProject } from 'vitest/config'
import path from 'node:path'

export default defineProject({
  resolve: {
    alias: {
      '@shared': path.resolve(__dirname, '../../packages/shared/src'),
    },
  },
  test: {
    name: 'workers',
    environment: 'node',
    globals: true,
    include: ['src/**/*.{test,spec}.ts'],
  },
})
```

### Key config rules

- **`extends: true`** — inline project inherits all root config (plugins, pool, aliases, etc.). False by default.
- **Root config is NOT a project** — it provides global settings only. Projects must be explicitly listed.
- **Project names must be unique** — used for `--project` filtering and reporter output.
- **Root-only options** (disallowed in `defineProject`): `coverage`, `reporters`, `resolveSnapshotPath`.
- **`environmentMatchGlobs`** can set environment per file pattern instead of per project:
  ```ts
  test: {
    environmentMatchGlobs: [
      ['**/*.dom.test.ts', 'jsdom'],
      ['**/*.node.test.ts', 'node'],
    ],
  }
  ```

### Running tests

```bash
# All projects (CI)
vitest run

# All projects (dev, watch mode)
vitest

# Single project by name
vitest --project backend

# Multiple specific projects
vitest --project backend --project workers

# With Turborepo (per-package, CI caching)
turbo run test

# Typecheck tests (runs tsc --noEmit on test files)
vitest run --typecheck
```

### Turborepo integration (D3 alignment)

Two approaches coexist. The Turborepo `with-vitest` example (vercel/turbo/examples/with-vitest) and community consensus both recommend a **hybrid**:

| Context | Command | Why |
|---|---|---|
| **CI** | `turbo run test` | Per-package caching. Only re-runs tests for changed packages. |
| **Local dev** | `vitest` (root) | Single process, unified watch, merged coverage. |

**`turbo.json` test task:**

```json
{
  "tasks": {
    "test": {
      "dependsOn": ["transit", "@repo/vitest-config#build"],
      "inputs": ["$TURBO_DEFAULT$", "vitest.config.ts"],
      "outputs": ["coverage/coverage-final.json"]
    }
  }
}
```

- `dependsOn: ["transit"]` — ensures internal package builds complete before tests (D3 alignment).
- `dependsOn: ["@repo/vitest-config#build"]` — if using a shared vitest config package.
- `inputs` — include `vitest.config.ts` so cache invalidates when test config changes.

**Root `package.json` scripts:**

```json
{
  "scripts": {
    "test": "turbo run test",
    "test:watch": "vitest",
    "test:ci": "vitest run --coverage",
    "test:typecheck": "vitest run --typecheck"
  }
}
```

The `turbo run test` path works because each app's `package.json` has its own `"test": "vitest run"` script. Turbo runs it per package, and Vitest picks up the local `vitest.config.ts` via `defineProject`.

### Coverage config and merging

**Provider choice — v8 is default and recommended:**
- `v8`: native V8 coverage, zero transpilation overhead, the default in 4.x.
- `istanbul`: traditional code instrumentation, needed only for specific edge cases.

**Merging across projects:**

Vitest 4.x uses AST-based coverage remapping (replaced `v8-to-istanbul`). When running `vitest run --coverage` from root with multiple projects, coverage is merged automatically into a single report. Known issues:

- **Duplicate entries possible** when projects use different transpilers (SWC vs Babel) — ensure consistent transpilation across projects or use recent 4.x patches (PR #9480 fixed the core merging logic).
- **`--project` filtering** (e.g., `vitest run --project backend --coverage`) correctly scopes coverage to that project only (fixed in PR #7885).

For the Turborepo per-package path (CI), coverage files must be merged manually:

```bash
# Each package outputs coverage/coverage-final.json
# Merge with nyc or a custom script
npx nyc report --reporter=html --temp-dir=./coverage
```

---

## TypeScript config for tests

### Pattern: separate tsconfig for test files

Each app gets a `tsconfig.test.json` that extends the app's main tsconfig and adds Vitest-specific types:

```json
// apps/backend/tsconfig.test.json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "types": ["vitest/globals"]
  },
  "include": ["src/**/*.test.ts", "src/test/**/*.ts"]
}
```

Why a separate tsconfig:
1. `"types": ["vitest/globals"]` adds `describe`, `it`, `expect`, `vi`, etc. to the global scope — but only for test files. Adding it to the main tsconfig would leak test types into production code.
2. Separation means `tsc --noEmit` on the main tsconfig doesn't typecheck test files (faster production builds).
3. Vitest's `--typecheck` flag targets the test tsconfig.

### vitest/globals vs /// reference

Two ways to resolve Vitest global types:

| Method | Where | Notes |
|---|---|---|
| `"types": ["vitest/globals"]` | `tsconfig.test.json` `compilerOptions` | Preferred. Clean, project-wide. |
| `/// <reference types="vitest/globals" />` | Top of each test file | Fine for one-off, tedious at scale. |

**Important:** If `@jest/globals` or `@types/jest` are installed, they conflict with `vitest/globals` — remove them.

### Path alias resolution in tests

Aliases configured in `resolve.alias` (Vite-level) in each app's `vitest.config.ts` are consistent with the D2 tsconfig paths. The `vitest.config.ts` alias must match `tsconfig.json` `compilerOptions.paths`:

```ts
// vitest resolve.alias
resolve: {
  alias: {
    '@shared': path.resolve(__dirname, '../../packages/shared/src'),
    '@backend': path.resolve(__dirname, './src'),
  },
}
```

```json
// tsconfig.json compilerOptions.paths (D2)
{
  "compilerOptions": {
    "paths": {
      "@shared/*": ["../../packages/shared/src/*"],
      "@backend/*": ["./src/*"]
    }
  }
}
```

Vitest resolves `resolve.alias` at transform time; TypeScript uses `compilerOptions.paths` for typechecking. Both must agree.

---

## Mocking strategy

### vi.mock() with workspace aliases

Mocking `@shared` packages requires explicit factory functions. Bare alias mocking is unreliable:

```ts
// ❌ Fails — bare alias resolution is inconsistent in workspace mode
vi.mock('@shared/schemas')

// ✅ Works — factory function provides the mock inline
vi.mock('@shared/schemas', () => ({
  userSchema: z.object({ id: z.string().uuid(), email: z.string().email() }),
  createUserSchema: z.object({ email: z.string().email() }),
}))

// ✅ Works — dynamic import from __mocks__ directory
vi.mock('@shared/schemas', () => import('../../__mocks__/@shared/schemas'))
```

**Rule:** always pass a factory function (`() => ({ ... })`) to `vi.mock()` when the import path is a workspace alias. The bare form relies on `__mocks__` auto-discovery, which is broken for pnpm `workspace:*` dependencies (Vitest issue #7708).

**Alias resolution fix:** if aliases fail during test execution despite correct `resolve.alias`, add them to `test.alias` as well:

```ts
test: {
  alias: {
    '@shared': path.resolve(__dirname, '../../packages/shared/src'),
  },
},
```

### vi.mock() vs vi.spyOn()

| Technique | When to use |
|---|---|
| `vi.mock(module, factory)` | Replace an entire module. Use for `@shared` packages, external SDKs, database clients. |
| `vi.spyOn(obj, method)` | Observe or override one method while keeping the rest real. Use for logging, metrics, partial overrides. |

```ts
// vi.mock — replace whole module
vi.mock('@prisma/client', () => ({
  PrismaClient: vi.fn(() => mockPrisma),
}))

// vi.spyOn — override one method, keep rest real
vi.spyOn(console, 'error').mockImplementation(() => {})
vi.spyOn(metrics, 'increment').mockResolvedValue(undefined)
```

### **mocks** directory pattern

Vitest supports Jest-style `__mocks__` directories for auto-discovery. However, they do not work with pnpm `workspace:*` packages (the symlink resolution path doesn't match `node_modules` expectations). **For internal `@shared` packages, prefer inline factory functions** in `vi.mock()` calls.

The `__mocks__` pattern remains useful for third-party dependencies (e.g., `__mocks__/bullmq.ts`, `__mocks__/@prisma/client.ts`):

```
apps/backend/
  src/
    __mocks__/
      bullmq.ts
      @prisma/
        client.ts
```

### Mocking Prisma client (no database)

**Recommended: dependency injection** — inject a mock PrismaClient rather than mocking the module:

```ts
import { DeepMockProxy, mockDeep } from 'vitest-mock-extended'
import type { PrismaClient } from '@prisma/client'

type TestContext = {
  prisma: DeepMockProxy<PrismaClient>
}

const createTestContext = (): TestContext => ({
  prisma: mockDeep<PrismaClient>(),
})

// In test:
const ctx = createTestContext()
ctx.prisma.user.findUnique.mockResolvedValue({ id: '1', email: 'test@x.com' })

const result = await getUserById('1', ctx.prisma)
expect(result).toEqual({ id: '1', email: 'test@x.com' })
```

This requires production code to accept `prisma` as a parameter (or via request-scoped context in Fastify). It avoids module mocking entirely — faster, more explicit, no `vi.mock()` side effects.

**Alternative:** module-level mock via singleton pattern (Prisma's own recommended approach adapted for Vitest):

```ts
// src/test/singleton.ts
import { PrismaClient } from '@prisma/client'
import { mockDeep, mockReset, DeepMockProxy } from 'vitest-mock-extended'
import { vi, beforeEach } from 'vitest'

vi.mock('@prisma/client', () => ({
  PrismaClient: vi.fn(() => prismaMock),
}))

export const prismaMock = mockDeep<PrismaClient>()

beforeEach(() => {
  mockReset(prismaMock)
})
```

### Mocking BullMQ Queue (no Redis)

Same pattern: dependency injection extracts business logic from queue mechanics.

```ts
// Job handler file (apps/workers/src/jobs/process-order.ts)
import type { OrderJobPayload } from '@shared/schemas'

// Pure business logic — testable without BullMQ
export async function processOrder(
  payload: OrderJobPayload,
  deps: { db: PrismaClient; inventoryApi: InventoryClient },
): Promise<void> {
  const order = await deps.db.order.findUniqueOrThrow({ where: { id: payload.orderId } })
  await deps.inventoryApi.reserve(order.items)
}

// BullMQ wrapper — thin, tested only in integration
export const processOrderWorker = new Worker('orders', async (job) => {
  await processOrder(job.data, { db: prisma, inventoryApi: inventoryClient })
})
```

Test the handler in isolation:

```ts
it('reserves inventory for valid order', async () => {
  const db = mockDeep<PrismaClient>()
  db.order.findUniqueOrThrow.mockResolvedValue({ id: '1', items: [{ sku: 'A', qty: 2 }] })
  const inventoryApi = { reserve: vi.fn().mockResolvedValue(undefined) }

  await processOrder({ orderId: '1' }, { db, inventoryApi })

  expect(inventoryApi.reserve).toHaveBeenCalledWith([{ sku: 'A', qty: 2 }])
})
```

If direct queue mocking is needed for integration-level tests:

```ts
// src/__mocks__/bullmq.ts
import { vi } from 'vitest'

export const mockQueueAdd = vi.fn().mockResolvedValue({ id: 'mock-job-1' })

export const Queue = vi.fn().mockImplementation(() => ({
  add: mockQueueAdd,
  close: vi.fn().mockResolvedValue(undefined),
}))

export const Worker = vi.fn()
export { mockQueueAdd }
```

---

## Testing Fastify routes

> Builds on D8 context without implementing D8. Establishes the `inject()` pattern only.

### app.inject() — the primary mechanism

Fastify's `app.inject()` (powered by `light-my-request`) fakes an HTTP request through the full route lifecycle without opening a real socket. It's the only correct way to test Fastify routes — never call `app.listen()` in tests.

```ts
import { buildTestApp } from '../src/test/app'

let app: FastifyInstance

beforeAll(async () => {
  app = await buildTestApp()
  await app.ready()
})

afterAll(async () => {
  await app.close()
})

it('returns 200 for GET /health', async () => {
  const res = await app.inject({ method: 'GET', url: '/health' })
  expect(res.statusCode).toBe(200)
  expect(res.json()).toEqual({ status: 'ok' })
})
```

**Inject options** (`InjectOptions` type):
- `method`, `url`, `query`, `payload`, `headers`, `cookies`, `authority`

**Response** (`LightMyRequestResponse`):
- `.statusCode`, `.body` (string), `.json()` (parsed), `.headers`, `.cookies`

### Test app factory function

Each app should expose a `buildTestApp()` that constructs a Fastify instance with only what tests need. It's a simplified version of the production `buildApp()`, but still registers plugins required for the routes under test.

```ts
// apps/backend/src/test/app.ts
import Fastify, { FastifyInstance } from 'fastify'
import { routes } from '../routes'

export async function buildTestApp(): Promise<FastifyInstance> {
  const app = Fastify({
    logger: false,           // silence logs in tests
  })

  // Register only plugins needed by routes under test
  // Do NOT register auth plugin — bypass or use mock headers instead
  // Do NOT register rate-limiting plugin
  // Do NOT register CORS plugin (inject bypasses it anyway)

  await app.register(routes)

  return app
}
```

### Typed request/response in inject calls

Fastify's `inject()` returns `LightMyRequestResponse`. For typed response bodies, cast `res.json()`:

```ts
import type { UserResponse } from '@shared/schemas'

const res = await app.inject({ method: 'GET', url: '/users/1' })
const body = res.json() as UserResponse
expect(body.email).toBe('test@example.com')
```

### Bypassing auth in tests

Three approaches, in order of preference:

1. **Don't register the auth plugin** in `buildTestApp()`. Routes that check `request.user` will fail — only test auth-independent routes this way.
2. **Inject mock auth headers** that the auth plugin recognizes as valid without real verification:
   ```ts
   const res = await app.inject({
     method: 'GET',
     url: '/users/me',
     headers: { authorization: 'Bearer test-token' },
   })
   ```
3. **Register a mock auth decorator** that sets `request.user` directly:
   ```ts
   app.decorateRequest('user', null)
   app.addHook('onRequest', async (req) => {
     req.user = { id: 'test-user', role: 'admin' }
   })
   ```

### Testing error responses

Verify error shape matches @shared error types from D5:

```ts
import type { ApiError } from '@shared/types'

it('returns 404 for missing user', async () => {
  const res = await app.inject({ method: 'GET', url: '/users/nonexistent' })
  expect(res.statusCode).toBe(404)
  const body = res.json() as ApiError
  expect(body).toMatchObject({
    statusCode: 404,
    error: 'Not Found',
    message: expect.any(String),
  })
})
```

---

## Testing job handlers (workers)

### Isolate handler from BullMQ infrastructure

The core pattern: **job handler = pure function + thin BullMQ wrapper.** The pure function is unit-tested; the BullMQ integration is tested only in integration/e2e.

```ts
// apps/workers/src/jobs/send-email.ts
import type { EmailJobPayload } from '@shared/schemas'

export async function handleSendEmail(
  payload: EmailJobPayload,
  deps: { emailService: EmailService },
): Promise<void> {
  if (!payload.to || !payload.subject) {
    throw new Error('Invalid email payload')
  }
  await deps.emailService.send({
    to: payload.to,
    subject: payload.subject,
    body: payload.body,
  })
}
```

### Test with typed job payload

Use D5-exported types for compile-time safety:

```ts
import { describe, it, expect, vi } from 'vitest'
import { handleSendEmail } from './send-email'
import type { EmailJobPayload } from '@shared/schemas'

it('sends email for valid payload', async () => {
  const emailService = { send: vi.fn().mockResolvedValue(undefined) }
  const payload: EmailJobPayload = {
    to: 'user@example.com',
    subject: 'Welcome',
    body: 'Hello!',
  }

  await handleSendEmail(payload, { emailService })

  expect(emailService.send).toHaveBeenCalledWith(payload)
})

it('throws for invalid payload', async () => {
  const emailService = { send: vi.fn() }
  const payload = { to: '', subject: '', body: '' } as EmailJobPayload

  await expect(handleSendEmail(payload, { emailService })).rejects.toThrow('Invalid email payload')
  expect(emailService.send).not.toHaveBeenCalled()
})
```

### Testing retry logic and error behavior

The handler function should throw on errors that warrant a retry. BullMQ's retry mechanism (retry count, backoff) is BullMQ's concern, not the handler's:

```ts
it('throws retryable error on transient API failure', async () => {
  const emailService = { send: vi.fn().mockRejectedValue(new Error('Network timeout')) }
  const payload: EmailJobPayload = { to: 'a@b.com', subject: 'S', body: 'B' }

  // Handler throws → BullMQ catches → BullMQ applies retry policy
  await expect(handleSendEmail(payload, { emailService })).rejects.toThrow('Network timeout')
})
```

The retry policy (max attempts, backoff) is tested at the BullMQ integration layer or via e2e, not in unit tests.

---

## Type-level testing

### expect-type API (1.3.0)

```ts
import { expectTypeOf } from 'expect-type'

// Strict equality — both types must be exactly the same
expectTypeOf({ a: 1 }).toEqualTypeOf<{ a: number }>()

// Structural "is-a" check — replaces deprecated .toMatchTypeOf
expectTypeOf<{ a: number; b: string }>().toExtend<{ a: number }>()

// Strict object subset — matches keys and types exactly (catches readonly mismatches)
expectTypeOf({ a: 1, b: 'x' } as const).toMatchObjectType<{ a: number; b: string }>()

// Primitive checks
expectTypeOf<number>().toBeNumber()
expectTypeOf<string>().toBeString()
expectTypeOf<never>().toBeNever()
expectTypeOf<unknown>().toBeUnknown()
expectTypeOf<any>().toBeAny()
expectTypeOf<void>().toBeVoid()
expectTypeOf<() => void>().toBeFunction()
expectTypeOf<{ x: number }>().toBeObject()
expectTypeOf<string[]>().toBeArray()

// Function parameter/return inspection
expectTypeOf(myFn).parameter(0).toBeString()
expectTypeOf(myFn).parameters.toEqualTypeOf<[name: string, age: number]>()
expectTypeOf(myFn).returns.toEqualTypeOf<Promise<User>>()

// Promise unwrapping
expectTypeOf(Promise.resolve(42)).resolves.toBeNumber()

// Branded types
type BrandedId = string & { __brand: 'UserId' }
expectTypeOf<BrandedId>().branded.toEqualTypeOf<BrandedId>()

// Negation
expectTypeOf(42).not.toBeString()

// Map (1.2.0+)
expectTypeOf([1, 2, 3]).map.toEqualTypeOf<number[]>()
```

### assertType (built into Vitest)

Simpler API for basic type assertions:

```ts
import { assertType } from 'vitest'

const answer = 42
assertType<number>(answer)
// @ts-expect-error answer is not a string
assertType<string>(answer)
```

### expect-type vs tsd

| Factor | expect-type (Vitest) | tsd |
|---|---|---|
| **Where tests live** | Inline in `*.test.ts` | Separate `*.test-d.ts` files |
| **Runner** | `vitest --typecheck` | Own tsc instance |
| **Error assertion** | `// @ts-expect-error` | `expectError(() => ...)` (block-level) |
| **Dep size** | 0 dependencies | Ships patched TypeScript (~2.6MB) |
| **Config** | Inherits project tsconfig | Separate `"tsd"` in package.json |
| **Best for** | Application code, full-stack projects | Published libraries (public type API) |

Recommendation for this monorepo: use **expect-type** everywhere. It's already in Vitest, type assertions live next to runtime tests, and `vitest run --typecheck` runs them alongside unit tests in one pass. No separate tool needed for `@packages/shared` since types are consumed internally.

### What to type-test in @packages/shared

1. **Inferred types from Zod schemas:**
   ```ts
   import { userSchema } from './user.schema'
   import { expectTypeOf } from 'expect-type'

   it('infers User type from schema', () => {
     type User = z.infer<typeof userSchema>
     expectTypeOf<User>().toMatchObjectType<{ id: string; email: string }>()
   })
   ```

2. **Utility types:**
   ```ts
   it('ApiError has correct shape', () => {
     expectTypeOf<ApiError>().toMatchObjectType<{ statusCode: number; error: string; message: string }>()
   })
   ```

3. **Job payload types:**
   ```ts
   it('EmailJobPayload requires to and subject', () => {
     expectTypeOf<EmailJobPayload>().toMatchObjectType<{ to: string; subject: string }>()
   })
   ```

### Where type tests live

Two valid patterns:

1. **Co-located** — type test in the same file as the schema's runtime tests:
   ```
   packages/shared/src/schemas/
     user.schema.ts
     user.schema.test.ts   ← both runtime + type assertions
   ```

2. **Dedicated type test directory** (for types with no runtime tests):
   ```
   packages/shared/src/
     __typetests__/
       api-error.test.ts
       utility-types.test.ts
   ```

Co-location is preferred when there's already a `.test.ts` file. Dedicated directory for type-only tests.

---

## Shared typed fixtures

### Factory function pattern

Fixtures are factory functions that accept overrides and return a fully typed object validated against a Zod schema:

```ts
// packages/shared/src/test-fixtures/user.fixture.ts
import { userSchema, type User } from '../schemas/user.schema'

export const createUser = (overrides: Partial<User> = {}): User => {
  return userSchema.parse({
    id: '00000000-0000-0000-0000-000000000001',
    email: 'test@example.com',
    name: 'Test User',
    role: 'user',
    createdAt: new Date('2025-01-01'),
    ...overrides,
  })
}
```

Key properties:
- **`.parse()` validates** — fixture won't silently produce invalid data.
- **Defaults are valid** — zero-override call returns a valid object.
- **Overrides are typed** — `Partial<User>` means only valid keys can be overridden.
- **`.parse()` runs at call time** — not at module load. So overrides are applied before validation.

### Zod-generated valid test data

For schemas with complex constraints (email format, UUID, enums), use [`@anatine/zod-mock`](https://github.com/anatine/zod-mock) or `faker` + `.parse()`:

```ts
import { faker } from '@faker-js/faker'
import { createUserSchema, type CreateUser } from '@shared/schemas'

export const createValidCreateUserPayload = (overrides: Partial<CreateUser> = {}): CreateUser => {
  return createUserSchema.parse({
    email: faker.internet.email(),
    name: faker.person.fullName(),
    ...overrides,
  })
}
```

For most cases, hand-crafted valid defaults (first pattern) are simpler and more predictable than faker-generated values. Use faker when test data variety matters (property-based testing, seed data).

### Fixture location and import strategy

```
packages/shared/src/
  test-fixtures/
    user.fixture.ts
    job.fixture.ts
    index.ts              ← barrel re-export
```

Apps import from the shared fixtures barrel:

```ts
import { createUser, createValidCreateUserPayload } from '@shared/test-fixtures'
```

Per-app fixtures (specific to one app's domain) live in the app:

```
apps/backend/src/
  test/
    fixtures/
      order.fixture.ts
```

---

## Antipatterns and common mistakes

1. **Spinning up a real HTTP server in tests** — use `app.inject()`, never `app.listen()`.
2. **`vitest.workspace.ts` / `defineWorkspace()` in 4.x** — these are removed. Use `test.projects` in `vitest.config.ts`.
3. **`vi.mock('@shared/x')` without factory** — bare alias mocking is unreliable in workspace mode. Always provide a factory function.
4. **Type assertions (`as`) instead of `expectTypeOf`** — `as` casts silence type errors; `expectTypeOf` catches them at compile time.
5. **Shared mutable state across test files** — global `beforeEach` resets help, but module-level singletons survive across files. Use `vi.resetModules()` or avoid module-level state.
6. **Importing from app internals in @shared tests** — `@packages/shared` must not import from `apps/*`. Tests should verify shared types without depending on app-specific values.
7. **Testing BullMQ retry mechanics in unit tests** — the handler should throw; BullMQ's retry count/backoff is the queue's responsibility. Test handler throwing behavior, not retry counts.
8. **Not calling `app.ready()` before `inject()`** — Fastify plugins load asynchronously. `app.ready()` must resolve before any `inject()` call, or inject calls may hit unregistered routes.
9. **Leaking `vitest/globals` types into production tsconfig** — use a separate `tsconfig.test.json` that extends the app tsconfig.
10. **Forgetting `afterAll(app.close())`** — Fastify holds connections (even inject-only). Unclosed instances leak memory across test files.

---

## Audit-flagged gaps — resolution

### Gap 1: Vitest workspace config format

**Resolved.** Vitest 4.1.7 uses `test.projects` in root `vitest.config.ts`. `vitest.workspace.ts` files and `defineWorkspace()` are removed (deprecated in 3.2, removed in 4.0). Per-package configs use `defineProject()` for type safety. Only filenames `vitest.config.*` / `vite.config.*` are recognized as project configs.

Source: vitest.dev/guide/projects, Vitest 4.0.0 release notes, PR #8218.

### Gap 2: Turborepo + Vitest integration

**Resolved.** The hybrid approach is the current best practice: `turbo run test` for CI (per-package caching, only changed packages re-test) and `vitest` or `vitest run` from root for local dev (unified watch, merged coverage). The `turbo.json` test task uses `dependsOn: ["transit"]` (aligning with D3) and includes `vitest.config.ts` in inputs for cache invalidation.

Source: github.com/vercel/turborepo/tree/main/examples/with-vitest, Turborepo docs.

### Gap 3: Coverage merging across workspace projects

**Resolved.** When running `vitest run --coverage` from root with `test.projects`, coverage is merged automatically (Vitest 4.x uses AST-based remapping). Known issue: duplicate coverage entries when different transpilers produce inconsistent source maps. Fix: consistent transpilation across projects, or use istanbul provider. For the Turborepo per-package path (CI), manual merging with `nyc report` is required.

Source: Vitest PR #8064, Issue #9366, PR #9480, Discussion #9637.

### Gap 4: vi.mock() with workspace path aliases

**Resolved.** Bare `vi.mock('@shared/module')` without a factory function is unreliable in workspace mode — it relies on `__mocks__` auto-discovery which doesn't work with pnpm `workspace:*` symlinks. **Always use a factory function.** Configure aliases in both `resolve.alias` and `test.alias` for redundancy.

Source: Vitest Discussions #4927, Issue #7708, Discussion #7252.

### Gap 5: expect-type vs tsd

**Resolved.** For this monorepo (internal types, application code), `expect-type` is the recommended choice. It ships with Vitest (no extra dependency), type assertions live alongside runtime tests, and `vitest run --typecheck` runs them in one pass. tsd is better for published libraries with public type APIs, where block-level `expectError()` and independent compiler config matter.

Source: github.com/mmkal/expect-type (README, 1.3.0 release), community discussions.

---

## Reference snippets

### vitest.config.ts (root, annotated)

```ts
// vitest.config.ts — repo root
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    // -- Global (root-level) settings --
    reporters: process.env.CI ? ['json', 'html'] : ['default'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
      reportsDirectory: './coverage',
      include: ['apps/*/src/**/*.ts', 'packages/*/src/**/*.ts'],
      exclude: ['**/*.test.ts', '**/*.spec.ts', '**/test/**'],
    },

    // -- Projects --
    projects: [
      'apps/backend',
      'apps/frontend',
      'apps/workers',
      'packages/shared',
      'packages/config-eslint',
    ],
  },
})
```

### Per-app vitest.config.ts (backend)

```ts
// apps/backend/vitest.config.ts
import { defineProject } from 'vitest/config'
import path from 'node:path'

export default defineProject({
  resolve: {
    alias: {
      '@shared': path.resolve(__dirname, '../../packages/shared/src'),
      '@backend': path.resolve(__dirname, './src'),
    },
  },
  test: {
    name: 'backend',
    environment: 'node',
    globals: true,
    setupFiles: ['./src/test/setup.ts'],
    include: ['src/**/*.test.ts'],
  },
})
```

### tsconfig.test.json (extends app tsconfig)

```json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "types": ["vitest/globals"]
  },
  "include": ["src/**/*.test.ts", "src/test/**/*.ts"]
}
```

### Test app factory for Fastify

```ts
// apps/backend/src/test/app.ts
import Fastify, { FastifyInstance } from 'fastify'
import { routes } from '../routes'

export async function buildTestApp(overrides?: {
  decorateUser?: Record<string, unknown>
}): Promise<FastifyInstance> {
  const app = Fastify({ logger: false })

  if (overrides?.decorateUser) {
    app.decorateRequest('user', null)
    app.addHook('onRequest', async (req) => {
      ;(req as any).user = overrides.decorateUser
    })
  }

  await app.register(routes)
  return app
}
```

### Route test with inject() (typed)

```ts
// apps/backend/src/routes/users.test.ts
import { describe, it, expect, beforeAll, afterAll } from 'vitest'
import type { FastifyInstance } from 'fastify'
import { buildTestApp } from '../test/app'
import type { UserResponse, ApiError } from '@shared/types'
import { prismaMock } from '../test/singleton'

let app: FastifyInstance

beforeAll(async () => {
  app = await buildTestApp()
  await app.ready()
})

afterAll(async () => {
  await app.close()
})

it('GET /users/:id returns user', async () => {
  prismaMock.user.findUnique.mockResolvedValue({
    id: 'u1', email: 'a@b.com', name: 'Test', role: 'user', createdAt: new Date(),
  })

  const res = await app.inject({ method: 'GET', url: '/users/u1' })

  expect(res.statusCode).toBe(200)
  const body = res.json() as UserResponse
  expect(body).toMatchObject({ id: 'u1', email: 'a@b.com' })
})

it('GET /users/:id returns 404 for missing user', async () => {
  prismaMock.user.findUnique.mockResolvedValue(null)

  const res = await app.inject({ method: 'GET', url: '/users/nonexistent' })

  expect(res.statusCode).toBe(404)
  const body = res.json() as ApiError
  expect(body).toMatchObject({ statusCode: 404, error: 'Not Found' })
})
```

### Job handler unit test (mocked dependencies)

```ts
// apps/workers/src/jobs/process-order.test.ts
import { describe, it, expect, vi } from 'vitest'
import { processOrder } from './process-order'
import { mockDeep, DeepMockProxy } from 'vitest-mock-extended'
import type { PrismaClient } from '@prisma/client'
import type { OrderJobPayload } from '@shared/schemas'

let db: DeepMockProxy<PrismaClient>
let inventoryApi: { reserve: ReturnType<typeof vi.fn> }

beforeEach(() => {
  db = mockDeep<PrismaClient>()
  inventoryApi = { reserve: vi.fn() }
})

it('reserves inventory for valid order', async () => {
  db.order.findUniqueOrThrow.mockResolvedValue({
    id: '1', items: [{ sku: 'A', qty: 2 }],
  } as any)
  inventoryApi.reserve.mockResolvedValue(undefined)

  const payload: OrderJobPayload = { orderId: '1' }
  await processOrder(payload, { db, inventoryApi })

  expect(inventoryApi.reserve).toHaveBeenCalledWith([{ sku: 'A', qty: 2 }])
})

it('throws on missing order', async () => {
  db.order.findUniqueOrThrow.mockRejectedValue(new Error('Not found'))
  const payload: OrderJobPayload = { orderId: 'missing' }

  await expect(processOrder(payload, { db, inventoryApi })).rejects.toThrow('Not found')
})
```

### Type-level test with expectTypeOf

```ts
// packages/shared/src/schemas/user.schema.test.ts
import { describe, it } from 'vitest'
import { expectTypeOf } from 'expect-type'
import { z } from 'zod'
import { userSchema } from './user.schema'

describe('User type inference', () => {
  it('User matches schema shape', () => {
    type User = z.infer<typeof userSchema>
    expectTypeOf<User>().toMatchObjectType<{
      id: string
      email: string
      name: string
      role: string
      createdAt: Date
    }>()
  })

  it('User.id is a string (not number)', () => {
    type User = z.infer<typeof userSchema>
    expectTypeOf<User['id']>().toBeString()
    expectTypeOf<User['id']>().not.toBeNumber()
  })
})
```

### Typed fixture factory function

```ts
// packages/shared/src/test-fixtures/user.fixture.ts
import { userSchema, type User } from '../schemas/user.schema'

export const createUser = (overrides: Partial<User> = {}): User => {
  return userSchema.parse({
    id: '00000000-0000-0000-0000-000000000001',
    email: 'test@example.com',
    name: 'Test User',
    role: 'user',
    createdAt: new Date('2025-01-01'),
    ...overrides,
  })
}

// packages/shared/src/test-fixtures/index.ts
export { createUser } from './user.fixture'
export { createOrderJob } from './job.fixture'
```

---

## Sources used

- [Vitest 4.0.0 Release Notes](https://github.com/vitest-dev/vitest/releases/tag/v4.0.0) — official
- [Vitest Projects Guide](https://vitest.dev/guide/projects) — official
- [Vitest Configuration Reference](https://vitest.dev/config/) — official
- [Vitest Migration Guide (v4)](https://vitest.dev/guide/migration) — official
- [Turborepo with-vitest Example](https://github.com/vercel/turborepo/tree/main/examples/with-vitest) — official
- [expect-type README (1.3.0)](https://github.com/mmkal/expect-type) — official
- [Fastify Testing Guide](https://github.com/fastify/fastify/blob/main/docs/Guides/Testing.md) — official
- [Vitest PR #8218 — Remove deprecated workspace option](https://github.com/vitest-dev/vitest/pull/8218) — official
- [Vitest PR #8064 — AST-based coverage remapping](https://github.com/vitest-dev/vitest/pull/8064) — official
- [Vitest Issue #9366 — Duplicate coverage merging](https://github.com/vitest-dev/vitest/issues/9366) — official
- [Vitest Issue #7708 — pnpm workspace:* **mocks**](https://github.com/vitest-dev/vitest/issues/7708) — official
- [Vitest Discussions #4927 — vi.mock with aliases](https://github.com/vitest-dev/vitest/discussions/4927) — community
- [Prisma Testing Guide (singleton pattern)](https://www.prisma.io/docs/orm/prisma-client/testing/unit-testing) — official
- [BullMQ Testing Patterns (OneUptime)](https://oneuptime.com/blog/bullmq-testing) — expert
