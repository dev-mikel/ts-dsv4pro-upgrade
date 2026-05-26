# Skill: TypeScript — D8: Backend App (Fastify)

## Summary

Backend HTTP API layer built with **Fastify 5.8.5** (stable, April 2026) and **@fastify/type-provider-zod 1.0.0** (official Fastify org package, April 2026). The backend follows the plugin-per-module pattern: each domain module is a self-contained Fastify plugin registered at startup. All request/response validation uses Zod schemas from `@shared` (D4, D5) via the type provider — no manual validation, no inline DTOs. Environment variables are validated with Zod at startup (fail-fast). Graceful shutdown with SIGTERM/SIGINT handlers is mandatory for containerized deployment (S2).

**Audit risk:** Moderate-High (all 6 flagged gaps resolved). **Dependencies:** D1 (TS 6.0.3), D2 (compiler), D3 (monorepo), D4 (Zod 4.4.3), D5 (shared types/errors), D6 (testing with inject()), D7 (linting). Knowledge cutoff: 2026-05-25.

**Zod compatibility status: CONFIRMED.** `@fastify/type-provider-zod@1.0.0` requires Zod 4.2+. D4 documents Zod 4.4.3. These are compatible. The legacy `fastify-type-provider-zod@6.1.0` (requires Zod >=4.1.5) is also compatible but deprecated in favor of the official package.

## Version landscape

| Package | Version | Notes |
|---|---|---|
| Fastify | **5.8.5** (Apr 2026) | Current stable major. v4 EOL was June 2025. |
| @fastify/type-provider-zod | **1.0.0** (Apr 2026) | Official Fastify org package. Supersedes `fastify-type-provider-zod`. |
| fastify-type-provider-zod | 6.1.0 (Oct 2025) | Legacy community package. Still works but migration recommended. |
| fastify-plugin | 5.x | `encapsulate: true` option added in v5. |
| @fastify/env | 6.0.0 (Mar 2026) | Documented as alternative; manual Zod validation is preferred. |
| Zod | 4.4.3 (D4) | @fastify/type-provider-zod requires Zod 4.2+. |
| TypeScript | 6.0.3 (D1) | Baseline 5.5+ as declared in scope. |
| Node.js | 20+ | Fastify 5 requires Node.js 20+. |

### Fastify 5.x breaking changes from 4.x

Key changes that affect the plugin-per-module pattern:

| Change | Impact |
|---|---|
| `listen(port, host, cb)` removed | Must use `listen({ port, host })` object-only signature |
| `logger` option removed | Use `loggerInstance` for custom logger; `logger: true/false` for pino config still works |
| `jsonShortHand` removed | All schemas must be full JSON Schema. Zod type provider handles this automatically. |
| `error` in `setErrorHandler` typed as `unknown` | Must narrow with `instanceof` or type guards. Generic `setErrorHandler<TError>()` available. |
| `request.params` has null prototype | Cannot call `toString()`, `hasOwnProperty()` on params directly |
| Type providers split | `ValidatorSchema` and `SerializerSchema` are separate types. Handled internally by `@fastify/type-provider-zod`. |
| `useSemicolonDelimiter` defaults to `false` | Non-standard per RFC 3986. Opt back in if needed. |
| `reply.redirect()` new signature | Check docs if using redirects. |
| `reply.sent` is read-only | Use `reply.hijack()` to manually handle the response. |
| Removed APIs | `request.connection` → `request.socket`, `reply.getResponseTime()` → `reply.elapsedTime`, `getDefaultRoute()`/`setDefaultRoute()` → `setNotFoundHandler`/`setErrorHandler` |
| `request.hostname` no longer includes port | Use `request.host` for host:port, `request.hostname` for host only, `request.port` for port |

