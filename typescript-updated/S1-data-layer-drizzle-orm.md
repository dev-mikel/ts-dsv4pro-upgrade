# Skill: TypeScript — S1: Data Layer (Drizzle ORM)

## Summary

Covers the data layer of the monorepo: Drizzle ORM with PostgreSQL via the `postgres.js` driver. Targets **drizzle-orm 0.45.2** (stable), **drizzle-kit 0.31.10**, and **drizzle-zod 0.8.3**. Knowledge cutoff: May 2026. Scope: PostgreSQL only, stable 0.45.x line only — no 1.0.0-beta features.

**Driver:** `postgres.js` 3.4.x (zero dependencies, persistent connection pool).

**Depends on:** D1 (TypeScript foundation), D2 (monorepo structure), D3 (Turborepo), D4 (shared schemas/Zod), D5 (backend architecture), D8 (graceful shutdown), D10 (container lifecycle), S2 (database provisioning).

---

## Architectural rationale

Drizzle is the data layer for the entire monorepo, replacing what would otherwise be three inferior options. The decision rests on one engineering constraint: **types must be derived from the actual schema definition automatically, not maintained by hand.** Type drift between the database and application code is the most dangerous category of data-layer bug — it passes every test and fails only in production.

### Over Prisma

- **Domain isolation:** Prisma's single `schema.prisma` file exposes all tables to every domain via one `PrismaClient` god object. Domain A can query Domain B's tables with no compiler error. Drizzle schemas are TypeScript files split naturally by domain — the compiler enforces what each module imports.
- **Docker footprint:** Prisma's Rust query engine adds ~40-50MB to Docker images and introduces measurable cold-start overhead. Drizzle + `postgres.js` is pure JavaScript with zero native dependencies.
- **Type safety:** Prisma's `$queryRaw` escape hatch loses TypeScript type safety. Drizzle's entire query API is typed end-to-end — even raw SQL via the `sql` template tag produces typed results when annotated.
- **Migration workflow:** Prisma's migration engine is a black box. `drizzle-kit generate` produces readable SQL diffs that teams review before applying — the same workflow as hand-written migrations, but automated.

### Over no ORM (postgres.js + manual types)

- **Type drift eliminated:** Manual TypeScript types (hand-written interfaces for query results) inevitably drift from the actual database schema. Adding a column to a table but forgetting to update the TypeScript type is a silent bug — the compiler sees no error. Drizzle's `InferSelectModel` is always in sync because the table definition is the single source of truth.
- **Migration tooling:** `drizzle-kit` diffs schema changes, generates SQL, and tracks applied migrations in a `__drizzle_migrations` table. Manual SQL migration files require discipline and custom tooling to achieve the same safety.
- **Query builder vs raw SQL:** Drizzle's query builder catches missing columns, wrong types, and invalid operators at compile time. Raw SQL strings catch nothing until runtime.

### Over Kysely

- **Single source of truth:** Kysely requires manual type definitions or a separate codegen step to extract types from the database. Drizzle's schema definition IS the type definition — no codegen, no drift.
- **Native migrations:** `drizzle-kit` handles the full migration lifecycle. Kysely requires separate migration tooling (e.g., `kysely-codegen` + a custom runner).
- **Query ergonomics:** Drizzle's query builder maps more directly to SQL for typical CRUD patterns. Kysely's expression builder is powerful but more verbose for common operations.

### The core argument

Types must be derived from the schema definition automatically. Drizzle is the only option in this stack that satisfies that constraint while also supporting domain-separated schema files, a small Docker footprint, a SQL-close query API, and a built-in migration workflow. Everything else in this document supports that single proposition.

---

## Version landscape

### Stable line (this stack)

| Package | Version | Released |
|---|---|---|
| `drizzle-orm` | **0.45.2** | 2026-03-27 |
| `drizzle-kit` | **0.31.10** | 2026-03-17 |
| `drizzle-zod` | **0.8.3** | 2026 (latest) |
| `postgres` (postgres.js) | **3.4.9** | 2026 |

### 1.0.0-beta line (NOT used)

Drizzle's 1.0.0-beta line introduces breaking changes between beta releases with an explicit team warning that "something will definitely break." It is not production-ready. The beta line includes:
- `defineRelations()` / RQBv2 — a new relations API incompatible with the v1 `relations()` API
- `drizzle-orm/zod` — built-in Zod support that replaces the standalone `drizzle-zod` package
- MSSQL dialect support
- JIT mappers, new casing API, and other internal rewrites

**Policy:** Zero beta dependencies. Zero beta API patterns. Everything in this document is confirmed against 0.45.x stable.

### Key changes in recent stable releases

- **0.32.0:** Identity column support (`generatedAlwaysAsIdentity`)
- **0.30.5:** `$onUpdate` / `$onUpdateFn` for auto-updating columns
- **0.45.0:** Subqueries in `SELECT` fields; no breaking changes from 0.44.x

### Identity columns vs serial

PostgreSQL recommends `GENERATED ALWAYS AS IDENTITY` over `SERIAL` for new schemas. Identity columns are SQL-standard, support sequence options (`startWith`, `increment`), and avoid the sequence ownership issues that `SERIAL` types can cause. Drizzle supports both, but this stack uses identity columns exclusively for primary keys.

---

## /packages/db structure

```
/packages/db
  /src
    /schema                     # TypeScript schema files, one per domain
      users.schema.ts           #   pgTable + relations + InferSelect/InsertModel exports
      orders.schema.ts
      products.schema.ts
      ...
    /repositories               # Typed query functions, one per domain
      users.repository.ts       #   async functions, optional tx parameter
      orders.repository.ts
      products.repository.ts
      ...
    /migrations                 # SQL files generated by drizzle-kit (committed to git)
      0000_initial.sql
      0001_add_orders.sql
      ...
    client.ts                   # drizzle client instantiation + postgres.js connection
    index.ts                    # barrel export — re-exports all schema types + repositories
  drizzle.config.ts             # drizzle-kit configuration
  package.json
```

