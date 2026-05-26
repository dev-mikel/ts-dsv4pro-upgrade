# Skill: TypeScript — D4: Runtime Validation with Zod

## Summary

Runtime validation with Zod 4.4.3 (stable, released 2026-05-04). Schemas are the single source of truth for runtime validation and compile-time types in `@shared/schemas`, consumed by all three apps: frontend (React Hook Form), backend (Fastify type provider), and workers (BullMQ job payloads).

**Audit risk:** High. **Dependencies:** D1 (TS Core, strict mode required), D3 (Monorepo workspace).

**Knowledge cutoff:** 2026-05-25.

---

## Version landscape

### Current stable: Zod 4.4.3

Zod 4 shipped as stable after ~24 minor versions of Zod 3. It requires **TypeScript 5.5+** with `strict: true`, `strictNullChecks: true`, `strictFunctionTypes: true`, and `noImplicitThis: true`.

Key high-level changes from v3:
- **Import path:** `zod/v4` (or `zod/v4/core` for library code)
- **~6.5x faster** parsing for `z.object()`
- **100x fewer TS instantiations** for `z.object()`
- All string format methods promoted to top-level (`z.email()`, `z.uuid()`, etc.)
- `.strict()`, `.passthrough()`, `.strip()` deprecated — use `z.strictObject()` / `z.looseObject()`
- `.superRefine()` deprecated — use `.check()`
- `.format()` and `.flatten()` deprecated — use `z.treeifyError()`
- `.merge()` deprecated, `.deepPartial()` dropped, `z.nativeEnum()` deprecated
- `z.interface()` was a beta feature — **removed before stable** (PR #4316, May 2025)
- Recursive schemas use JavaScript getters — no more `z.lazy()` needed
- Defaults short-circuit on `undefined` and must match the **output** type

### Integration compatibility

| Integration | Package | Min Version | Status |
|---|---|---|---|
| React Hook Form | `@hookform/resolvers` | 5.1.0+ (latest 5.4.0) | Zod 4 supported via `zod/v4` import |
| Fastify | `@fastify/type-provider-zod` | 1.0.0 | Zod 4.2+ required; built-in JSON Schema conversion used internally |

### Migration

An official codemod exists: `npx codemod jssg run zod-3-4`. Most projects migrate in under a day. Deprecated v3 APIs still work (with warnings) but will be removed in Zod 5.

---

## Core schema patterns

### Primitives

```ts
import { z } from "zod/v4";

// Strings
z.string();
z.string().min(1).max(255);
z.string().length(10);

// Numbers
z.number();
z.number().int();
z.number().positive();
z.number().min(0).max(100);
z.number().multipleOf(5);

// Booleans
z.boolean();

// Dates
z.date();
z.date().min(new Date("2024-01-01"));

// Literals (now supports multiple values)
z.literal("active");
z.literal("active", "inactive", "pending");  // Zod 4: multiple literals

// Enums
z.enum(["admin", "user", "guest"]);

// Native TypeScript enums (z.nativeEnum() deprecated — z.enum() handles both)
enum Color { Red = "red", Green = "green", Blue = "blue" }
const ColorSchema = z.enum(Color);
// Access via .enum: ColorSchema.enum.Red  (NOT .Enum or .Values — both removed)

// Stringbool (Zod 4 new)
z.stringbool(); // accepts "true"/"false" strings, outputs boolean
```

### String format validators (top-level in Zod 4)

The chained forms (`z.string().email()`) are deprecated. Use the top-level functions:

```ts
z.email();          // was z.string().email()
z.email({ regex: /custom/ });  // custom regex (new in v4)
z.url();            // was z.string().url()
z.uuid();           // was z.string().uuid() — stricter (validates RFC 9562 variant bits)
z.guid();           // permissive UUID-like pattern
z.ipv4();           // was z.string().ip() with version option
z.ipv6();
z.cidrv4();         // was z.string().cidr()
z.cidrv6();
z.base64();
z.base64url();      // no padding allowed
z.emoji();
z.cuid();
z.cuid2();
z.ulid();
z.nanoid();
z.iso.datetime();   // was z.string().datetime()
z.iso.date();
z.iso.time();
z.iso.duration();
```

### Object schemas

```ts
// Default: strips unknown keys (same as Zod 3)
const User = z.object({
  name: z.string(),
  email: z.email(),
});

// Strict: errors on unknown keys (was z.object({...}).strict())
const StrictUser = z.strictObject({
  name: z.string(),
});

// Loose: allows unknown keys through (was z.object({...}).passthrough())
const LooseUser = z.looseObject({
  name: z.string(),
});

// catchall: validates unknown keys against a schema
const WithTags = z.object({
  id: z.string(),
}).catchall(z.string()); // all extra keys must be string values
```

**Breaking change:** `z.object({})` and `z.strictObject({})` infer as `Record<string, never>`, not `{}`. If you need an extensible empty base, start with at least one property.

### Arrays, tuples, records, maps, sets

```ts
// Arrays
z.array(z.string());
z.string().array();  // method form still works

// Non-empty arrays
z.array(z.string()).nonempty();

// Tuples
z.tuple([z.string(), z.number()]);

// Records
z.record(z.string(), z.number());          // { [k: string]: number }
z.record(z.string(), z.number(), { 0: 42 }); // with default
z.partialRecord(z.string(), z.number());    // allows missing keys at parse
z.looseRecord(z.string(), z.number());      // allows extra unknown keys

// Maps
z.map(z.string(), z.number());

// Sets
z.set(z.string());
```

### Union, discriminated union, intersection

```ts
// Union
const Result = z.union([z.string(), z.number()]);
// or: z.string().or(z.number()) — still works

// Discriminated union (upgraded in v4 — supports unions and pipes as members)
const ApiResponse = z.discriminatedUnion("status", [
  z.object({ status: z.literal("success"), data: z.array(z.string()) }),
  z.object({ status: z.literal("error"), message: z.string(), code: z.number() }),
]);

// Intersection
const Combined = z.intersection(BaseSchema, ExtendedSchema);

// XOR (exactly one of the sets of properties must be present)
const XORed = z.xor(
  z.object({ email: z.email() }),
  z.object({ phone: z.string() }),
);
```

### Optional, nullable, nullish

```ts
// Optional: key may be absent OR value may be undefined
z.string().optional();

// Nullable: value may be null
z.string().nullable();

// Nullish: value may be null OR undefined
z.string().nullish();

// In object shapes, declare as optional key:
z.object({
  name: z.string(),
  bio: z.string().optional(),   // bio may be absent or undefined
  age: z.number().nullable(),    // age must be present but may be null
  nickname: z.string().nullish(), // may be absent, undefined, or null
});
```

### Unknown, any, never, void

```ts
z.unknown();  // accepts anything — type-safe (forces narrowing)
z.any();      // accepts anything — opt-out of type checking (avoid)
z.never();    // accepts nothing (useful for discriminated unions)
z.void();     // accepts undefined only
```

**Rule:** Prefer `z.unknown()` over `z.any()`. `z.any()` is an escape hatch that disables type checking on the output — use only when you genuinely cannot type the data.

---

## Type inference

### z.infer — primary pattern

```ts
const UserSchema = z.object({
  id: z.string().uuid(),
  name: z.string(),
  email: z.email(),
});

type User = z.infer<typeof UserSchema>;
// { id: string; name: string; email: string }
```

`z.infer<T>` extracts the **output type** (equivalent to `z.output<T>`).

### z.input vs z.output — when they differ

`z.input` and `z.output` diverge when a schema uses `.transform()`, `.preprocess()`, or `.coerce`:

```ts
const LengthSchema = z.string().transform(val => val.length);
// z.input  → string
// z.output → number

const CoercedNum = z.coerce.number();
// In Zod 4: z.input → unknown (was `string` in Zod 3)
// z.output → number
```

If you need the input type narrower than `unknown` with coercion, pass an explicit generic:

```ts
const CoercedNum = z.coerce.number<number>();
// z.input → number
```

### Exporting inferred types alongside schemas

The co-location pattern: every schema file exports the schema (`const`) and its inferred type (`type`) together:

```ts
// packages/shared/schemas/user.ts
import { z } from "zod/v4";

export const UserSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1),
  email: z.email(),
});

export type User = z.infer<typeof UserSchema>;

// For schemas with transforms, export input type too:
export const CreateUserSchema = z.object({
  name: z.string().min(1),
  email: z.email(),
}).transform(data => ({ ...data, createdAt: new Date() }));

export type CreateUserInput = z.input<typeof CreateUserSchema>;
export type CreateUserOutput = z.output<typeof CreateUserSchema>;
```

---

## Schema composition

### .extend() — add fields to an object

```ts
const Base = z.object({ id: z.string(), createdAt: z.date() });
const WithName = Base.extend({ name: z.string() });
```

Accepts a **shape object** `{ key: schema }`, not a full schema. Conflicts: right overrides.

### .pick() and .omit()

```ts
const Full = z.object({ id: z.string(), name: z.string(), email: z.email(), password: z.string() });

// Array syntax
const Public = Full.pick(["id", "name"]);

// Mask syntax (Zod 4+)
const Safe = Full.omit({ password: true });
```

### .partial() and .required()

```ts
const Required = z.object({ name: z.string(), age: z.number() });
const Optional = Required.partial();   // all fields optional
const RequiredAgain = Optional.required();  // all fields required again
```

`.deepPartial()` was **dropped** in Zod 4. For deep partial, apply `.partial()` on nested objects manually.

### .brand() — nominal typing

```ts
const UserId = z.string().uuid().brand<"UserId">();
type UserId = z.infer<typeof UserId>; // string & z.$brand<"UserId">

const PostId = z.string().uuid().brand<"PostId">();

// UserId and PostId are incompatible despite both being strings
function getUser(id: UserId) { /* ... */ }
getUser("some-uuid" as PostId); // TypeScript error
```

In Zod 4.2+, `.brand()` accepts a second generic to control brand direction.

### Recursive schemas (getters, not z.lazy)

Zod 4 uses JavaScript getters for recursive types. `z.lazy()` is no longer needed:

```ts
// Self-referential
const Category = z.object({
  name: z.string(),
  get subcategories() {
    return z.array(Category);
  },
});
type Category = z.infer<typeof Category>;
// { name: string; subcategories: Category[] }

// Mutually recursive
const User = z.object({
  email: z.email(),
  get posts() {
    return z.array(Post);
  },
});

const Post = z.object({
  title: z.string(),
  get author() {
    return User;
  },
});
```

Key advantages over the old `z.lazy()`:
- No manual type casting required
- Recursive schemas are plain `ZodObject` instances with full method support (`.pick()`, `.omit()`, `.partial()`, `.extend()`, etc.)
- TypeScript infers recursive types correctly

**Avoid nesting function calls** in recursive definitions — prefer methods (`.nullable()`, `.array()`) over functions (`z.nullable()`, `z.array()`) to avoid premature type evaluation.

### Generic schema factories

```ts
// Zod 4 simplified generics: ZodType<Output, Input> (no Def generic)
function paginatedResponse<T extends z.ZodType>(itemSchema: T) {
  return z.object({
    data: z.array(itemSchema),
    total: z.number().int(),
    page: z.number().int(),
    pageSize: z.number().int(),
  });
}

const UserPageSchema = paginatedResponse(UserSchema);
type UserPage = z.infer<typeof UserPageSchema>;
```

Note: `z.ZodTypeAny` was removed in Zod 4. Use `z.ZodType` directly, or constrain with `z.ZodType<unknown, unknown>` if needed.

### What's deprecated or dropped

| Method | Status in Zod 4 |
|---|---|
| `.merge()` | Deprecated (strictness inheritance issues + TS perf) |
| `.deepPartial()` | Dropped |
| `.strict()` / `.passthrough()` / `.strip()` | Deprecated (instance methods) |
| `z.strictObject()` / `z.looseObject()` | Preferred (top-level functions) |
| `z.nativeEnum()` | Deprecated — `z.enum()` handles both |
| `z.lazy()` | Still exists for non-object contexts, but getters are preferred |
| `z.lazyobject` | Removed |

---

## Refinements and transforms

### .refine() — custom validation

```ts
const Password = z.string()
  .refine(val => val.length >= 8, { error: "Password must be at least 8 characters" })
  .refine(val => /[A-Z]/.test(val), { error: "Password must contain an uppercase letter" });
```

Key changes in Zod 4:
- **`ctx.path` is dropped** — not available inside `.refine()`
- Passing a function as the second argument is **dropped**
- Use the unified `error` parameter (not `message`)
- **Type predicates no longer narrow types** in `.refine()`
- `.refine()` can be **async** — must use `.parseAsync()` then

### abort: true — prevent subsequent checks

In Zod 4, **`.transform()` runs even if `.refine()` fails**, unless you use `abort: true`:

```ts
// BUG-PRONE: transform runs on invalid data
const Bad = z.string()
  .refine(val => { try { JSON.parse(val); return true; } catch { return false; } })
  .transform(val => JSON.parse(val)); // runs even if refine fails!

// CORRECT: abort blocks transform on refine failure
const Safe = z.string()
  .refine(val => {
    try { JSON.parse(val); return true; }
    catch { return false; }
  }, { error: "Invalid JSON", abort: true })
  .transform(val => JSON.parse(val)); // only runs if refine passed
```

### .check() — replacement for .superRefine()

`.superRefine()` is deprecated. Use `.check()` instead:

```ts
// Zod 3: .superRefine()
z.string().superRefine((val, ctx) => {
  if (val.length < 5) ctx.addIssue({ code: "custom", message: "Too short" });
});

// Zod 4: .check() — accumulates all errors by default
z.string().check(
  z.minLength(5),
  z.maxLength(100),
  z.refine(val => val.includes("@"), { error: "Must contain @" }),
);
```

`.check()` runs all validations and collects all errors. `.pipe()` aborts on first error — use `.check()` for form validation where you want all field errors, and `.pipe()` for staged transformations.

### .transform() — type-changing transforms

```ts
const StringToLength = z.string().transform(val => val.length);
// z.input → string, z.output → number

// With pipe for post-transform validation:
const Trimmed = z.string()
  .transform(val => val.trim())
  .pipe(z.string().min(1));
```

Critical Zod 4 rule: the order of `.refine()` and `.transform()` matters. `.transform()` runs **after** all non-aborting `.refine()` calls, regardless of whether they passed.

### .overwrite() — transforms that don't change type

New in Zod 4. For transformations that don't change the inferred type:

```ts
z.number().overwrite(val => val ** 2).max(100);
// Returns ZodNumber (not ZodPipe), preserving introspection and JSON Schema conversion
```

### .pipe() — chaining schemas

```ts
z.string()
  .transform(val => val.trim())
  .pipe(z.string().min(1))
  .pipe(z.email());
```

`.pipe()` passes the value through a sequence of schemas, aborting on first error. Use for transformation pipelines where each stage needs valid input.

### .preprocess() — still valid, now returns ZodPipe

```ts
const TrimmedString = z.preprocess(
  (val: unknown) => typeof val === "string" ? val.trim() : val,
  z.string().min(1),
);
// In Zod 4: returns ZodPipe<ZodTransform, ZodString> (was ZodPreprocess in v3)
```

`.preprocess()` is **not deprecated**. It still works but now returns `ZodPipe` internally. The preprocessor function receives `unknown` input and its result is fed to the target schema.

### .default() and .prefault()

```ts
// Zod 4 .default() — short-circuits on undefined, must match OUTPUT type
z.string()
  .transform(val => val.length)
  .default(4);  // ✅ default matches output type (number)

// Use .prefault() for old Zod 3 behavior — default goes through the schema pipeline
z.string()
  .transform(val => val.length)
  .prefault("tuna");  // default is parsed by the schema: "tuna" → 4
```

`.default()` in Zod 4 always applies, even inside optional object fields — if the key is missing, the default populates it. This is a breaking change from Zod 3.

---

## Error handling

### ZodError structure (Zod 4)

In Zod 4, errors from `.safeParse()` **no longer extend `Error`** (performance optimization — avoids call stack snapshot). Errors from `.parse()` still throw `Error` instances.

```ts
interface $ZodError<T = unknown> {
  issues: $ZodIssue[];
  _zod: { output: T; def: $ZodIssue[] };
}
```

`.errors` alias is **dropped** — use `.issues`. `.formErrors` is dropped.

### Issue codes (streamlined from ~16 to 11 in v4)

| Code | Description |
|---|---|
| `invalid_type` | Expected string, received number, etc. |
| `too_big` | Exceeds maximum |
| `too_small` | Below minimum |
| `invalid_string_format` | Email, UUID, URL format failures |
| `not_multiple_of` | Not a multiple of step |
| `unrecognized_keys` | Unknown property in strict object |
| `invalid_value` | Invalid enum/literal/date value (merged from v3) |
| `invalid_union` | No union member matched |
| `invalid_key` | Invalid key in record/map (new in v4) |
| `invalid_element` | Invalid element in map/set (new in v4) |
| `custom` | From `.refine()` failures |

Each issue has `code`, `path: PropertyKey[]`, `message: string`, and `input?: unknown`.

### .safeParse() vs .parse()

```ts
// .parse() — throws on failure (errors extend Error)
try {
  const data = UserSchema.parse(input);
} catch (e) {
  // e is a ZodError (extends Error)
}

// .safeParse() — returns discriminated union (errors do NOT extend Error)
const result = UserSchema.safeParse(input);
if (!result.success) {
  // result.error.issues — no .stack, cheaper to create
} else {
  // result.data — validated output
}
```

**Prefer `.safeParse()` at production boundaries** — no `try/catch`, faster error creation, and explicit control flow.

### Custom error messages

Zod 4 uses a **unified `error` parameter** (replaces `message`, `invalid_type_error`, `required_error`, `errorMap` from v3):

```ts
const Name = z.string().min(1, {
  error: "Name is required",
});

const Age = z.number().int({
  error: (issue) => {
    if (issue.code === "invalid_type") return "Age must be a number";
    return "Age must be a whole number";
  },
});
```

Error maps can return `string`, `{ message: string }`, or `undefined` (to yield to the next error map in the chain).

### Precedence: Schema-level > Per-parse > Global

```ts
// Global (lowest precedence)
z.config({ errorMap: myGlobalMap });

// Per-parse (middle precedence)
schema.parse(data, { errorMap: myParseMap });

// Schema-level (highest precedence — overrides both)
z.string().min(1, { error: "Required" });
```

### Formatting errors

**`.format()` and `.flatten()` are deprecated.** Use the new formatting APIs:

```ts
// Primary: z.treeifyError() — nested structure mirroring the schema
const tree = z.treeifyError(result.error);
// { errors: string[], properties?: { [key]: $ZodErrorTree }, items?: $ZodErrorTree[] }

// Human-readable string
const pretty = z.prettifyError(result.error);
// "✖ Invalid input: expected string, received number\n  → at username"

// Still available but deprecated:
// z.flattenError(result.error) — returns { formErrors, fieldErrors }
```

### Formatting for API responses

```ts
function formatZodError(error: z.$ZodError): { message: string; field?: string }[] {
  return error.issues.map(issue => ({
    message: issue.message,
    field: issue.path.length > 0 ? issue.path.join(".") : undefined,
  }));
}
```

### Formatting for form field errors

```ts
function formatFormErrors(error: z.$ZodError): Record<string, string> {
  const tree = z.treeifyError(error);
  const fieldErrors: Record<string, string> = {};

  function walk(node: typeof tree, prefix = "") {
    if (node.errors.length > 0) {
      fieldErrors[prefix] = node.errors[0];
    }
    if (node.properties) {
      for (const [key, child] of Object.entries(node.properties)) {
        walk(child, prefix ? `${prefix}.${key}` : key);
      }
    }
    if (node.items) {
      node.items.forEach((item, i) => {
        if (item) walk(item, `${prefix}[${i}]`);
      });
    }
  }

  walk(tree);
  return fieldErrors;
}
```

---

## Async validation

### .parseAsync() and .safeParseAsync()

```ts
// Schemas with async refinements or transforms MUST use async parse
const UniqueEmail = z.email().refine(async (email) => {
  const exists = await db.users.findByEmail(email);
  return !exists;
}, { error: "Email already registered" });

const result = await UniqueEmail.safeParseAsync(input);
```

Async refinements have these limitations:
- Cannot be used for synchronous-only code paths
- Serial execution — each async refine runs one at a time
- Adds latency proportional to the number and duration of async checks
- Not suitable for high-throughput validation (workers); use sync schemas for job payloads

---

## Schema organization in @packages/shared

### Directory structure

```
packages/shared/src/schemas/
  user.ts        # Schemas for user domain
  post.ts        # Schemas for post domain
  common.ts      # Reusable primitives (phone, email patterns, etc.)
  api.ts         # API response envelope schemas
  index.ts       # Barrel export
```

### Co-location pattern

Every domain file exports the schema `const` and its inferred `type` together:

```ts
// packages/shared/src/schemas/user.ts
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

### Barrel export

```ts
// packages/shared/src/schemas/index.ts
export {
  UserSchema, type User,
  CreateUserSchema, type CreateUserInput, type CreateUserOutput,
} from "./user";

export {
  PostSchema, type Post,
  CreatePostSchema, type CreatePostInput, type CreatePostOutput,
} from "./post";

export {
  ApiResponseSchema, type ApiResponse,
  PaginatedResponseSchema, type PaginatedResponse,
} from "./api";
```

### Naming convention

Use the `Schema` suffix on the const and a bare name for the type. This avoids name collisions (can't have both named `User`) and makes the schema identifiable at import sites:

```ts
import { UserSchema, type User } from "@shared/schemas";
```

Alternative (no suffix): `export const User = z.object({...})` and `export type User = z.infer<typeof User>`. This works in Zod 4 but can be confusing — the same identifier means different things in value vs type position. The `Schema` suffix is clearer for multi-consumer reuse.

### Multi-consumer considerations

One schema defined once in `@shared/schemas` validates at every boundary:

- **Frontend:** imports schema for React Hook Form `zodResolver(schema)` — form values validated on submit
- **Backend:** Fastify type provider uses schema for request body/params/querystring validation
- **Workers:** job payload validated with `schema.safeParse(payload)` at queue consumption

This means:
- Schemas must be **pure validation** — no transforms that add consumer-specific behavior (like hashing passwords or setting timestamps — those belong in service layer logic)
- Use `z.input` for consumer-facing types (form inputs, API request bodies) and `z.output` for internal types
- Keep schemas focused: one schema per boundary (create, update, response), not one mega-schema
- Avoid `.transform()` in shared schemas that add side effects — transforms should be pure data reshaping

---

## Antipatterns and common mistakes

### Using z.any() as an escape hatch

```ts
// BAD: disables all type checking
const LazySchema = z.object({ data: z.any() });

// GOOD: be explicit about what you accept
const LazySchema = z.object({ data: z.unknown() });

// BETTER: if you know the shape, define it
const LazySchema = z.object({ data: z.record(z.string(), z.unknown()) });
```

### Transforms that break z.input / z.output expectations

```ts
// SURPRISING: z.input is string but default must be number
const Bad = z.string().transform(s => s.length).default(4);
// The default(4) is correct (matches output), but callers may expect string input.
// Be explicit about the input/output contract in shared schemas.
```

### Schemas too tightly coupled to a single consumer

```ts
// BAD: backend-specific middleware baked into schema
const UserSchema = z.object({
  password: z.string().transform(p => bcrypt.hashSync(p)),  // side effect in schema!
});

// GOOD: pure validation only — hashing belongs in the service layer
const UserSchema = z.object({
  password: z.string().min(8),
});
```

### Forgetting safeParse at production boundaries

```ts
// BAD: throws on invalid job payload — crashes worker
const data = JobPayloadSchema.parse(rawPayload);

// GOOD: handles invalid data gracefully
const result = JobPayloadSchema.safeParse(rawPayload);
if (!result.success) {
  logger.error("Invalid job payload", { issues: result.error.issues });
  return;
}
```

### Chaining .refine() without abort: true before .transform()

```ts
// BAD: transform runs on data that failed validation
z.string()
  .refine(val => isValid(val))
  .transform(val => doExpensiveTransform(val));

// GOOD: abort blocks transform when refine fails
z.string()
  .refine(val => isValid(val), { error: "Invalid", abort: true })
  .transform(val => doExpensiveTransform(val));
```

### Using deprecated v3 APIs in new code

```ts
// BAD: deprecated in v4
z.string().email();
z.object({}).strict();
z.nativeEnum(Color);
const err = result.error.format();

// GOOD: v4 APIs
z.email();
z.strictObject({});
z.enum(Color);
const tree = z.treeifyError(result.error);
```

### Duplicate Zod instances in monorepo

Zod 4 uses a global registry singleton. Two copies of Zod in node_modules cause `"ID already exists in registry"` errors. Fix with root `package.json`:

```json
{
  "overrides": {
    "zod": "4.4.3"
  }
}
```

---

## Audit-flagged gaps — resolution

1. **Zod 4.x release status** — RESOLVED. Zod 4.4.3 is stable, released 2026-05-04. Source: GitHub releases API (`tag_name: v4.4.3`, `prerelease: false`).

2. **z.interface()** — RESOLVED. It was a beta feature in early Zod 4 pre-releases. Removed via PR #4316 (merged May 2025) before stable release. It allowed key-level optionality via `"name?"` syntax and recursive schemas via getters. The getter-based recursion was merged into `z.object()`; the key-optionality syntax was dropped. Source: GitHub PR #4316 + web search confirmation.

3. **Inference semantics in 4.x** — RESOLVED. `z.infer` and `z.output` are unchanged from v3 (both extract the output type). `z.input` changed: `z.coerce` types now have `unknown` input instead of their specific type. `.default()` now expects a value matching the output type (not input type). Source: Zod v4 migration guide, GitHub issues #4883, #4140.

4. **RHF resolver compatibility** — RESOLVED. `@hookform/resolvers@5.1.0+` (latest 5.4.0 as of 2026-05-21) supports Zod 4. Usage: `import { zodResolver } from "@hookform/resolvers/zod"` with `import { z } from "zod/v4"`. Source: GitHub react-hook-form/resolvers releases page, PR #777.

5. **Fastify type provider compatibility** — RESOLVED. The official `@fastify/type-provider-zod@1.0.0` (released 2026-04-19) requires Zod 4.2+. The community `fastify-type-provider-zod@6.1.0` also supports Zod 4. Use the official package for new projects. Source: GitHub fastify/type-provider-zod releases, npm registry.

6. **.preprocess() status** — RESOLVED. Still valid, not deprecated. In Zod 4 it returns `ZodPipe<ZodTransform, ZodSchema>` instead of the old `ZodPreprocess`. The preprocessor function operates on `unknown` input. Source: Zod v4 migration guide.

---

## Reference snippets

### Base entity schema with inferred type (co-location pattern)

```ts
import { z } from "zod/v4";

export const UserSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1),
  email: z.email(),
  role: z.enum(["admin", "user", "guest"]),
  createdAt: z.date(),
  updatedAt: z.date(),
});
export type User = z.infer<typeof UserSchema>;
```

### Discriminated union schema for API responses

```ts
export const ApiResponseSchema = z.discriminatedUnion("status", [
  z.object({
    status: z.literal("success"),
    data: z.unknown(),
  }),
  z.object({
    status: z.literal("error"),
    error: z.object({
      message: z.string(),
      code: z.string(),
    }),
  }),
]);
export type ApiResponse<T = unknown> =
  | { status: "success"; data: T }
  | { status: "error"; error: { message: string; code: string } };