Source: [Fastify v5 Migration Guide](https://fastify.dev/docs/latest/Guides/Migration-Guide-V5/)

## Fastify instance setup

```ts
import Fastify from "fastify";
import type { ZodTypeProvider } from "@fastify/type-provider-zod";
import {
  serializerCompiler,
  validatorCompiler,
} from "@fastify/type-provider-zod";

const app = Fastify({
  // Logger: pino is built in. Use `true` for defaults, object for config.
  logger: {
    level: process.env.LOG_LEVEL ?? "info",
    // In production, output JSON for log aggregation (Datadog, Grafana, etc.)
    ...(process.env.NODE_ENV === "production" && {
      transport: undefined, // default pino output is JSON — keep it
    }),
  },

  // trustProxy: trust X-Forwarded-* headers when behind a reverse proxy.
  // Required for correct IP logging and rate limiting behind nginx/ALB.
  trustProxy: true,

  // Keep-alive timeout: Fastify defaults to 72s. Lower for containers
  // so idle connections don't block graceful shutdown.
  keepAliveTimeout: 5000,

  // Force-close idle keep-alive connections on app.close().
  // Prevents 72s shutdown hangs in containerized environments.
  forceCloseConnections: true,

  // Connection timeout for new connections to complete the handshake.
  connectionTimeout: 5000,

  // AJV: Zod type provider replaces AJV for validation, but AJV is still
  // used internally for route matching. Default AJV config is fine.
  // Only configure if you need custom formats or keywords.
  ajv: {
    customOptions: {
      // Remove additional properties from body by default.
      // Zod's .strip() equivalent. Aligns with Zod strictObject behavior.
      removeAdditional: true,
    },
  },
});

// Register Zod compilers BEFORE any routes or plugins that define routes.
// These replace Fastify's default AJV-based validator and serializer.
app.setValidatorCompiler(validatorCompiler);
app.setSerializerCompiler(serializerCompiler);
```

### TypeScript: FastifyInstance type

```ts
import type {
  FastifyInstance,
  FastifyBaseLogger,
  RawServerDefault,
} from "fastify";
import type { ZodTypeProvider } from "@fastify/type-provider-zod";

// Scoped: use withTypeProvider() on the instance directly (preferred)
app.withTypeProvider<ZodTypeProvider>().get("/", { schema }, handler);

// Reusable type alias: when passing the instance around
type AppInstance = FastifyInstance<
  RawServerDefault,
  RawRequestDefaultExpression<RawServerDefault>,
  RawReplyDefaultExpression<RawServerDefault>,
  FastifyBaseLogger,
  ZodTypeProvider
>;
```

`withTypeProvider<ZodTypeProvider>()` returns a scoped instance whose route methods (`get`, `post`, `put`, `delete`, `patch`, `route`) accept Zod schemas and infer types from them. The type provider does not propagate globally — each encapsulated context must call `withTypeProvider()` if it defines routes. Plugins that only register other plugins (no routes of their own) don't need it.

## Plugin-per-module pattern

### fp() — what it does

`fp()` from `fastify-plugin` breaks Fastify's encapsulation. Without `fp()`, a plugin registered via `app.register()` is encapsulated: its decorators, hooks, and routes are invisible to the parent scope. With `fp()`, the plugin's registrations "leak" upward to the parent — which is exactly what you want for shared infrastructure (database connections, auth decorators, typed config).

**When `fp()` is required:**
- Decorating the instance with shared state (`app.decorate("db", ...)`)
- Adding global hooks that should affect parent routes
- Registering plugins that other plugins depend on (e.g., a DB plugin that auth depends on)

**When `fp()` is not needed (plain `register` is fine):**
- Domain route plugins (users, orders, products) — encapsulation keeps their routes isolated
- Plugins that only add routes under a prefix — encapsulation is the desired behavior

**`encapsulate: true` option (Fastify 5):** Pass `{ encapsulate: true }` as the second argument to `fp()` to keep the plugin encapsulated while still getting `fp()` benefits (name, dependencies, decorators validation). Use this for domain route plugins when you want `fp()`'s dependency management but encapsulation too.

### Plugin structure

One file per domain module. Exports a `FastifyPluginAsync`:

```ts
// apps/backend/src/plugins/users.ts
import type { FastifyPluginAsync } from "fastify";
import type { ZodTypeProvider } from "@fastify/type-provider-zod";
import { CreateUserSchema, UserSchema } from "@shared/schemas";
import type { CreateUserInput, User } from "@shared/schemas";
import type { CreateUserOutput } from "@shared/schemas";
import { UserIdSchema } from "@shared/schemas";

interface UsersPluginOptions {
  userService: UserService;
}

const usersPlugin: FastifyPluginAsync<UsersPluginOptions> = async (
  fastify,
  opts,
) => {
  const { userService } = opts;

  fastify.withTypeProvider<ZodTypeProvider>().post("/", {
    schema: {
      body: CreateUserSchema,
      response: {
        201: UserSchema,
      },
    },
    handler: async (request, reply) => {
      // request.body is typed as CreateUserInput
      const user = await userService.create(request.body);
      // reply.send() is type-checked against UserSchema
      return reply.code(201).send(user);
    },
  });

  fastify.withTypeProvider<ZodTypeProvider>().get("/:id", {
    schema: {
      params: UserIdSchema,
      response: {
        200: UserSchema,
      },
    },
    handler: async (request, reply) => {
      // request.params.id is typed as string
      const user = await userService.findById(request.params.id);
      return reply.send(user);
    },
  });
};

export default usersPlugin;
```

**Key pattern:** `fastify.withTypeProvider<ZodTypeProvider>()` must be called on the encapsulated `fastify` instance (the first parameter of the plugin), NOT on the outer `app`. This is because plugins are encapsulated — the outer `app`'s type provider does not propagate into the plugin's scope.

### Typed plugin options

Plugin options are typed as the generic parameter of `FastifyPluginAsync<T>` or `FastifyPluginCallback<T>`:

```ts
interface DbPluginOptions {
  url: string;
}

// Async plugin (preferred — no done() callback)
const dbPlugin: FastifyPluginAsync<DbPluginOptions> = async (fastify, opts) => {
  const db = await createConnection(opts.url);
  fastify.decorate("db", db);
};

// Callback plugin (legacy, still works)
const dbPluginCb: FastifyPluginCallback<DbPluginOptions> = (fastify, opts, done) => {
  createConnection(opts.url, (err, db) => {
    if (err) return done(err);
    fastify.decorate("db", db);
    done();
  });
};
```

### Plugin registration

```ts
// apps/backend/src/app.ts
import fp from "fastify-plugin";
import dbPlugin from "./plugins/db.js";
import usersPlugin from "./plugins/users.js";

// Infrastructure plugins use fp() — they decorate the instance
await app.register(fp(dbPlugin), { url: process.env.DATABASE_URL });

// Domain plugins use plain register — encapsulation isolates their routes
// prefix: "/api/users" means all routes in usersPlugin are relative to that prefix
await app.register(usersPlugin, {
  prefix: "/api/users",
  userService: app.userService, // typed options passed through
});
```

### Plugin dependency order

Plugins load at `.listen()`, `.ready()`, or `.inject()` time — not at `register()` time. `register()` only queues the plugin. Fastify uses `avvio` internally for graph-based async boot ordering.

**Controlling order with `fp()` dependencies:**

```ts
export default fp(authPlugin, {
  name: "auth",
  fastify: "5.x",
  dependencies: ["db"], // db plugin must load before auth
  decorators: {
    fastify: ["db"], // validates that app.db exists before auth loads
  },
});
```

**Controlling order with `app.after()`:**

```ts
// Register infrastructure first
await app.register(fp(dbPlugin), { url: process.env.DATABASE_URL });
await app.register(fp(authPlugin));

// after() fires once all plugins registered so far have loaded
app.after(() => {
  // Now db and auth are guaranteed loaded
  app.register(usersPlugin, { prefix: "/api/users" });
});
```

**General rule:** Infrastructure plugins (db, auth, config) use `fp()` with explicit `dependencies`. Domain route plugins use `after()` or just rely on `fp()` dependencies of the plugins they consume. Plugin loading is deferred — you cannot access a decorator created by another plugin until `.ready()` or inside an `after()` callback.

## Typed decorators

### Traditional approach: module augmentation

```ts
// apps/backend/src/types/fastify.d.ts — or inline in the decorator plugin
declare module "fastify" {
  interface FastifyInstance {
    db: Database;
    config: AppConfig;
  }

  interface FastifyRequest {
    session: Session;
    requestId: string;
  }

  interface FastifyReply {
    sendSuccess<T>(data: T): void;
  }
}
```

This is global: once augmented, types apply to all Fastify instances in the project. Works fine for a single-server app. The `@shared` types (D5) plug directly in here — `Database`, `Session`, `AppConfig` are all `@shared` types or re-exports.

### Fastify 5 alternative: getDecorator<T>()

```ts
// No declare module needed — scoped to the encapsulated context
fastify.decorateRequest("session", null);

fastify.addHook("onRequest", async (request) => {
  const db = fastify.getDecorator<Database>("db");
  const session = await loadSession(db, request.headers.authorization);
  request.setDecorator("session", session);
});

fastify.get("/me", (request, reply) => {
  const session = request.getDecorator<Session>("session");
  reply.send(session.user);
});
```

**Recommendation:** Use module augmentation for shared infrastructure decorators that every plugin accesses (`app.db`, `request.session`). Use `getDecorator<T>()` for scoped decorators that only specific plugins need, or in multi-server codebases where global augmentation would cause conflicts. For this monorepo (single backend process), module augmentation is simpler and more ergonomic for the common decorators.

### decorate / decorateRequest / decorateReply

```ts
// Instance decorator: available on fastify.* in all child contexts (if fp-wrapped)
fastify.decorate("db", dbClient);

// Request decorator: new instance per request. Must initialize to null.
fastify.decorateRequest("session", null);

// Reply decorator: new instance per reply.
fastify.decorateReply("sendSuccess", function <T>(this: FastifyReply, data: T) {
  return this.send({ success: true, data });
});
```

**Critical rule:** Request and reply decorators must be initialized to `null`/`undefined` at registration time and set to their real value in an `onRequest` hook. This avoids shared mutable state across requests.

**Getter/setter syntax (Fastify 5):**

```ts
fastify.decorate("computedProp", {
  getter() {
    return this._internalValue;
  },
  setter(val: string) {
    this._internalValue = val;
  },
});
```

## Zod type provider integration

### Setup

```ts
import { serializerCompiler, validatorCompiler } from "@fastify/type-provider-zod";
import type { ZodTypeProvider } from "@fastify/type-provider-zod";

// Call ONCE at app level, before any routes. These replace AJV globally.
app.setValidatorCompiler(validatorCompiler);
app.setSerializerCompiler(serializerCompiler);
```

### Schema on routes

Every route schema field accepts a Zod schema:

| Schema key | Validates | Type inferred as |
|---|---|---|
| `body` | Request body | `request.body` |
| `querystring` | Query parameters | `request.query` |
| `params` | Route parameters | `request.params` |
| `headers` | Request headers | `request.headers` |
| `response` | Response body (status-code-keyed) | `reply.send()` parameter |

### Reusing @shared Zod schemas

D4 and D5 establish that Zod schemas and their inferred types live in `@shared/schemas`:

```ts
// @shared/src/schemas/user.ts (from D4/D5)
export const CreateUserSchema = z.object({
  name: z.string().min(1).max(200),
  email: z.email(),
});

export type CreateUserInput = z.input<typeof CreateUserSchema>;
export type CreateUserOutput = z.output<typeof CreateUserSchema>;

// Apps/backend route — reuse the same schema
import { CreateUserSchema } from "@shared/schemas";
import type { CreateUserInput, CreateUserOutput } from "@shared/schemas";

fastify.withTypeProvider<ZodTypeProvider>().post("/", {
  schema: {
    body: CreateUserSchema,
    response: { 201: UserSchema },
  },
  handler: async (request, reply) => {
    // request.body: CreateUserInput — fully typed from @shared
    const user = await userService.create(request.body);
    // reply.send(user): type-checked against UserSchema (User type)
    return reply.code(201).send(user);
  },
});
```

The Zod type provider generates JSON Schema from the Zod schema at runtime (using Zod 4's native `.toJSONSchema()` internally) and passes it to Fastify's validation pipeline. TypeScript inference happens at the type level — the type provider maps Zod schema types through Fastify's generic route handler types.

### FastifyPluginAsyncZod convenience type

For domain plugins where every route uses Zod schemas:

```ts
import type { FastifyPluginAsyncZod } from "@fastify/type-provider-zod";

const plugin: FastifyPluginAsyncZod = async (fastify, _opts) => {
  // fastify is already typed with ZodTypeProvider — no withTypeProvider() needed
  fastify.get("/", {
    schema: {
      querystring: z.object({ page: z.coerce.number().optional() }),
      response: { 200: UserListSchema },
    },
    handler: async (request, reply) => {
      // request.query.page: number | undefined (fully typed)
      return reply.send(await listUsers(request.query.page));
    },
  });
};
```

### Error handling when Zod validation fails

When validation fails, `@fastify/type-provider-zod` does NOT throw a raw `ZodError`. It throws a `FastifyError` with a `.validation` property containing the Zod issues array and a `.validationContext` property indicating which part failed (`"body"`, `"querystring"`, `"params"`, `"headers"`, `"response"`).

**Type guards provided by the package:**

```ts
import {
  hasZodFastifySchemaValidationErrors,
  isResponseSerializationError,
} from "@fastify/type-provider-zod";
```

| Guard | Purpose |
|---|---|
| `hasZodFastifySchemaValidationErrors(err)` | Request validation failed — body/query/params/headers |
| `isResponseSerializationError(err)` | Response failed to serialize (internal error — the handler returned data that doesn't match the response schema) |

## Centralized error handling

### setErrorHandler() typed

In Fastify 5, the error parameter is typed as `unknown`. Narrow with type guards:

```ts
import { FastifyError } from "fastify";
import {
  hasZodFastifySchemaValidationErrors,
  isResponseSerializationError,
} from "@fastify/type-provider-zod";
import type { ApiError } from "@shared";

app.setErrorHandler((err, request, reply) => {
  // 1) Zod validation errors — request didn't match the schema
  if (hasZodFastifySchemaValidationErrors(err)) {
    const apiError: ApiError = {
      success: false,
      error: {
        code: "VALIDATION_ERROR",
        message: `Request ${err.validationContext} doesn't match the schema`,
        details: err.validation, // ZodIssue[]
      },
    };
    return reply.code(400).send(apiError);
  }

  // 2) Response serialization error — handler returned wrong shape
  if (isResponseSerializationError(err)) {
    return reply.code(500).send({
      success: false,
      error: {
        code: "INTERNAL_ERROR",
        message: "Response doesn't match the schema",
      },
    });
  }

  // 3) Fastify internal errors (route not found handled by setNotFoundHandler)
  if (err instanceof FastifyError) {
    return reply.code(err.statusCode ?? 500).send({
      success: false,
      error: {
        code: err.code,
        message: err.message,
      },
    });
  }

  // 4) Unknown errors — log the full error, return a sanitised response
  request.log.error(err);
  return reply.code(500).send({
    success: false,
    error: {
      code: "INTERNAL_ERROR",
      message: "An unexpected error occurred",
    },
  });
});
```

### Mapping domain errors to HTTP status codes

Use custom `Error` subclasses with an HTTP status code property:

```ts
// @shared/src/errors/index.ts — co-locate error definitions with the contract
export class NotFoundError extends Error {
  readonly statusCode = 404;
  readonly code = "NOT_FOUND";
  constructor(message: string) {
    super(message);
    this.name = "NotFoundError";
  }
}