### Barrel export strategy (`index.ts`)

```ts
// Schema tables + types — consumed by repositories and apps
export { users, usersRelations, type User, type NewUser } from './schema/users.schema';
export { orders, ordersRelations, type Order, type NewOrder } from './schema/orders.schema';

// Repositories — consumed by apps
export { UsersRepository } from './repositories/users.repository';
export { OrdersRepository } from './repositories/orders.repository';
```

### What stays internal

- The raw Drizzle client (`db`) is **never exported** from `@packages/db`. All database access goes through repository functions.
- `client.ts` is internal to `@packages/db`.
- `drizzle.config.ts` is internal to `@packages/db`.

### Architectural rules

- `/apps/*` never import from each other — only from `@packages/db`
- `@packages/db` never imports from any app
- Each domain module imports only its own repository — the TypeScript import graph enforces this
- Domain schemas may reference each other's tables for foreign keys (via `() => otherTable` lazy references)

---

## Schema definition

### pgTable — identity column primary keys

```ts
import { pgTable, integer, text, varchar, boolean, timestamp, uuid, jsonb } from 'drizzle-orm/pg-core';

export const users = pgTable('users', {
  // Primary key — generatedAlwaysAsIdentity() is the PostgreSQL standard
  id: integer().primaryKey().generatedAlwaysAsIdentity(),

  // String types
  name: varchar({ length: 255 }).notNull(),
  email: varchar({ length: 255 }).notNull().unique(),
  bio: text(),

  // Boolean
  isActive: boolean('is_active').notNull().default(true),

  // Timestamps
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true })
    .notNull()
    .defaultNow()
    .$onUpdate(() => new Date()),

  // UUID
  externalId: uuid('external_id').defaultRandom().notNull().unique(),

  // JSONB
  metadata: jsonb('metadata'),
});
```

### Column types used in this stack

| Drizzle type | PostgreSQL type | Usage |
|---|---|---|
| `integer()` | `integer` | Primary keys (with identity), foreign keys, counters |
| `varchar({ length: N })` | `varchar(N)` | Short strings with length constraint (emails, slugs) |
| `text()` | `text` | Unbounded strings (bios, descriptions) |
| `boolean()` | `boolean` | Flags and toggles |
| `timestamp({ withTimezone: true })` | `timestamptz` | All timestamps — always use `withTimezone` |
| `uuid()` | `uuid` | External/public identifiers |
| `jsonb()` | `jsonb` | Semi-structured data, metadata, configuration |
| `numeric({ precision, scale })` | `numeric` | Exact decimal (currency, quantities) |
| `real()` | `real` | Approximate floats (analytics, scores) |

### Identity column options

```ts
// Minimal — recommended default
id: integer().primaryKey().generatedAlwaysAsIdentity(),

// With start value (e.g., for production seed data reserve)
id: integer().primaryKey().generatedAlwaysAsIdentity({ startWith: 10000 }),

// Full sequence control (rarely needed)
id: integer().primaryKey().generatedAlwaysAsIdentity({
  name: 'users_id_seq',
  startWith: 1,
  increment: 1,
  minValue: 1,
  maxValue: 2147483647,
  cache: 1,
}),
```

Use `generatedByDefaultAsIdentity()` only when the application needs to supply its own IDs (e.g., importing data with existing IDs). In all other cases, `generatedAlwaysAsIdentity()` is correct.

### Reusable timestamp pattern

```ts
// Define once, spread across tables
const timestamps = {
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true })
    .notNull()
    .defaultNow()
    .$onUpdate(() => new Date()),
  deletedAt: timestamp('deleted_at', { withTimezone: true }), // soft delete
};

export const orders = pgTable('orders', {
  id: integer().primaryKey().generatedAlwaysAsIdentity(),
  total: numeric({ precision: 10, scale: 2 }).notNull(),
  ...timestamps,
});
```

### $onUpdate — auto-updating columns

`$onUpdate` is a **runtime** feature (drizzle-orm only, no DDL generated). When a row is updated and the column is not explicitly set, the function runs and its return value becomes the column value.

```ts
// For timestamps — returns a JS value
updatedAt: timestamp().$onUpdate(() => new Date()),

// For counters — returns a SQL expression referencing the column itself
updateCounter: integer().default(1).$onUpdateFn(() => sql`${table.updateCounter} + 1`),
```

Two variants:
- `$onUpdate(() => value)` — returns a plain JavaScript value
- `$onUpdateFn(() => sql\`...\`)` — returns a SQL expression that can reference the column

If no `default()` or `$defaultFn()` is set, `$onUpdate` also fires on inserts.

### Foreign keys

```ts
export const orders = pgTable('orders', {
  id: integer().primaryKey().generatedAlwaysAsIdentity(),
  // Lazy reference — () => users.id avoids circular import issues
  userId: integer('user_id')
    .notNull()
    .references(() => users.id, { onDelete: 'cascade' }),
  ...timestamps,
});
```

Always use lazy references (`() => table.column`) to avoid import order problems when schema files reference each other.

### Indexes

```ts
export const users = pgTable('users', {
  id: integer().primaryKey().generatedAlwaysAsIdentity(),
  name: varchar({ length: 255 }).notNull(),
  email: varchar({ length: 255 }).notNull(),
}, (table) => [
  // Composite index
  index('users_name_idx').on(table.name),
  // Unique index
  uniqueIndex('users_email_idx').on(table.email),
]);
```

Indexes are defined as the third argument to `pgTable` (an array of index definitions). This keeps indexes co-located with the table definition.

### Enums

```ts
import { pgEnum } from 'drizzle-orm/pg-core';

export const orderStatusEnum = pgEnum('order_status', [
  'pending',
  'confirmed',
  'shipped',
  'delivered',
  'cancelled',
]);

export const orders = pgTable('orders', {
  id: integer().primaryKey().generatedAlwaysAsIdentity(),
  status: orderStatusEnum().notNull().default('pending'),
  ...timestamps,
});
```