```

### Schema with transform (input != output type)

```ts
export const TrimmedString = z.string()
  .transform(val => val.trim())
  .pipe(z.string().min(1));

export type TrimmedStringInput = z.input<typeof TrimmedString>;  // string
export type TrimmedStringOutput = z.output<typeof TrimmedString>; // string
```

### Generic schema factory pattern

```ts
export function paginatedResponse<T extends z.ZodType>(itemSchema: T) {
  return z.object({
    data: z.array(itemSchema),
    total: z.number().int().min(0),
    page: z.number().int().min(1),
    pageSize: z.number().int().min(1).max(100),
  });
}

export const UserPageSchema = paginatedResponse(UserSchema);
export type UserPage = z.infer<typeof UserPageSchema>;
```

### Error formatter for API response

```ts
export function formatApiError(error: z.$ZodError): {
  message: string;
  issues: Array<{ path: string; message: string }>;
} {
  return {
    message: "Validation failed",
    issues: error.issues.map(issue => ({
      path: issue.path.join("."),
      message: issue.message,
    })),
  };
}
```

### Error formatter for form field errors

```ts
export function formatFormErrors(error: z.$ZodError): Record<string, string> {
  const tree = z.treeifyError(error);
  const fields: Record<string, string> = {};

  function walk(node: z.$ZodErrorTree, prefix = "") {
    if (node.errors.length > 0) {
      fields[prefix || "_root"] = node.errors[0];
    }
    if (node.properties) {
      for (const [key, child] of Object.entries(node.properties)) {
        walk(child, prefix ? `${prefix}.${key}` : key);
      }
    }
  }

  walk(tree);
  return fields;
}
```

### Barrel export from @packages/shared/schemas

```ts
// packages/shared/src/schemas/index.ts
export {
  UserSchema, type User,
  CreateUserSchema, type CreateUserInput, type CreateUserOutput,
  UpdateUserSchema, type UpdateUserInput,
} from "./user";