export class UnauthorizedError extends Error {
  readonly statusCode = 401;
  readonly code = "UNAUTHORIZED";
  constructor(message = "Authentication required") {
    super(message);
    this.name = "UnauthorizedError";
  }
}

export class ForbiddenError extends Error {
  readonly statusCode = 403;
  readonly code = "FORBIDDEN";
  constructor(message = "Insufficient permissions") {
    super(message);
    this.name = "ForbiddenError";
  }
}
```

Then in the error handler (above the generic `Error` case):

```ts
// Catch domain errors before the generic instanceof Error check
if (err instanceof NotFoundError) {
  return reply.code(404).send({
    success: false,
    error: { code: err.code, message: err.message },
  });
}
if (err instanceof UnauthorizedError) {
  return reply.code(401).send({
    success: false,
    error: { code: err.code, message: err.message },
  });
}
```

### Error response shape

Aligned with the `ApiError` type from D5 (`@shared/src/types/utility.ts`):

```ts
// @shared
export interface ApiError {
  success: false;
  error: {
    code: string;
    message: string;
    details?: unknown;
  };
}
export type ApiResult<T> = ApiResponse<T> | ApiError;
```

This shape is the contract between backend (produces it) and frontend (consumes it). Every error response, regardless of status code, conforms to this shape so the frontend error handling is uniform.

## Environment variables

### Recommended: manual Zod validation at startup

```ts
// apps/backend/src/env.ts
import { z } from "zod/v4";