`pgEnum` creates a native PostgreSQL enum type. Define enums at the top of the schema file (before table definitions) so the type exists before the table references it.

### Domain-split schema files

Each domain gets its own schema file. Cross-references use lazy imports:

```ts
// products.schema.ts
import { pgTable, integer, text, numeric } from 'drizzle-orm/pg-core';

export const products = pgTable('products', {
  id: integer().primaryKey().generatedAlwaysAsIdentity(),
  name: text().notNull(),
  price: numeric({ precision: 10, scale: 2 }).notNull(),
});

export type Product = typeof products.$inferSelect;
export type NewProduct = typeof products.$inferInsert;
```

```ts
// orders.schema.ts — references products via lazy import
import { pgTable, integer, numeric } from 'drizzle-orm/pg-core';
import { products } from './products.schema'; // OK — products.schema has no circular dep
import { timestamps } from './_shared';        // shared column patterns

export const orders = pgTable('orders', {
  id: integer().primaryKey().generatedAlwaysAsIdentity(),
  productId: integer('product_id')
    .notNull()
    .references(() => products.id),
  quantity: integer().notNull().default(1),
  ...timestamps,
});
```

---

## Type inference

Drizzle's core value proposition: types are inferred from schema definitions, not maintained by hand.

### InferSelectModel / InferInsertModel

```ts
// Schema file: users.schema.ts
import { pgTable, integer, text } from 'drizzle-orm/pg-core';

export const users = pgTable('users', {
  id: integer().primaryKey().generatedAlwaysAsIdentity(),
  name: text().notNull(),
  email: text().notNull(),
  bio: text(),
  isActive: integer().notNull().default(1),
});

// Type for a SELECT result row — all columns as they exist in the database
export type User = typeof users.$inferSelect;
// { id: number; name: string; email: string; bio: string | null; isActive: number }

// Type for an INSERT — auto-generated/defaulted columns are optional
export type NewUser = typeof users.$inferInsert;
// { id?: number; name: string; email: string; bio?: string | null; isActive?: number }
```

Key properties:
- `$inferSelect` — every column is present, nullable columns are `T | null`, never-optional
- `$inferInsert` — auto-generated columns (`generatedAlwaysAsIdentity`, `defaultNow`, `default()`) become optional; nullable columns without defaults remain required
- These types are always in sync with the table definition — no drift possible

### Exporting types

Every schema file exports its inferred types alongside the table definition. The barrel export (`index.ts`) re-exports them for consumers:

```ts
// Barrel: packages/db/src/index.ts
export { users, type User, type NewUser } from './schema/users.schema';
export { orders, type Order, type NewOrder } from './schema/orders.schema';
```

Apps import types from `@packages/db`:

```ts
import type { User, NewUser } from '@packages/db';
```

### Custom type aliases (optional)

For clarity, some teams prefer `Select*` / `Insert*` naming:

```ts
export type SelectUser = typeof users.$inferSelect;
export type InsertUser = typeof users.$inferInsert;
```

Either convention works — be consistent within the codebase.

---

## Drizzle client setup

### client.ts — singleton pattern

```ts
import { drizzle } from 'drizzle-orm/postgres-js';
import postgres from 'postgres';
import * as schema from './schema';

// postgres.js connection — persistent pool for the process lifetime
const client = postgres(process.env.DATABASE_URL!, {
  max: 20,                    // pool size — tune per deployment
  idle_timeout: 30,           // seconds before idle connections close
  connect_timeout: 10,        // seconds
  prepare: true,              // use server-side prepared statements
});

// Drizzle instance with schema for relational queries
export const db = drizzle({ client, schema });

// Graceful shutdown — close pool on SIGTERM (aligns with D10 container lifecycle)
process.on('SIGTERM', async () => {
  await client.end({ timeout: 5 });
});

// Expose underlying client for migration runner and health checks
export { client };
```

Key points:
- `schema` is passed to `drizzle()` to enable relational queries (`db.query.*`)
- `client.end()` drains active queries before closing connections (consistent with D8/D10 graceful shutdown)
- The `db` export is consumed internally by repositories — it is **not** re-exported from `@packages/db`
- `client` is exported for the migration runner (which needs `max: 1`) and health checks

### drizzle.config.ts

```ts
import { defineConfig } from 'drizzle-kit';

export default defineConfig({
  dialect: 'postgresql',
  schema: './src/schema/*.ts',   // glob — picks up all domain schema files
  out: './src/migrations',       // SQL output directory
  dbCredentials: {
    url: process.env.DATABASE_URL!,
  },
  // Optional: custom migrations table
  migrations: {
    table: '__drizzle_migrations',
    schema: 'public',
  },
});
```

---

## Query API

All queries use the typed query builder. The `db` object provides `select`, `insert`, `update`, `delete`.

### Select

```ts
import { eq, and, or, gt, lt, gte, lte, like, ilike, inArray, isNull, isNotNull } from 'drizzle-orm';

// Basic select
const allUsers = await db.select().from(users);

// With where
const user = await db
  .select()
  .from(users)
  .where(eq(users.id, 42));

// Multiple conditions
const results = await db
  .select()
  .from(users)
  .where(and(
    eq(users.isActive, true),
    gt(users.createdAt, new Date('2025-01-01')),
  ));

// OR conditions
const results = await db
  .select()
  .from(users)
  .where(or(
    eq(users.role, 'admin'),
    eq(users.role, 'moderator'),
  ));

// Partial columns
const names = await db
  .select({ id: users.id, name: users.name })
  .from(users);

// Limit, offset, orderBy
const page = await db
  .select()
  .from(users)
  .orderBy(users.createdAt)
  .limit(20)
  .offset(0);

// Conditional filters — undefined values are ignored
const filters: SQL[] = [];
if (searchTerm) filters.push(ilike(users.name, `%${searchTerm}%`));
if (role) filters.push(eq(users.role, role));
const results = await db.select().from(users).where(and(...filters));
```