export {
  PostSchema, type Post,
  CreatePostSchema, type CreatePostInput, type CreatePostOutput,
} from "./post";

export {
  ApiResponseSchema, type ApiResponse,
  PaginatedResponseSchema, type PaginatedResponse,
} from "./api";

export {
  phoneSchema,
  emailSchema,
} from "./common";
```

---

## Sources used

- [Zod 4 Release (v4.4.3)](https://github.com/colinhacks/zod/releases/tag/v4.4.3) — official
- [Zod 4 Migration Guide / Changelog](https://zod.dev/v4/changelog) — official
- [Zod v4 Home / Release Notes](https://zod.dev/v4) — official
- [Zod API Reference](https://zod.dev/api) — official
- [Zod v4 Versioning Guide](https://zod.dev/v4/versioning) — official
- [Zod Error Customization](https://zod.dev/error-customization) — official
- [Zod JSON Schema](https://zod.dev/json-schema) — official
- [Zod Packages Overview](https://zod.dev/packages/zod) — official
- [Zod Core Reference](https://zod.dev/packages/core) — official
- [Zod Mini Reference](https://zod.dev/packages/mini) — official
- [Zod GitHub Repository](https://github.com/colinhacks/zod) — official
- [Zod PR #4316 — remove z.interface()](https://github.com/colinhacks/zod/pull/4316) — official
- [Zod PR #4271 — recursive schema getters](https://github.com/colinhacks/zod/pull/4271) — official
- [Zod PR #5941 — restore preprocess on absent keys](https://github.com/colinhacks/zod/pull/5941) — official
- [Zod Issue #4883 — .default() behavior change](https://github.com/colinhacks/zod/issues/4883) — official
- [Zod Issue #5043 — .check() order control](https://github.com/colinhacks/zod/issues/5043) — official
- [Zod Issue #4140 — .default() + .optional() interaction](https://github.com/colinhacks/zod/issues/4140) — official
- [Zod Issue #4941 — .catch() + .optional() bug](https://github.com/colinhacks/zod/issues/4941) — official
- [@hookform/resolvers PR #777 — Zod 4 support](https://github.com/react-hook-form/resolvers/pull/777) — official
- [@fastify/type-provider-zod releases](https://github.com/fastify/type-provider-zod) — official
- [Pockit Zod v4 Migration Guide](https://pockit.tools/blog/zod-v4-migration-guide-breaking-changes-new-features/) — secondary
- [Codemod Zod 3→4 Guide](https://docs.codemod.com/guides/migrations/zod-3-4) — secondary