const envSchema = z.object({
  NODE_ENV: z.enum(["development", "production", "test"]),
  PORT: z.coerce.number().int().positive().default(3000),
  HOST: z.string().default("0.0.0.0"),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url().optional(),
  LOG_LEVEL: z.enum(["trace", "debug", "info", "warn", "error", "fatal"]).default("info"),
  JWT_SECRET: z.string().min(32),
});

export type EnvConfig = z.output<typeof envSchema>;

export function loadEnv(): EnvConfig {
  const result = envSchema.safeParse(process.env);
  if (!result.success) {
    console.error("Invalid environment variables:", result.error.toString());
    process.exit(1);
  }
  return result.data;
}
```

**Why manual Zod over `@fastify/env`:**
- Simpler: no plugin registration, no lifecycle coupling. Just a function that returns a typed object.
- Type-safe: `EnvConfig` is fully inferred from the Zod schema. `@fastify/env` attached config to `fastify.config` without type inference from Zod (it uses JSON Schema internally).
- Fail-fast: if `process.env` is invalid, the process exits before the server even starts.
- Less dependencies: no `@fastify/env` package needed.
- `z.coerce` handles string→number/boolean conversion that `@fastify/env` would need explicit `type` declarations for.

### Passing config to plugins

Pass the typed config object via plugin options or decorate it on the instance:

```ts
// Option A: decorate the instance (access via fastify.config in any plugin)
import fp from "fastify-plugin";

declare module "fastify" {
  interface FastifyInstance {
    config: EnvConfig;
  }
}

const configPlugin: FastifyPluginAsync = async (fastify) => {
  const config = loadEnv();
  fastify.decorate("config", config);
};

export default fp(configPlugin, { name: "config" });