### Insert

```ts
// Single insert with returning
const [newUser] = await db
  .insert(users)
  .values({ name: 'Alice', email: 'alice@example.com' })
  .returning();
// newUser is fully typed (User / InferSelectModel)

// Bulk insert
const newUsers = await db
  .insert(users)
  .values([
    { name: 'Alice', email: 'alice@example.com' },
    { name: 'Bob', email: 'bob@example.com' },
  ])
  .returning();

// Insert without returning (when you don't need the result)
await db.insert(users).values({ name: 'Charlie', email: 'charlie@example.com' });
```

### Update

```ts
// Update with returning
const [updated] = await db
  .update(users)
  .set({ name: 'Alice Updated' })
  .where(eq(users.id, 42))
  .returning();

// Conditional set — only update provided fields
await db
  .update(users)
  .set({
    ...(name && { name }),
    ...(email && { email }),
  })
  .where(eq(users.id, 42));
```

### Delete

```ts
// Delete with returning
const [deleted] = await db
  .delete(users)
  .where(eq(users.id, 42))
  .returning();

// Soft delete (preferred over hard delete in most domains)
await db
  .update(users)
  .set({ deletedAt: new Date() })
  .where(eq(users.id, 42));
```

### Joins

```ts
// Inner join — joined columns are non-null
const result = await db
  .select({
    orderId: orders.id,
    userName: users.name,
    total: orders.total,
  })
  .from(orders)
  .innerJoin(users, eq(orders.userId, users.id));

// Left join — joined columns are T | null
const result = await db
  .select({
    userId: users.id,
    orderId: orders.id,
  })
  .from(users)
  .leftJoin(orders, eq(users.id, orders.userId));
// result[0].orderId is typed as number | null

// Nested object pattern — whole nested object is nullable instead of each field
const result = await db
  .select({
    userId: users.id,
    userName: users.name,
    order: {
      id: orders.id,
      total: orders.total,
    },
  })
  .from(users)
  .leftJoin(orders, eq(users.id, orders.userId));
// result[0].order is typed as { id: number; total: number } | null
```

### Where operators — full reference

| Operator | Import | Usage |
|---|---|---|
| `eq` | `drizzle-orm` | `eq(table.col, value)` |
| `ne` | `drizzle-orm` | `ne(table.col, value)` |
| `gt` | `drizzle-orm` | `gt(table.col, value)` |
| `gte` | `drizzle-orm` | `gte(table.col, value)` |
| `lt` | `drizzle-orm` | `lt(table.col, value)` |
| `lte` | `drizzle-orm` | `lte(table.col, value)` |
| `like` | `drizzle-orm` | `like(table.col, '%pattern%')` |
| `ilike` | `drizzle-orm` | `ilike(table.col, '%pattern%')` — case-insensitive |
| `inArray` | `drizzle-orm` | `inArray(table.col, [1, 2, 3])` |
| `isNull` | `drizzle-orm` | `isNull(table.col)` |
| `isNotNull` | `drizzle-orm` | `isNotNull(table.col)` |
| `and` | `drizzle-orm` | `and(cond1, cond2)` |
| `or` | `drizzle-orm` | `or(cond1, cond2)` |

---

## Relations and Relational Queries (stable v1)

### relations() — the v1 API

The stable 0.45.x line uses the v1 `relations()` API imported from `drizzle-orm/relations`. This is **not** `defineRelations()` (RQBv2, beta-only).

```ts
import { relations } from 'drizzle-orm/relations';

// users.schema.ts — one user has many orders
export const usersRelations = relations(users, ({ many }) => ({
  orders: many(orders),
}));

// orders.schema.ts — each order belongs to one user
export const ordersRelations = relations(orders, ({ one }) => ({
  user: one(users, {
    fields: [orders.userId],
    references: [users.id],
  }),
}));
```

### One-to-one

```ts
export const profilesRelations = relations(profiles, ({ one }) => ({
  user: one(users, {
    fields: [profiles.userId],
    references: [users.id],
  }),
}));
```

### One-to-many

```ts
export const usersRelations = relations(users, ({ many }) => ({
  orders: many(orders),
  posts: many(posts),
}));
```

### Many-to-many (junction table)

```ts
// usersToGroups.ts — junction table
export const usersToGroups = pgTable('users_to_groups', {
  userId: integer('user_id').notNull().references(() => users.id),
  groupId: integer('group_id').notNull().references(() => groups.id),
}, (table) => [
  // No single-column PK — composite primary better suits junction tables
]);

export const usersToGroupsRelations = relations(usersToGroups, ({ one }) => ({
  user: one(users, {
    fields: [usersToGroups.userId],
    references: [users.id],
  }),
  group: one(groups, {
    fields: [usersToGroups.groupId],
    references: [groups.id],
  }),
}));

export const usersRelations = relations(users, ({ many }) => ({
  usersToGroups: many(usersToGroups),
}));

export const groupsRelations = relations(groups, ({ many }) => ({
  usersToGroups: many(usersToGroups),
}));
```

### Self-referencing relations (disambiguation with relationName)

```ts
export const usersRelations = relations(users, ({ one, many }) => ({
  // Who invited this user
  invitedBy: one(users, {
    fields: [users.invitedById],
    references: [users.id],
    relationName: 'invitedBy',
  }),
  // Who this user invited
  invitees: many(users, { relationName: 'invitedBy' }),
}));
```

### Relational queries — db.query.*

With `schema` passed to `drizzle({ client, schema })`, relational queries are available on `db.query.<table>`:

```ts
// Find many with included relations
const usersWithOrders = await db.query.users.findMany({
  with: {
    orders: true,
  },
});
// usersWithOrders[0].orders is typed as Order[]

// Nested relations
const usersWithOrdersAndProducts = await db.query.users.findMany({
  with: {
    orders: {
      with: {
        product: true,
      },
    },
  },
});

// Filter on relation
const usersWithRecentOrders = await db.query.users.findMany({
  with: {
    orders: {
      where: (orders, { gt }) => gt(orders.createdAt, new Date('2025-01-01')),
    },
  },
});

// Limit and order relation results
const usersWithLatestOrders = await db.query.users.findMany({
  with: {
    orders: {
      limit: 5,
      orderBy: (orders, { desc }) => [desc(orders.createdAt)],
    },
  },
});

// Column selection
const partialUsers = await db.query.users.findMany({
  columns: { id: true, name: true },
  with: {
    orders: {
      columns: { id: true, total: true },
    },
  },
});

// Find first (single result or undefined)
const user = await db.query.users.findFirst({
  where: (users, { eq }) => eq(users.email, 'alice@example.com'),
});

// Find first or throw
const user = await db.query.users.findFirstOrThrow({
  where: (users, { eq }) => eq(users.id, 42),
});
```

### What NOT to use

- `defineRelations()` — RQBv2, beta only
- `db._query.*` — legacy v1 query syntax; use `db.query.*` instead

---

## Transactions

### db.transaction()

```ts
await db.transaction(async (tx) => {
  // tx has the same API as db — select, insert, update, delete, query
  const [user] = await tx
    .insert(users)
    .values({ name: 'Alice', email: 'alice@example.com' })
    .returning();

  await tx.insert(orders).values({
    userId: user.id,
    total: '99.99',
  });
});
// Both inserts committed atomically. If either fails, both roll back.
```

### Composable pattern — tx as parameter

Repository functions accept an optional `tx` parameter so callers can compose them into a single transaction:

```ts
// users.repository.ts
async function createUser(
  data: NewUser,
  tx: TxDb = db,  // default to non-transactional db
): Promise<User> {
  const [user] = await tx.insert(users).values(data).returning();
  return user;
}

// orders.repository.ts
async function createOrder(
  data: NewOrder,
  tx: TxDb = db,
): Promise<Order> {
  const [order] = await tx.insert(orders).values(data).returning();
  return order;
}

// Caller composes both into one transaction
await db.transaction(async (tx) => {
  const user = await createUser({ name: 'Alice', email: 'alice@example.com' }, tx);
  await createOrder({ userId: user.id, total: '99.99' }, tx);
});
```

The `TxDb` type is inferred from `db`:

```ts
type TxDb = typeof db;
// or more precisely:
import { type PostgresJsDatabase } from 'drizzle-orm/postgres-js';
type TxDb = PostgresJsDatabase<typeof schema> | PostgresJsTransaction<typeof schema>;
```

### Transaction options (PostgreSQL)

```ts
await db.transaction(async (tx) => {
  // ...
}, {
  isolationLevel: 'read committed',  // or 'serializable' | 'repeatable read'
  accessMode: 'read write',         // or 'read only'
  deferrable: true,                 // only with 'serializable' + 'read only'
});
```

### Nested transactions (savepoints)

```ts
await db.transaction(async (tx) => {
  await tx.insert(users).values({ name: 'Alice', email: 'alice@example.com' });

  // Inner transaction becomes a savepoint
  await tx.transaction(async (tx2) => {
    await tx2.insert(orders).values({ userId: 1, total: '99.99' });
  });
  // If inner fails, only the savepoint rolls back — outer continues
});
```

### Rollback

Throw inside the transaction callback to roll back:

```ts
await db.transaction(async (tx) => {
  await tx.insert(users).values(data);
  if (someCondition) {
    throw new Error('Rollback — condition not met');
  }
  await tx.insert(orders).values(moreData);
});
```

Or use `tx.rollback()` which throws a `RollbackError`:

```ts
await db.transaction(async (tx) => {
  await tx.insert(users).values(data);
  if (someCondition) {
    tx.rollback(); // throws RollbackError, rolls back
  }
});
```

---

## Repository pattern

Each domain gets one repository file. Each operation is one async function. The raw `db` client never leaves `@packages/db`.

### Anatomy of a repository function

```ts
// packages/db/src/repositories/users.repository.ts
import { eq, and, ilike, type SQL } from 'drizzle-orm';
import { db } from '../client';
import { users, type User, type NewUser } from '../schema/users.schema';

// Type for the transaction-capable database
type TxDb = typeof db;

export const UsersRepository = {
  async findById(id: number, tx: TxDb = db): Promise<User | undefined> {
    const [result] = await tx
      .select()
      .from(users)
      .where(eq(users.id, id));
    return result;
  },

  async findByEmail(email: string, tx: TxDb = db): Promise<User | undefined> {
    const [result] = await tx
      .select()
      .from(users)
      .where(eq(users.email, email));
    return result;
  },

  async search(
    params: { name?: string; isActive?: boolean; limit?: number; offset?: number },
    tx: TxDb = db,
  ): Promise<User[]> {
    const filters: SQL[] = [];
    if (params.name) filters.push(ilike(users.name, `%${params.name}%`));
    if (params.isActive !== undefined) filters.push(eq(users.isActive, params.isActive));

    return tx
      .select()
      .from(users)
      .where(and(...filters))
      .limit(params.limit ?? 50)
      .offset(params.offset ?? 0);
  },

  async create(data: NewUser, tx: TxDb = db): Promise<User> {
    const [user] = await tx
      .insert(users)
      .values(data)
      .returning();
    return user;
  },

  async update(id: number, data: Partial<NewUser>, tx: TxDb = db): Promise<User | undefined> {
    const [updated] = await tx
      .update(users)
      .set(data)
      .where(eq(users.id, id))
      .returning();
    return updated;
  },

  async softDelete(id: number, tx: TxDb = db): Promise<User | undefined> {
    const [deleted] = await tx
      .update(users)
      .set({ deletedAt: new Date() })
      .where(eq(users.id, id))
      .returning();
    return deleted;
  },
};
```