// Option B: pass explicitly via options (more explicit, less magic)
await app.register(usersPlugin, { config: loadEnv() });
```

Option A (decorator) is preferred for shared infrastructure config that many plugins need. Option B works for plugin-specific config that only one plugin needs.

### @fastify/env — when to consider it

Use `@fastify/env` v6.0.0+ only if you need its specific features:
- Built-in `.env` file loading with `dotenv: true`
- Fastify lifecycle integration (env loaded as a plugin, guaranteed before `.ready()`)
- Combined with Zod via `z.toJSONSchema()` for the schema definition:

```ts
// Only if you need @fastify/env features
await app.register(fastifyEnv, {
  schema: envSchema.toJSONSchema({ target: "draft-7" }),
  dotenv: true,
  confKey: "config",
});
```

For the monorepo context, manual Zod validation is recommended. It's simpler, more type-safe, and consistent with how Zod is used everywhere else (D4).

## Health check

### Endpoint structure

| Endpoint | Probe type | Purpose |
|---|---|---|
| `/health/live` | Liveness | Is the process alive? Always returns 200 if process is up. |
| `/health/ready` | Readiness | Can the service accept traffic? Checks DB, cache, etc. |

**Liveness must never fail due to a database outage** — that would trigger a Kubernetes restart loop.

```ts
// Liveness — cheap, always 200 while process is alive
app.get("/health/live", async () => {
  return { status: "ok" };
});

// Readiness — checks critical dependencies
app.get("/health/ready", async (request, reply) => {
  const db = app.getDecorator<Database>("db");
  const checks: Record<string, boolean> = {
    database: false,
  };

  try {
    await db.raw("SELECT 1");
    checks.database = true;
  } catch {
    // DB not ready
  }

  // Add Redis check if REDIS_URL is configured
  // Add BullMQ check if jobs are used

  const allHealthy = Object.values(checks).every(Boolean);
  if (!allHealthy) {
    return reply.code(503).send({
      status: "degraded",
      checks,
      timestamp: new Date().toISOString(),
    });
  }

  return {
    status: "ok",
    checks,
    timestamp: new Date().toISOString(),
  };
});
```

**Kubernetes alignment:**
- `livenessProbe` → `/health/live` with conservative `failureThreshold: 3`
- `readinessProbe` → `/health/ready` with `periodSeconds: 5`
- Fastify must bind to `0.0.0.0` for K8s probes to reach it (localhost is the pod-internal loopback)

## Graceful shutdown

### Complete handler

```ts
// apps/backend/src/shutdown.ts
import type { FastifyInstance } from "fastify";

const SHUTDOWN_TIMEOUT_MS = 25_000; // Must be < Docker's stop_grace_period (30s default)

export function registerShutdownHandlers(app: FastifyInstance) {
  const shutdown = async (signal: string) => {
    app.log.info(`Received ${signal}. Starting graceful shutdown...`);

    // Fallback: force exit if close takes too long
    const forceExit = setTimeout(() => {
      app.log.error("Forced shutdown after timeout — some connections may have been dropped");
      process.exit(1);
    }, SHUTDOWN_TIMEOUT_MS);
    forceExit.unref(); // Don't keep the event loop alive just for this timer

    try {
      await app.close();
      clearTimeout(forceExit);
      app.log.info("Server closed successfully");
      process.exit(0);
    } catch (err) {
      app.log.error("Error during graceful shutdown");
      app.log.error(err);
      process.exit(1);
    }
  };

  process.on("SIGTERM", () => shutdown("SIGTERM"));
  process.on("SIGINT", () => shutdown("SIGINT"));
}
```

### What app.close() does

1. Fires `preClose` hooks (cleanup, close DB connections, drain queues)
2. Stops accepting new connections (returns 503 so reverse proxies rebalance)
3. Waits for in-flight requests to complete (up to `keepAliveTimeout`)
4. Fires `onClose` hooks
5. Closes the HTTP server

### Critical settings for containerized environments

| Fastify option | Value | Why |
|---|---|---|
| `keepAliveTimeout` | `5000` | Fastify defaults to 72s. At 5s, idle keep-alive connections close quickly and don't block shutdown. |
| `forceCloseConnections` | `true` | Force-closes idle keep-alive sockets on `close()`. Without this, `app.close()` blocks until all idle connections timeout. |
| `connectionTimeout` | `5000` | Max time for new connection handshake. Protects against slow-loris attacks. |

### Docker alignment (S2)

```
Docker stop → sends SIGTERM to PID 1
  → SIGTERM handler calls app.close()
    → preClose hooks run
    → new connections rejected (503)
    → in-flight requests complete (or forceClosed after keepAliveTimeout)
    → onClose hooks run
    → process.exit(0)
  → Docker waits stop_grace_period (default 30s)
  → If process hasn't exited, Docker sends SIGKILL

CRITICAL: app SHUTDOWN_TIMEOUT (25s) < Docker stop_grace_period (30s)
```

**Dockerfile CMD must use exec form** so Node.js receives SIGTERM directly (not a shell):

```dockerfile
CMD ["node", "dist/apps/backend/main.js"]
```

## Application bootstrap

### Entry point

```ts
// apps/backend/src/main.ts
import { buildApp } from "./app.js";
import { registerShutdownHandlers } from "./shutdown.js";

async function main() {
  const app = await buildApp();

  registerShutdownHandlers(app);

  try {
    // Object-only signature (Fastify 5 requirement)
    const address = await app.listen({
      port: app.config.PORT,
      host: app.config.HOST, // "0.0.0.0" for containers
    });
    app.log.info(`Server listening at ${address}`);
  } catch (err) {
    app.log.error("Failed to start server");
    app.log.error(err);
    process.exit(1);
  }
}

main();
```

### App assembly

```ts
// apps/backend/src/app.ts
import Fastify from "fastify";
import fp from "fastify-plugin";
import {
  serializerCompiler,
  validatorCompiler,
} from "@fastify/type-provider-zod";
import { loadEnv } from "./env.js";
import configPlugin from "./plugins/config.js";
import dbPlugin from "./plugins/db.js";
import authPlugin from "./plugins/auth.js";
import usersPlugin from "./plugins/users.js";
import ordersPlugin from "./plugins/orders.js";
import { errorHandler } from "./error-handler.js";

export async function buildApp() {
  const config = loadEnv(); // Fail-fast: exits before app created if env is invalid

  const app = Fastify({
    logger: { level: config.LOG_LEVEL },
    trustProxy: true,
    keepAliveTimeout: 5000,
    forceCloseConnections: true,
    connectionTimeout: 5000,
  });

  // Compilers — must be set before any routes
  app.setValidatorCompiler(validatorCompiler);
  app.setSerializerCompiler(serializerCompiler);

  // Error handler — must be set before routes so validation errors are caught
  app.setErrorHandler(errorHandler);

  // Infrastructure plugins (fp-wrapped, order matters)
  await app.register(fp(configPlugin)); // fastify.config
  await app.register(fp(dbPlugin), { url: config.DATABASE_URL }); // fastify.db
  await app.register(fp(authPlugin)); // fastify.authenticate, request.session

  // Domain route plugins (encapsulated, order doesn't matter between them)
  await app.register(usersPlugin, { prefix: "/api/users" });
  await app.register(ordersPlugin, { prefix: "/api/orders" });

  // Health check — bare routes on the app, no prefix
  app.get("/health/live", async () => ({ status: "ok" }));
  app.get("/health/ready", async (request, reply) => {
    try {
      await app.db.raw("SELECT 1");
      return { status: "ok", checks: { database: true } };
    } catch {
      return reply.code(503).send({
        status: "degraded",
        checks: { database: false },
      });
    }
  });

  return app;
}
```

**Key ordering rules:**
1. `loadEnv()` first — fails before Fastify is even created if env is invalid
2. `setValidatorCompiler` / `setSerializerCompiler` before any routes
3. `setErrorHandler` before any routes (otherwise default handler catches validation errors)
4. Infrastructure plugins (`fp`-wrapped) before domain plugins
5. Domain route plugins last
6. `app.listen()` last — triggers plugin loading and starts the server

### Startup error handling

Errors during `app.listen()` are caught in `main()`. Errors during plugin loading (e.g., DB connection failure) propagate through `avvio` and reject the `app.listen()` promise. Fastify logs plugin errors automatically via pino.

## Antipatterns and common mistakes

1. **Inline DTOs instead of @shared types.**
   Don't define `interface CreateUserBody { name: string }` in the route file. Import `CreateUserInput` from `@shared/schemas`. The type is already inferred from the schema. Duplicating it breaks the single source of truth.

2. **Manual validation instead of Zod type provider.**
   Don't call `CreateUserSchema.parse(request.body)` inside a handler. The type provider handles this automatically via the route `schema` option. Manual parsing bypasses the type provider's TypeScript inference and the centralized error handler.

3. **Missing fp() on decorator plugins.**
   If you decorate the instance without `fp()`, the decorator is only visible inside the plugin's encapsulated scope — not in the parent or sibling scopes. Wrap infrastructure plugins with `fp()` so their decorators propagate upward.

4. **No graceful shutdown in a containerized app.**
   Without SIGTERM handling, Docker sends SIGKILL after `stop_grace_period`, dropping in-flight requests. Always register shutdown handlers.

5. **Binding to `localhost` instead of `0.0.0.0`.**
   In a container, binding to `127.0.0.1` means the port isn't reachable from outside the container. Always use `0.0.0.0` for containerized deployments. Same applies to Kubernetes probes — they hit the pod IP, not localhost.

6. **Using `app.listen(3000)` instead of `app.listen({ port: 3000 })`.**
   Fastify 5 removed the variadic signature. The old syntax is a TypeScript error. Always use the object form.

7. **Forgetting to unsubscribe `forceExit` timer.**
   If `app.close()` succeeds, the `setTimeout` must be cleared with `clearTimeout()`. Use `.unref()` on the timer so it doesn't prevent the process from exiting if `app.close()` succeeds but the event loop hasn't drained yet.

8. **Accessing decorators before plugins load.**
   `app.register()` is synchronous (it queues, not loads). Decorators from a plugin are not available until the plugin loads at `.ready()` or `.listen()`. Use `app.after()` or access them inside route handlers/hooks.

9. **Response schema mismatch with handler return type.**
   If `response: { 200: UserSchema }` but the handler returns `{ ...user, extraField }`, the response serialization will fail. This is caught by `isResponseSerializationError` in the error handler.

10. **Not using `withTypeProvider()` inside encapsulated plugins.**
    Type providers don't propagate through encapsulation. Each plugin that defines routes needs its own `withTypeProvider<ZodTypeProvider>()` call on its local `fastify` instance.

## Audit-flagged gaps — resolution

1. **Fastify 5.x breaking changes.** RESOLVED. Confirmed from the official [v5 Migration Guide](https://fastify.dev/docs/latest/Guides/Migration-Guide-V5/). Key impacts: `listen()` object-only signature, `logger`→`loggerInstance`, `jsonShortHand` removed, `error` typed as `unknown` in `setErrorHandler`, `request.params` null prototype, removed APIs. All examples in this document use v5 syntax.

2. **fastify-type-provider-zod integration.** RESOLVED. The package moved to the official Fastify org as `@fastify/type-provider-zod@1.0.0` (April 2026). Integration is `setValidatorCompiler`/`setSerializerCompiler` + `withTypeProvider<ZodTypeProvider>()`. For error handling, use `hasZodFastifySchemaValidationErrors()` and `isResponseSerializationError()` type guards. Legacy `fastify-type-provider-zod@6.1.0` has the same API but is deprecated.

3. **Zod 4.x compatibility.** RESOLVED. `@fastify/type-provider-zod@1.0.0` requires Zod 4.2+. D4 documents Zod 4.4.3. Compatible. The type provider uses Zod 4's native `.toJSONSchema()` internally (eliminating the `zod-to-json-schema` dependency that Zod 3 required).

4. **Typed decorators in Fastify 5.x.** RESOLVED. The traditional `declare module 'fastify'` interface augmentation pattern still works unchanged in v5. Fastify 5 added `getDecorator<T>()` / `setDecorator()` as an alternative for scoped typing without global augmentation. Recommendation: use module augmentation for shared infrastructure decorators; use `getDecorator<T>()` for scoped decorators or multi-server setups.

5. **@fastify/env vs manual Zod env validation.** RESOLVED. Manual Zod validation (`z.object().safeParse(process.env)`) is the recommended approach. It's simpler, type-safe, and provides fail-fast behavior without a plugin dependency. `@fastify/env@6.0.0` (March 2026) supports Zod→JSON Schema conversion via `z.toJSONSchema()` for projects that need its `.env` file loading or lifecycle integration. Source: Fastify issue #6279 / PR #6281 (August 2025).

6. **Graceful shutdown timeout pattern.** RESOLVED. Confirmed: Fastify does NOT provide a built-in force-exit timeout. The pattern is manual: `setTimeout` + `process.exit(1)` as fallback, with `.unref()` on the timer. Fastify 5's `keepAliveTimeout` and `forceCloseConnections` options address the underlying connection-holding problem. Source: [Fastify Server docs](https://fastify.dev/docs/latest/Reference/Server/) and community patterns.

## Reference snippets

### app.ts — Fastify instance setup + type provider

```ts
import Fastify from "fastify";
import type { ZodTypeProvider } from "@fastify/type-provider-zod";
import {
  serializerCompiler,
  validatorCompiler,
} from "@fastify/type-provider-zod";
import fp from "fastify-plugin";
import { loadEnv } from "./env.js";
import configPlugin from "./plugins/config.js";
import dbPlugin from "./plugins/db.js";
import authPlugin from "./plugins/auth.js";
import usersPlugin from "./plugins/users.js";
import { errorHandler } from "./error-handler.js";