### Conventions

- Functions are grouped in an exported object (e.g., `UsersRepository`) rather than individual exports — this makes imports cleaner: `UsersRepository.findById(42)` vs tracking individual function imports
- Every mutation returns the affected row(s) via `.returning()` — callers get typed results
- The `tx` parameter defaults to `db` so callers who don't need transactions omit it
- `findById` returns `T | undefined` (not throwing) — callers decide how to handle "not found"
- Search/list functions accept a params object, not positional arguments — extensible without breaking callers

### What stays internal vs exported

| Export | Internal |
|---|---|
| Repository objects | `db` client |
| Schema tables (`users`, `orders`) | `client.ts` |
| Inferred types (`User`, `NewUser`) | `drizzle.config.ts` |
| Relation definitions (`usersRelations`) | Migration files (consumed by drizzle-kit, not by apps) |

---

## drizzle-kit — migration workflow

### Commands

| Command | Purpose | When |
|---|---|---|
| `drizzle-kit generate` | Diffs schema against last snapshot, produces SQL migration file | Development — after schema changes |
| `drizzle-kit migrate` | Applies pending SQL migration files to database | Deployment — before app starts |
| `drizzle-kit push` | Pushes schema directly to DB, no SQL files | **Development only** |
| `drizzle-kit studio` | Launches Drizzle Studio GUI on `127.0.0.1:4983` | Local data inspection |
| `drizzle-kit check` | Checks migrations for race conditions | CI — after generate, before merge |
| `drizzle-kit drop` | Drops all migration tracking — **never** in production | Reset local/dev environment |

### The generate → review → migrate sequence

```
1. Developer changes TypeScript schema
2. drizzle-kit generate  →  produces src/migrations/000N_name.sql
3. Developer reviews the generated SQL (MANDATORY)
4. SQL file is committed to git alongside schema change
5. Deployment pipeline: drizzle-kit migrate  →  applies pending SQL to database
6. App container starts AFTER migrations complete
```

### Migration file format

Default `index` prefix produces sequential files:
```
src/migrations/
  0000_initial.sql
  0001_add_orders_status.sql
  0002_add_products.sql
```

Each file is a standard SQL migration:
```sql
-- 0001_add_orders_status.sql
ALTER TABLE "orders" ADD COLUMN "status" "order_status" DEFAULT 'pending' NOT NULL;
```

### drizzle-kit push — development only

`drizzle-kit push` bypasses SQL file generation and applies schema changes directly. It is for rapid prototyping. **Never use in production.** In production, the SQL migration file must be reviewed and committed before it reaches the database.

### Migration runner

Migrations run as a separate step before container startup — NOT during application initialization:

```ts
// packages/db/src/migrate.ts — called from deployment scripts, NOT from app
import { drizzle } from 'drizzle-orm/postgres-js';
import { migrate } from 'drizzle-orm/postgres-js/migrator';
import postgres from 'postgres';

const migrationClient = postgres(process.env.DATABASE_URL!, { max: 1 });
const db = drizzle(migrationClient);

await migrate(db, { migrationsFolder: './src/migrations' });
await migrationClient.end();
```

Key: `max: 1` is required by postgres.js for the migration runner. The migration client is separate from the application's connection pool and is closed immediately after migrations complete.

---

## Drizzle + Zod integration

### Standalone drizzle-zod (0.45.x stable)

In the 0.45.x stable line, Zod integration is provided by the **separate `drizzle-zod` package** — NOT `drizzle-orm/zod` (which is 1.0.0-beta only).

```bash
npm install drizzle-zod@^0.8.3 zod
```

```ts
import { createInsertSchema, createSelectSchema } from 'drizzle-zod';
import { users } from './schema/users.schema';

// Select schema — validates data coming OUT of the database
export const userSelectSchema = createSelectSchema(users);

// Insert schema — validates data going INTO the database
// Auto-generated columns (identity, defaultNow) are automatically omitted
export const userInsertSchema = createInsertSchema(users);
```

### Refinements — field-level customization

```ts
export const userInsertSchema = createInsertSchema(users, {
  name: (schema) => schema.min(1).max(255),
  email: (schema) => schema.email(),
  bio: (schema) => schema.max(1000).optional(),
});

// userInsertSchema is a Zod schema — use it in your API layer
type ValidatedInsert = z.infer<typeof userInsertSchema>;
```

### Relationship with @shared/schemas (D4)

This stack has two sources of type information:
1. **Drizzle schema** — the database table definitions in `@packages/db/src/schema/`
2. **`@shared/schemas`** — shared Zod validation schemas defined in D4

The relationship:
- **Drizzle types are the source of truth for the database shape.** `InferSelectModel` and `InferInsertModel` describe what the database accepts and returns.
- **`drizzle-zod` generates Zod schemas from Drizzle types.** This is useful when the API validation schema matches the database schema exactly (simple CRUD).
- **`@shared/schemas` is the source of truth for business-logic validation.** When validation rules go beyond database constraints (cross-field validation, conditional requirements, domain-specific rules), use hand-crafted Zod schemas in `@shared/schemas` rather than `drizzle-zod`-generated ones.

Rule of thumb:
- Simple CRUD endpoint with no extra validation → `drizzle-zod` is sufficient, no need for `@shared/schemas`
- Any business logic validation beyond "is this a valid row?" → define the Zod schema in `@shared/schemas`, reference Drizzle types for consistency but don't auto-generate

The two should agree on the base shape (column types, nullability) but may differ on additional constraints. Drizzle owns the database shape; `@shared/schemas` owns the API contract.

### createSelectSchema vs createInsertSchema behavior

| Function | Includes identity columns? | Includes defaultNow columns? | Use case |
|---|---|---|---|
| `createSelectSchema` | Yes (present in output) | Yes | Validating API responses, webhook payloads |
| `createInsertSchema` | No (auto-omitted) | No (auto-omitted) | Validating API request bodies |
| `createUpdateSchema` | Yes (for WHERE) | Yes (for WHERE) | Validating PATCH/PUT request bodies |

---

## Turborepo integration

### drizzle-kit generate as a codegen task

```jsonc
// turbo.json
{
  "tasks": {
    "@packages/db#generate": {
      "dependsOn": ["^build"],
      "inputs": ["src/schema/**/*.ts"],
      "outputs": ["src/migrations/**/*.sql"],
      "cache": false  // Never cache — migrations are a side effect of schema state
    }
  }
}
```

`drizzle-kit generate` should NOT be cached. The migration output depends on external state (the database schema at time of generation), not just the TypeScript source files. Cache hits would produce stale migration files.

### Migration in deployment flow

Migrations run **outside Turborepo** — they are a deployment step, not a build step:

```
Build pipeline:
  1. turbo build          → compiles TypeScript
  2. docker build         → produces container image
  3. (no migration here)

Deployment pipeline:
  1. drizzle-kit migrate  → applies pending SQL to target database
  2. docker run           → starts application container
```

Migrations must complete before the application starts. If the application connects before migrations run, it may query columns that don't exist yet.

---

## Antipatterns and common mistakes

### Using serial() instead of generatedAlwaysAsIdentity()

`serial()` is legacy PostgreSQL. Use `integer().primaryKey().generatedAlwaysAsIdentity()` for all new tables. Identity columns are SQL-standard and give you control over sequence options.

### Running drizzle-kit push in production

`push` bypasses SQL migration files — no review step, no audit trail, no rollback path. Production deployments must use `generate` → review → `migrate`.

### Running migrations at container startup

If migrations fail (e.g., conflicting changes from another deploy), the app starts anyway and may serve requests against an incorrect schema. Migrations must be a separate step that blocks container startup.

### Exporting the raw db client outside @packages/db

If `db` is exported, any module can execute arbitrary SQL — bypassing repositories, types, and domain isolation. The compiler can't help when someone writes `db.select().from(users)` in an app that shouldn't touch the `users` table.

### Using 1.0.0-beta in production

The Drizzle team explicitly warns that beta releases "will definitely break." Breaking changes between beta versions are expected and not documented as migrations.

### Manual types alongside Drizzle (defeats the purpose)

If you maintain hand-written TypeScript interfaces for your database rows, you've recreated the type-drift problem Drizzle was chosen to eliminate. Trust `$inferSelect` and `$inferInsert`.

### Importing drizzle-orm/zod in 0.45.x

`drizzle-orm/zod` does not exist in the 0.45.x npm package. It's a 1.0.0-beta feature. Use the standalone `drizzle-zod` package.

### Forgetting .returning() on mutations

Without `.returning()`, `insert`/`update`/`delete` return void in PostgreSQL. If the caller needs the created/updated/deleted row, they must chain `.returning()`.

### Not reviewing generated migration SQL

`drizzle-kit generate` produces SQL based on a diff algorithm. It can produce unexpected DDL (e.g., dropping and recreating a column instead of altering it). Always read the generated SQL before committing it.

### Using $onUpdate for DDL

`$onUpdate` is a runtime ORM feature — it does not produce SQL in generated migration files. The column still needs a `defaultNow()` or similar SQL-level default for the schema to be correct.

---

## Reference snippets

### packages/db/src/schema/users.schema.ts

```ts
import { pgTable, integer, text, varchar, boolean, timestamp, uniqueIndex } from 'drizzle-orm/pg-core';
import { relations } from 'drizzle-orm/relations';

export const users = pgTable('users', {
  id: integer().primaryKey().generatedAlwaysAsIdentity(),
  name: varchar({ length: 255 }).notNull(),
  email: varchar({ length: 255 }).notNull(),
  bio: text(),
  isActive: boolean('is_active').notNull().default(true),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true })
    .notNull()
    .defaultNow()
    .$onUpdate(() => new Date()),
}, (table) => [
  uniqueIndex('users_email_idx').on(table.email),
]);

export const usersRelations = relations(users, ({ many }) => ({
  orders: many(orders),
}));

export type User = typeof users.$inferSelect;
export type NewUser = typeof users.$inferInsert;
```

### packages/db/src/schema/orders.schema.ts (foreign key, enum)

```ts
import { pgTable, integer, numeric, pgEnum } from 'drizzle-orm/pg-core';
import { relations } from 'drizzle-orm/relations';
import { users } from './users.schema';

export const orderStatusEnum = pgEnum('order_status', [
  'pending', 'confirmed', 'shipped', 'delivered', 'cancelled',
]);

export const orders = pgTable('orders', {
  id: integer().primaryKey().generatedAlwaysAsIdentity(),
  userId: integer('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  status: orderStatusEnum().notNull().default('pending'),
  total: numeric({ precision: 10, scale: 2 }).notNull(),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true })
    .notNull()
    .defaultNow()
    .$onUpdate(() => new Date()),
});

export const ordersRelations = relations(orders, ({ one }) => ({
  user: one(users, {
    fields: [orders.userId],
    references: [users.id],
  }),
}));

export type Order = typeof orders.$inferSelect;
export type NewOrder = typeof orders.$inferInsert;
```

### packages/db/src/client.ts

```ts
import { drizzle } from 'drizzle-orm/postgres-js';
import postgres from 'postgres';
import * as schema from './schema';

const client = postgres(process.env.DATABASE_URL!, {
  max: 20,
  idle_timeout: 30,
  connect_timeout: 10,
  prepare: true,
});

export const db = drizzle({ client, schema });

process.on('SIGTERM', async () => {
  await client.end({ timeout: 5 });
});

export { client };
```

### packages/db/src/repositories/users.repository.ts (full annotated)