export async function buildApp() {
  const config = loadEnv();

  const app = Fastify({
    logger: { level: config.LOG_LEVEL },
    trustProxy: true,
    keepAliveTimeout: 5000,
    forceCloseConnections: true,
    connectionTimeout: 5000,
  });

  app.setValidatorCompiler(validatorCompiler);
  app.setSerializerCompiler(serializerCompiler);
  app.setErrorHandler(errorHandler);

  await app.register(fp(configPlugin));
  await app.register(fp(dbPlugin), { url: config.DATABASE_URL });
  await app.register(fp(authPlugin));

  await app.register(usersPlugin, { prefix: "/api/users" });

  app.get("/health/live", async () => ({ status: "ok" }));

  return app;
}
```

### Domain plugin — users plugin with typed routes

```ts
import type { FastifyPluginAsync } from "fastify";
import type { ZodTypeProvider } from "@fastify/type-provider-zod";
import {
  CreateUserSchema,
  UserSchema,
  UserIdSchema,
} from "@shared/schemas";
import type { CreateUserInput, User } from "@shared/schemas";

interface UsersPluginOptions {
  userService: UserService;
}

const usersPlugin: FastifyPluginAsync<UsersPluginOptions> = async (
  fastify,
  opts,
) => {
  const { userService } = opts;

  fastify.withTypeProvider<ZodTypeProvider>().post("/", {
    schema: {
      body: CreateUserSchema,
      response: { 201: UserSchema },
    },
    handler: async (request, reply) => {
      const user = await userService.create(request.body);
      return reply.code(201).send(user);
    },
  });

  fastify.withTypeProvider<ZodTypeProvider>().get("/:id", {
    schema: {
      params: UserIdSchema,
      response: { 200: UserSchema },
    },
    handler: async (request, reply) => {
      const user = await userService.findById(request.params.id);
      if (!user) throw new NotFoundError("User not found");
      return reply.send(user);
    },
  });
};

export default usersPlugin;
```

### Typed decorator — auth user on request

```ts
// apps/backend/src/types/fastify.d.ts
import type { Session } from "@shared/types";

declare module "fastify" {
  interface FastifyInstance {
    authenticate: (request: FastifyRequest, reply: FastifyReply) => Promise<void>;
    db: Database;
    config: AppConfig;
  }

  interface FastifyRequest {
    session: Session;
  }
}
```

```ts
// apps/backend/src/plugins/auth.ts
import fp from "fastify-plugin";
import type { FastifyPluginAsync } from "fastify";
import { UnauthorizedError } from "@shared/errors";

const authPlugin: FastifyPluginAsync = async (fastify) => {
  fastify.decorateRequest("session", null);

  fastify.decorate("authenticate", async (request: FastifyRequest, reply: FastifyReply) => {
    const token = request.headers.authorization?.replace("Bearer ", "");
    if (!token) throw new UnauthorizedError();
    const session = await verifyToken(token);
    request.setDecorator("session", session);
  });
};

export default fp(authPlugin, {
  name: "auth",
  dependencies: ["db"],
  decorators: { fastify: ["db", "config"] },
});
```

### Centralized error handler

```ts
// apps/backend/src/error-handler.ts
import type { FastifyErrorHandler } from "fastify";
import { FastifyError } from "fastify";
import {
  hasZodFastifySchemaValidationErrors,
  isResponseSerializationError,
} from "@fastify/type-provider-zod";
import { NotFoundError, UnauthorizedError, ForbiddenError } from "@shared/errors";
import type { ApiError } from "@shared";