```ts
import { eq, and, ilike, type SQL } from 'drizzle-orm';
import { db } from '../client';
import { users, type User, type NewUser } from '../schema/users.schema';

type TxDb = typeof db;

export const UsersRepository = {
  async findById(id: number, tx: TxDb = db): Promise<User | undefined> {
    const [result] = await tx.select().from(users).where(eq(users.id, id));
    return result;
  },

  async findByEmail(email: string, tx: TxDb = db): Promise<User | undefined> {
    const [result] = await tx.select().from(users).where(eq(users.email, email));
    return result;
  },

  async search(
    params: { name?: string; isActive?: boolean; limit?: number; offset?: number },
    tx: TxDb = db,
  ): Promise<User[]> {
    const filters: SQL[] = [];
    if (params.name) filters.push(ilike(users.name, `%${params.name}%`));
    if (params.isActive !== undefined) filters.push(eq(users.isActive, params.isActive));
    return tx.select().from(users).where(and(...filters)).limit(params.limit ?? 50).offset(params.offset ?? 0);
  },

  async create(data: NewUser, tx: TxDb = db): Promise<User> {
    const [user] = await tx.insert(users).values(data).returning();
    return user;
  },

  async update(id: number, data: Partial<NewUser>, tx: TxDb = db): Promise<User | undefined> {
    const [updated] = await tx.update(users).set(data).where(eq(users.id, id)).returning();
    return updated;
  },

  async softDelete(id: number, tx: TxDb = db): Promise<User | undefined> {
    const [deleted] = await tx.update(users).set({ deletedAt: new Date() }).where(eq(users.id, id)).returning();
    return deleted;
  },
};
```

### packages/db/src/index.ts (barrel export)

```ts
export { users, usersRelations, type User, type NewUser } from './schema/users.schema';
export { orders, ordersRelations, orderStatusEnum, type Order, type NewOrder } from './schema/orders.schema';
export { products, productsRelations, type Product, type NewProduct } from './schema/products.schema';

export { UsersRepository } from './repositories/users.repository';
export { OrdersRepository } from './repositories/orders.repository';
export { ProductsRepository } from './repositories/products.repository';
```

### drizzle.config.ts

```ts
import { defineConfig } from 'drizzle-kit';

export default defineConfig({
  dialect: 'postgresql',
  schema: './src/schema/*.ts',
  out: './src/migrations',
  dbCredentials: {
    url: process.env.DATABASE_URL!,
  },
});
```

### Composable transaction — two repositories in one tx

```ts
import { db } from '../client';
import { UsersRepository } from '../repositories/users.repository';
import { OrdersRepository } from '../repositories/orders.repository';

async function registerUserWithInitialOrder(userData: NewUser, orderData: NewOrder): Promise<{
  user: User;
  order: Order;
}> {
  return db.transaction(async (tx) => {
    const user = await UsersRepository.create(userData, tx);
    const order = await OrdersRepository.create(
      { ...orderData, userId: user.id },
      tx,
    );
    return { user, order };
  });
}
```

### drizzle-zod integration with @shared/schemas

```ts
// packages/db/src/schema/users.schema.ts — Drizzle table + inferred types
export const users = pgTable('users', { /* ... */ });
export type User = typeof users.$inferSelect;
export type NewUser = typeof users.$inferInsert;

// packages/shared/src/schemas/user.schemas.ts — API validation (from D4)
import { z } from 'zod';
import type { NewUser } from '@packages/db';

// When validation matches DB shape exactly, drizzle-zod is sufficient:
// export const createUserSchema = createInsertSchema(users);

// When business rules go beyond DB constraints, hand-craft:
export const createUserSchema = z.object({
  name: z.string().min(1).max(255),
  email: z.string().email(),
  bio: z.string().max(1000).optional(),
  // Business rule: must accept terms
  acceptedTerms: z.literal(true),
}) satisfies z.ZodType<Omit<NewUser, 'id' | 'createdAt' | 'updatedAt'> & { acceptedTerms: boolean }>;
```

---

## Sources used

- [PostgreSQL Get Started (New)](https://orm.drizzle.team/docs/get-started/postgresql-new) — official
- [SQL Schema Declaration](https://orm.drizzle.team/docs/sql-schema-declaration) — official
- [PostgreSQL Column Types](https://orm.drizzle.team/docs/column-types/pg) — official
- [Indexes & Constraints](https://orm.drizzle.team/docs/indexes-constraints) — official
- [Relations (v1 API)](https://orm.drizzle.team/docs/relations) — official
- [Select Query API](https://orm.drizzle.team/docs/select) — official
- [Joins](https://orm.drizzle.team/docs/joins) — official
- [Transactions](https://orm.drizzle.team/docs/transactions) — official
- [drizzle.config.ts Reference](https://orm.drizzle.team/docs/drizzle-config-file) — official
- [Migrations](https://orm.drizzle.team/docs/migrations) — official
- [Zod Integration (drizzle-zod)](https://orm.drizzle.team/docs/zod) — official
- [Drizzle ORM GitHub Releases](https://github.com/drizzle-team/drizzle-orm/releases) — official
- [Drizzle ORM 0.45.0 Release](https://github.com/drizzle-team/drizzle-orm/releases/tag/0.45.0) — official
- [Drizzle ORM 0.30.5 Release — $onUpdate](https://orm.drizzle.team/docs/latest-releases/drizzle-orm-v0305) — official
- [Drizzle Studio](https://orm.drizzle.team/docs/drizzle-studio) — official
- [PostgreSQL Timestamp Default](https://orm.drizzle.team/docs/guides/timestamp-default-value) — official
- [Include/Exclude Columns](https://orm.drizzle.team/docs/guides/include-or-exclude-columns) — official
- [Unique Case-Insensitive Email](https://orm.drizzle.team/docs/guides/unique-case-insensitive-email) — official
- [PostgreSQL Schemas with Enums](https://orm.drizzle.team/docs/schemas) — official
- [postgres.js GitHub](https://github.com/porsager/postgres) — official