export const errorHandler: FastifyErrorHandler = (err, request, reply) => {
  // Zod validation errors
  if (hasZodFastifySchemaValidationErrors(err)) {
    return reply.code(400).send({
      success: false,
      error: {
        code: "VALIDATION_ERROR",
        message: `Request ${err.validationContext} doesn't match the schema`,
        details: err.validation,
      },
    } satisfies ApiError);
  }

  // Response serialization errors
  if (isResponseSerializationError(err)) {
    return reply.code(500).send({
      success: false,
      error: {
        code: "INTERNAL_ERROR",
        message: "Response doesn't match the schema",
      },
    } satisfies ApiError);
  }

  // Domain errors
  if (err instanceof NotFoundError) {
    return reply.code(404).send({
      success: false,
      error: { code: err.code, message: err.message },
    } satisfies ApiError);
  }

  if (err instanceof UnauthorizedError || err instanceof ForbiddenError) {
    return reply.code(err.statusCode as 401 | 403).send({
      success: false,
      error: { code: err.code, message: err.message },
    } satisfies ApiError);
  }

  // Fastify internal errors
  if (err instanceof FastifyError) {
    return reply.code(err.statusCode ?? 500).send({
      success: false,
      error: { code: err.code, message: err.message },
    } satisfies ApiError);
  }

  // Unknown — log the real error, return sanitised
  request.log.error(err);
  return reply.code(500).send({
    success: false,
    error: {
      code: "INTERNAL_ERROR",
      message: "An unexpected error occurred",
    },
  } satisfies ApiError);
};
```

### Zod env validation at startup (typed config)

```ts
// apps/backend/src/env.ts
import { z } from "zod/v4";

const envSchema = z.object({
  NODE_ENV: z.enum(["development", "production", "test"]),
  PORT: z.coerce.number().int().positive().default(3000),
  HOST: z.string().default("0.0.0.0"),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url().optional(),
  LOG_LEVEL: z
    .enum(["trace", "debug", "info", "warn", "error", "fatal"])
    .default("info"),
  JWT_SECRET: z.string().min(32),
});

export type EnvConfig = z.output<typeof envSchema>;

export function loadEnv(): EnvConfig {
  const result = envSchema.safeParse(process.env);
  if (!result.success) {
    console.error("Invalid environment variables:", result.error.toString());
    process.exit(1);
  }
  return result.data;
}
```

### Graceful shutdown handler

```ts
// apps/backend/src/shutdown.ts
import type { FastifyInstance } from "fastify";

const SHUTDOWN_TIMEOUT_MS = 25_000;

export function registerShutdownHandlers(app: FastifyInstance) {
  const shutdown = async (signal: string) => {
    app.log.info(`Received ${signal}. Starting graceful shutdown...`);

    const forceExit = setTimeout(() => {
      app.log.error("Forced shutdown after timeout");
      process.exit(1);
    }, SHUTDOWN_TIMEOUT_MS);
    forceExit.unref();

    try {
      await app.close();
      clearTimeout(forceExit);
      app.log.info("Server closed successfully");
      process.exit(0);
    } catch (err) {
      app.log.error("Error during shutdown");
      app.log.error(err);
      process.exit(1);
    }
  };

  process.on("SIGTERM", () => shutdown("SIGTERM"));
  process.on("SIGINT", () => shutdown("SIGINT"));
}
```

### main.ts — bootstrap entry point

```ts
// apps/backend/src/main.ts
import { buildApp } from "./app.js";
import { registerShutdownHandlers } from "./shutdown.js";

async function main() {
  const app = await buildApp();

  registerShutdownHandlers(app);

  try {
    const address = await app.listen({
      port: app.config.PORT,
      host: app.config.HOST,
    });
    app.log.info(`Server listening at ${address}`);
  } catch (err) {
    app.log.error("Failed to start server");
    app.log.error(err);
    process.exit(1);
  }
}

main();
```

## Sources used

- [Fastify v5 Migration Guide](https://fastify.dev/docs/latest/Guides/Migration-Guide-V5/) — official
- [Fastify TypeScript Reference](https://fastify.dev/docs/latest/Reference/TypeScript/) — official
- [Fastify Plugins Reference](https://fastify.dev/docs/latest/Reference/Plugins/) — official
- [Fastify Encapsulation Reference](https://fastify.dev/docs/v5.6.x/Reference/Encapsulation/) — official
- [Fastify Type Providers Reference](https://fastify.dev/docs/v5.7.x/Reference/Type-Providers/) — official
- [Fastify Hooks Reference](https://fastify.io/docs/latest/Reference/Hooks/) — official
- [Fastify Server docs](https://fastify.dev/docs/latest/Reference/Server/) — official
- [@fastify/type-provider-zod — GitHub (turkerdev)](https://github.com/turkerdev/fastify-type-provider-zod) — official (Fastify org)
- [fastify-plugin — GitHub](https://github.com/fastify/fastify-plugin) — official
- [Fastify v5 CHANGELOG](https://github.com/fastify/fastify/releases) — official
- [Fastify PR #6308 — setErrorHandler error typed as unknown](https://github.com/fastify/fastify/pull/6308) — official
- [Fastify PR #5768 — getDecorator / setDecorator](https://github.com/fastify/fastify/pull/5768) — official
- [Fastify issue #6279 / PR #6281 — Zod JSON Schema support](https://github.com/fastify/fastify/issues/6279) — official
- [@fastify/env v6.0.0 release](https://github.com/fastify/fastify-env) — official
- [@fastify/type-provider-zod@1.0.0 (npm)](https://www.npmjs.com/package/@fastify/type-provider-zod) — official
