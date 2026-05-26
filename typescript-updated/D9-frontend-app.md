# Skill: TypeScript — D9: Frontend App

## Summary

Frontend application layer covering two options: **Option A — Next.js App Router** (primary) and **Option B — React + Vite SPA** (secondary). The frontend consumes types and Zod schemas directly from `@packages/shared` (D5) — no codegen, no OpenAPI client, no tRPC. The same Zod schemas used in the backend (D8) are imported directly in Server Actions and form validation.

**Audit risk:** TIER 1 CRITICAL — 8 flagged gaps all resolved. **Dependencies:** D1 (TS 6.0.3), D2 (compiler), D3 (Turborepo), D4 (Zod 4.4.3), D5 (shared types), D6 (testing), D7 (ESLint with `no-restricted-imports` to block `/apps/backend` imports), D8 (Fastify backend). Knowledge cutoff: 2026-05-25.

**CRITICAL — Caching model inversion:** Next.js 15+ changed the fundamental default: `fetch()` is **NOT cached by default**. Route handlers (GET) are **NOT cached by default**. Page segments are **NOT reused from the client cache** on `<Link>`/`useRouter` navigation. This is the exact inverse of Next.js 14. Writing Next.js 14 caching patterns in a 15/16 codebase produces code that fetches on every request where the developer expects a cache hit. Every caching behavior must be explicit.

## Version landscape

| Package | Version | Notes |
|---|---|---|
| Next.js | **16.2.6** (May 2026) | Current stable. 15.5.18 is the latest 15.x. |
| Next.js (Option A baseline) | **15.5+ / 16.x** | This skill documents 15+ behavior. 14.x caching model is explicitly out of scope. |
| React | **19.2.6** (May 2026) | Current stable. 19.1.7 also maintained. |
| TanStack Query (Option B) | **5.x** (latest) | `@tanstack/react-query` v5. Current stable. |
| @hookform/resolvers (Option B) | **5.4.0** (May 2026) | ZodResolver compatible with Zod 3.x. Zod 4.x needs 5.2.2+ or `standardSchemaResolver`. |
| TypeScript | **6.0.3** (D1 baseline) | Minimum 5.5+ as declared in scope. |
| Node.js | 20.9+ | Next.js 16 dropped Node.js 18. |

### Next.js 14 → 15 caching model changes (CRITICAL)

These three inversions are the most probable source of bugs when a developer with Next.js 14 experience writes 15/16 code:

| Behavior | Next.js 14 (default) | Next.js 15+ (default) | Opt-in syntax (15+) |
|---|---|---|---|
| `fetch()` caching | Cached | **NOT cached** | `fetch(url, { cache: 'force-cache' })` |
| GET route handler caching | Cached | **NOT cached** | `export const dynamic = 'force-static'` |
| Client cache on navigation | Reused page segments | **NOT reused** | `staleTimes: { dynamic: 30, static: 180 }` in `next.config` |

**Codemod available:** `npx @next/codemod@canary upgrade latest`

Source: [Next.js 14→15 Upgrade Guide](https://nextjs.org/docs/app/guides/upgrading/version-15)

### React 18 → 19 breaking API changes

| React 18 API | React 19 replacement | Notes |
|---|---|---|
| `useFormState` (from `react-dom`) | `useActionState` (from `react`) | Different import, different signature, includes `isPending` |
| `forwardRef` | `ref` as regular prop | `forwardRef` still works but no longer needed. Will be deprecated. |
| `<Context.Provider value={...}>` | `<Context value={...}>` | `.Provider` still works, will be deprecated in future |
| `ReactDOM.render` | `createRoot` | Removed in React 19 |
| `defaultProps` on function components | Default parameters | Removed for function components |
| String refs | Ref callbacks or `useRef`/`createRef` | Removed entirely |

Source: [React 19 Upgrade Guide](https://react.dev/blog/2024/04/25/react-19-upgrade-guide), [React 19 Release](https://react.dev/blog/2024/12/05/react-19)

---

─── OPTION A: Next.js App Router ───────────────────────

## Routing and file conventions

### Directory structure

```
app/
  layout.tsx              ← Root layout (required)
  page.tsx                ← Home page (/)
  loading.tsx             ← Loading UI for / (Suspense boundary)
  error.tsx               ← Error boundary for / and children
  not-found.tsx           ← 404 page
  (marketing)/            ← Route group (doesn't affect URL)
    layout.tsx
    about/page.tsx        ← /about
  dashboard/
    layout.tsx            ← Shared layout for /dashboard/*
    page.tsx              ← /dashboard
    settings/page.tsx     ← /dashboard/settings
    [id]/page.tsx          ← /dashboard/:id (dynamic segment)
  api/
    route.ts              ← Route handler at /api
```

### Route groups: `(group)` syntax

Route groups use parentheses to organize files without affecting the URL path. The segment `(marketing)` does NOT appear in the URL.

```tsx
// app/(marketing)/layout.tsx — applies to /about, /pricing, etc.
// app/(marketing)/about/page.tsx — renders at /about (NOT /marketing/about)
```

Use cases: different layouts for marketing vs dashboard sections, organizing without changing URLs.

### Dynamic routes and typed params

**params is now a Promise** in Next.js 15+. This is a breaking change from 14.

```tsx
// app/dashboard/[id]/page.tsx
// ── Async Page (Server Component) ──

type Params = Promise<{ id: string }>;
type SearchParams = Promise<{ [key: string]: string | string[] | undefined }>;

export default async function Page(props: {
  params: Params;
  searchParams: SearchParams;
}) {
  const { id } = await props.params;
  // id is typed as string — guaranteed
}
```

```tsx
// ── Sync Page (Client Component, or using the `use` hook) ──
'use client';

import { use } from 'react';

type Params = Promise<{ id: string }>;

export default function Page(props: { params: Params }) {
  const { id } = use(props.params);
  // ...
}
```

**Catch-all and optional catch-all routes:**

| File | Route | params type |
|---|---|---|
| `[...slug]/page.tsx` | `/a/b/c` | `Promise<{ slug: string[] }>` |
| `[[...slug]]/page.tsx` | `/` or `/a/b` | `Promise<{ slug?: string[] }>` |

### Parallel routes and intercepting routes (overview)

- **Parallel routes**: `@modal`, `@sidebar` folders — render multiple pages in the same layout simultaneously. Accessed via `props.modal`, `props.sidebar` in the layout.
- **Intercepting routes**: `(.)photo`, `(..)photo` — intercept navigation to show content in a modal instead of full-page navigation.

Both are advanced patterns. This skill covers the core types; see [Next.js Parallel Routes](https://nextjs.org/docs/app/building-your-application/routing/parallel-routes) for full documentation.

### Typed navigation hooks

```tsx
'use client';

import { useRouter, usePathname, useSearchParams } from 'next/navigation';

export function NavigationComponent() {
  const router = useRouter();
  const pathname = usePathname();  // string
  const searchParams = useSearchParams();  // ReadonlyURLSearchParams

  // Programmatic navigation — type-safe with typedRoutes
  router.push('/dashboard/settings');
  router.refresh(); // Re-fetch server data without full reload
}
```

### Typed routes (typedRoutes)

**Status: STABLE in Next.js 16.** No longer `experimental.typedRoutes` — use `typedRoutes` directly.

```ts
// next.config.ts
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  typedRoutes: true,
};

export default nextConfig;
```

When enabled, `href` on `<Link>` and `router.push()` are type-checked against actual route paths. Dynamic and catch-all params also get typed:

```tsx
import Link from 'next/link';

// ✅ Compiles — /dashboard/settings exists
<Link href="/dashboard/settings">Settings</Link>

// ❌ Type error — unknown route
<Link href="/dashboard/typo">Typo</Link>
```

### Link component

```tsx
import Link from 'next/link';

// Static href
<Link href="/dashboard">Dashboard</Link>

// Dynamic href with typed params (typedRoutes enabled)
<Link href={`/dashboard/${id}`}>User {id}</Link>

// With search params
<Link href={{ pathname: '/search', query: { q: 'next.js' } }}>Search</Link>
```

## Server vs Client Components — the mental model

### What runs where

| | Server Component (RSC) | Client Component |
|---|---|---|
| **Runs on** | Server only (never sent to client) | Server (SSR) + Client (hydration) |
| **File marker** | None (default) | `'use client'` at top of file |
| **Hooks** | No | Yes |
| **Event handlers** | No | Yes |
| **Browser APIs** | No | Yes |
| **async/await** | Yes | No (use hooks, Suspense) |
| **Direct DB access** | Yes | No |
| **Bundle size** | Zero JS sent to client | Full component JS shipped |

### 'use client' directive

The `'use client'` directive marks the boundary where client-side code begins. It must be the first statement in the file. Everything imported by a client component becomes part of the client bundle.

```tsx
'use client';  // Must be first line

import { useState } from 'react';

export function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

**Rules:**
- Add `'use client'` when you need hooks (`useState`, `useEffect`, etc.), event handlers (`onClick`, `onChange`), or browser APIs (`window`, `localStorage`).
- Server Components cannot import Client Components directly without consequences — the client boundary propagates through imports. However, you CAN pass a Server Component's rendered output as `children` to a Client Component.
- A file with `'use client'` still gets server-rendered (SSR) for the initial HTML — the directive only controls whether the component also runs on the client.

### Composing RSC and Client Components — the children pattern

This is the key mental model for RSC composition:

```tsx
// app/layout.tsx — Server Component (default, no directive)
// Can be async, can fetch data, can access DB.

import { ClientSidebar } from './client-sidebar';
import { ServerContent } from './server-content';

export default async function Layout({ children }: { children: React.ReactNode }) {
  const user = await getUser(); // Server-side DB call

  return (
    <ClientSidebar user={user}>
      {/* RSC output passed as children to a Client Component — stays on server */}
      <ServerContent />
    </ClientSidebar>
  );
}
```

```tsx
// app/client-sidebar.tsx
'use client';

export function ClientSidebar({
  children,
  user,
}: {
  children: React.ReactNode;
  user: { name: string };
}) {
  return (
    <div>
      <span>{user.name}</span>
      {children}  {/* Server-rendered content rendered here */}
    </div>
  );
}
```

The pattern: pass serializable props to Client Components, pass RSC-rendered JSX as `children`.

### When NOT to use RSC

Use Client Components (`'use client'`) when you need:
- `useState`, `useEffect`, `useReducer`, `useRef`, custom hooks
- Event handlers: `onClick`, `onChange`, `onSubmit`
- Browser APIs: `window`, `localStorage`, `navigator`, `IntersectionObserver`
- Real-time subscriptions: WebSocket, SSE, polling
- Third-party libraries that use any of the above

## Data fetching and caching (CRITICAL SECTION)

### The default: fetch() is NOT cached

In Next.js 15+, `fetch()` makes a fresh network request on every render by default. This is the inverse of Next.js 14.

```tsx
// app/page.tsx — Server Component
export default async function Page() {
  // This fetches fresh data on EVERY request. No cache. No stale data.
  const data = await fetch('https://api.example.com/products');
  const products = await data.json();

  return <ProductList products={products} />;
}
```

### Opting INTO caching

**Per-request caching with `cache: 'force-cache'`:**

```tsx
// Cache this specific fetch indefinitely (or until on-demand revalidation)
const data = await fetch('https://api.example.com/products', {
  cache: 'force-cache',
});
```

**Time-based revalidation with `next.revalidate`:**

```tsx
// Cache for 60 seconds, then revalidate on next request (stale-while-revalidate)
const data = await fetch('https://api.example.com/products', {
  next: { revalidate: 60 },
});
```

**Tag-based on-demand revalidation:**

```tsx
// Cache with tags for targeted invalidation
const data = await fetch('https://api.example.com/products', {
  next: { tags: ['products'] },
});
```

### Opting OUT of caching (explicit)

Since the default is NO caching, this is only needed to override a segment-level config or to be explicit:

```tsx
const data = await fetch('https://api.example.com/products', {
  cache: 'no-store',
});
```

### Route segment config

Control caching at the page/layout/route level:

```tsx
// Force fully dynamic rendering (every request is fresh)
export const dynamic = 'force-dynamic';

// Force fully static rendering (error if dynamic APIs used)
export const dynamic = 'force-static';

// Default — cache as much as possible without breaking dynamic behavior
export const dynamic = 'auto';

// Default revalidation interval (seconds)
export const revalidate = 3600; // Revalidate at most every hour

// Override fetch cache behavior for the entire segment
export const fetchCache = 'default-cache'; // Opt all fetches into caching
```

### unstable_cache — status and migration path

`unstable_cache` is still available in Next.js 16 but **deprecated** in favor of the `'use cache'` directive (Cache Components). It remains functional with a deprecation warning.

```tsx
import { unstable_cache } from 'next/cache';

// Legacy pattern — still works but consider migration
const getCachedUser = unstable_cache(
  async (id: string) => getUserFromDb(id),
  ['user'], // cache key prefix
  {
    tags: ['user'],
    revalidate: 3600,
  }
);

const user = await getCachedUser('123');
```

**Preferred replacement (Next.js 16, stable):**

```tsx
// Enable in next.config.ts first:
// cacheComponents: true

async function getUser(id: string) {
  'use cache';
  cacheTag('user');
  cacheLife('hours');
  return getUserFromDb(id);
}
```

### revalidatePath() and revalidateTag()

Used inside Server Actions or Route Handlers to invalidate cached data on demand:

```tsx
// app/actions.ts
'use server';

import { revalidatePath, revalidateTag } from 'next/cache';

export async function updateProduct(formData: FormData) {
  // ... mutation ...

  // Invalidate all cached data for /products
  revalidatePath('/products');

  // Or invalidate by tag — targets any fetch tagged with 'products'
  revalidateTag('products');
}
```

### Parallel vs sequential data fetching

```tsx
// ❌ Sequential — each fetch blocks the next (waterfall)
export default async function Page() {
  const user = await fetch('/api/user');      // Wait...
  const posts = await fetch('/api/posts');    // Then wait...
  return <PageContent user={await user.json()} posts={await posts.json()} />;
}

// ✅ Parallel — all fetches start simultaneously
export default async function Page() {
  const [user, posts] = await Promise.all([
    fetch('/api/user'),
    fetch('/api/posts'),
  ]);

  return (
    <PageContent
      user={await user.json()}
      posts={await posts.json()}
    />
  );
}
```

Use sequential when the second fetch depends on data from the first. Use parallel when fetches are independent.

```tsx
// Sequential — necessary when posts depends on userId
export default async function Page() {
  const userRes = await fetch('/api/user');
  const user = await userRes.json();
  const postsRes = await fetch(`/api/users/${user.id}/posts`);
  const posts = await postsRes.json();
}
```

### React cache() for request deduplication

Not to be confused with HTTP caching. React's `cache()` deduplicates calls within a single render pass — useful for ORM/database calls:

```tsx
import { cache } from 'react';

export const getUser = cache(async (id: string) => {
  return db.select().from(users).where(eq(users.id, id));
});

// Calling getUser('123') in multiple components within the same request
// only executes the DB query once.
```

## Server Actions

### 'use server' directive

**File-level** — all exports are Server Actions:

```ts
// app/actions.ts
'use server';

import { revalidatePath } from 'next/cache';

export async function createPost(formData: FormData) {
  // Runs on the server
  const title = formData.get('title');
  // ... mutate database ...
  revalidatePath('/posts');
}
```

**Function-level** — single function in a Server Component:

```tsx
// app/page.tsx — Server Component
export default function Page() {
  async function createPost(formData: FormData) {
    'use server';
    // Only this function runs on the server
  }

  return <form action={createPost}>...</form>;
}
```

### Calling Server Actions from Client Components

Import from a `'use server'` file:

```ts
// app/actions.ts
'use server';

export async function deletePost(id: string) {
  // ...
}
```

```tsx
// app/ui/delete-button.tsx
'use client';

import { deletePost } from '@/app/actions';

export function DeleteButton({ id }: { id: string }) {
  return (
    <button onClick={() => deletePost(id)}>
      Delete
    </button>
  );
}
```

### Form actions

Pass a Server Action directly to the `<form>` `action` prop:

```tsx
// Works in both Server AND Client Components
'use client';

import { createPost } from '@/app/actions';
import { useActionState } from 'react';

export function CreatePostForm() {
  const [state, action, isPending] = useActionState(createPost, null);

  return (
    <form action={action}>
      <input type="text" name="title" required />
      <button type="submit" disabled={isPending}>
        {isPending ? 'Creating...' : 'Create Post'}
      </button>
      {state?.error && <p className="error">{state.error}</p>}
    </form>
  );
}
```

### useActionState (React 19 — stable)

**Import:** `import { useActionState } from 'react';`
**Signature:** `useActionState(action, initialState, permalink?)` → `[state, dispatch, isPending]`

| Parameter | Description |
|---|---|
| `action` | The Server Action or async function. Receives `(prevState, formData)` when used with a form, or `(prevState, ...args)` when called directly. |
| `initialState` | The initial state value (any serializable type). |
| `permalink?` | Optional URL for progressive enhancement before JS loads. |

| Return value | Description |
|---|---|
| `state` | Current state returned by the last action execution |
| `dispatch` | Stable-identity wrapped action — safe to omit from effect deps |
| `isPending` | `true` while the action is running |

**This replaces `useFormState`** (deprecated, was in `react-dom`, had no `isPending`).

```tsx
'use client';

import { useActionState } from 'react';
import { updateName } from './actions';

const initialState = { error: null as string | null, success: false };

export function NameForm() {
  const [state, formAction, isPending] = useActionState(updateName, initialState);

  return (
    <form action={formAction}>
      <input name="name" />
      <button type="submit" disabled={isPending}>
        {isPending ? 'Saving...' : 'Save'}
      </button>
      {state.error && <p className="error">{state.error}</p>}
      {state.success && <p className="success">Saved!</p>}
    </form>
  );
}
```

### Zod validation inside Server Actions

Use the same Zod schemas from `@shared` (D4) — no duplication:

```ts
// app/actions.ts
'use server';

import { z } from 'zod';
import { createPostSchema } from '@shared/schemas'; // From D4/D5
import { revalidatePath } from 'next/cache';

// Infer the return type for the client
export type CreatePostState = {
  error?: string;
  fieldErrors?: Partial<Record<keyof z.infer<typeof createPostSchema>, string>>;
  success: boolean;
};

export async function createPost(
  prevState: CreatePostState,
  formData: FormData,
): Promise<CreatePostState> {
  // Parse FormData into a plain object first
  const raw = {
    title: formData.get('title'),
    content: formData.get('content'),
  };

  // Validate with @shared schema (same schema the backend uses — D8)
  const result = createPostSchema.safeParse(raw);

  if (!result.success) {
    return {
      error: 'Validation failed',
      fieldErrors: result.error.flatten().fieldErrors as any,
      success: false,
    };
  }

  try {
    // Mutate — this could be a DB call, an internal API call, etc.
    await db.insert(posts).values(result.data);

    revalidatePath('/posts');
    return { success: true };
  } catch (err) {
    return { error: 'Failed to create post', success: false };
  }
}
```

### Optimistic updates with useOptimistic

```tsx
'use client';

import { useOptimistic, useActionState } from 'react';
import { addComment } from './actions';
import type { Comment } from '@shared/types';

export function CommentList({ comments }: { comments: Comment[] }) {
  const [optimisticComments, addOptimistic] = useOptimistic(
    comments,
    (state, newComment: Comment) => [...state, newComment],
  );

  const [, action, isPending] = useActionState(
    async (_prev: null, formData: FormData) => {
      const body = formData.get('body') as string;
      // Show it immediately
      addOptimistic({ id: crypto.randomUUID(), body, author: 'You' });
      // Actually save it
      await addComment(formData);
    },
    null,
  );

  return (
    <div>
      {optimisticComments.map((c) => (
        <p key={c.id}>{c.author}: {c.body}</p>
      ))}
      <form action={action}>
        <input name="body" />
        <button disabled={isPending}>Add</button>
      </form>
    </div>
  );
}
```

### Error handling patterns

**Pattern 1: Return error state (recommended for forms)**
The Server Action returns a discriminated union. The client reads the state and displays inline errors.

**Pattern 2: Throw for error boundaries**

```ts
'use server';

export async function dangerousAction() {
  throw new Error('Something went wrong');
  // Caught by nearest error.tsx boundary
}
```

Use error boundaries for unexpected failures (network errors, DB outages). Use returned error state for expected validation failures.

### Typed Server Action return values

Export return types so clients can type the state:

```ts
// app/actions.ts
export type ActionState<T = void> =
  | { success: true; data: T }
  | { success: false; error: string };
```

```tsx
// Client component
'use client';

import { useActionState } from 'react';
import { type ActionState, myAction } from '@/app/actions';

export function Form() {
  const [state, action, pending] = useActionState(
    (_prev: ActionState, formData: FormData) => myAction(formData),
    { success: true, data: undefined },
  );

  if (!state.success) {
    // state.error is typed as string
  }
}
```

## React 19 APIs

### useActionState — STABLE

Covered in full above under Server Actions. Import: `import { useActionState } from 'react'`. Replaces `useFormState` from `react-dom`.

### use hook — STABLE

**Import:** `import { use } from 'react';`

Reads a Promise or Context during render. Unlike hooks, `use` can be called inside conditionals and loops. It works with Suspense for promises and replaces `useContext` for context.

```tsx
'use client';

import { use, Suspense } from 'react';
import type { ThemeContextType } from './theme';

// Reading a Promise (Suspense-powered)
function Comments({ promise }: { promise: Promise<Comment[]> }) {
  const comments = use(promise); // Suspends until resolved
  return <ul>{comments.map((c) => <li key={c.id}>{c.body}</li>)}</ul>;
}

// Reading Context (can be conditional — useContext cannot)
function ThemedButton({ themeContext }: { themeContext: React.Context<ThemeContextType> }) {
  const theme = use(themeContext);
  return <button style={{ background: theme.primary }}>Click</button>;
}

export default function Page({ promise }: { promise: Promise<Comment[]> }) {
  return (
    <Suspense fallback={<div>Loading comments...</div>}>
      <Comments promise={promise} />
    </Suspense>
  );
}
```

**Limitations:**
- `use` does NOT support promises created during render on the client side. Promises must come from a Server Component, a Suspense-compatible library, or a framework.
- `use` cannot be wrapped in try/catch — use Error Boundaries for rejected promises.
- In Server Components, prefer `async/await` over `use`.

### ref as regular prop — STABLE

**`forwardRef` is no longer needed in React 19.** `ref` is now a regular prop.

```tsx
// React 18 (old):
const TextInput = React.forwardRef<HTMLInputElement, { label: string }>(
  ({ label }, ref) => (
    <label>
      {label}
      <input ref={ref} />
    </label>
  )
);

// React 19 (current):
function TextInput({ label, ref }: { label: string; ref?: React.Ref<HTMLInputElement> }) {
  return (
    <label>
      {label}
      <input ref={ref} />
    </label>
  );
}
```

**Key rules:**
- Destructure `ref` BEFORE spreading `...rest` to avoid double-ref bugs
- `forwardRef` still works for backward compatibility but is unnecessary
- Do NOT mix `forwardRef` with ref-as-prop — undefined behavior
- Migration codemod: `npx @next/codemod@canary react-19/replace-forwardref`

### Context.Provider simplification — STABLE

```tsx
// React 18 (old):
<ThemeContext.Provider value="dark">
  <AuthContext.Provider value={user}>
    {children}
  </AuthContext.Provider>
</ThemeContext.Provider>

// React 19 (current) — render the context directly:
<ThemeContext value="dark">
  <AuthContext value={user}>
    {children}
  </AuthContext>
</ThemeContext>
```

`.Provider` still works but will be deprecated in a future version. `useContext()` and `use()` work identically with both syntaxes.

### useOptimistic — STABLE

Covered in the Server Actions section above. Import: `import { useOptimistic } from 'react'`. Provides optimistic UI updates during Server Action execution.

### useFormStatus — STABLE

**Import:** `import { useFormStatus } from 'react-dom';`

Reads the status of a parent `<form>` — must be used in a component rendered inside a `<form>`:

```tsx
'use client';

import { useFormStatus } from 'react-dom';

function SubmitButton() {
  const { pending, data, method, action } = useFormStatus();
  return <button disabled={pending}>{pending ? 'Submitting...' : 'Submit'}</button>;
}
```

## TypeScript patterns in Next.js

### Typing page props

```tsx
// app/dashboard/[id]/page.tsx

type Params = Promise<{ id: string }>;
type SearchParams = Promise<{ [key: string]: string | string[] | undefined }>;

interface PageProps {
  params: Params;
  searchParams: SearchParams;
}

export default async function Page({ params, searchParams }: PageProps) {
  const { id } = await params;
  const { q } = await searchParams;
  // id: string, q: string | string[] | undefined
}
```

### Typing layout props

```tsx
// app/dashboard/layout.tsx

interface LayoutProps {
  children: React.ReactNode;
  params: Promise<{ id?: string }>; // Only present if layout is in a dynamic segment
}

export default async function DashboardLayout({ children, params }: LayoutProps) {
  return <div>{children}</div>;
}
```

### Typing generateMetadata

```tsx
import type { Metadata } from 'next';

type Params = Promise<{ id: string }>;

export async function generateMetadata({
  params,
}: {
  params: Params;
}): Promise<Metadata> {
  const { id } = await params;
  return { title: `Post ${id}` };
}
```

### NEXT_PUBLIC env var typing

```ts
// env.d.ts — in project root
declare namespace NodeJS {
  interface ProcessEnv {
    NEXT_PUBLIC_API_URL: string;
    NEXT_PUBLIC_APP_NAME: string;
    NEXT_PUBLIC_ENABLE_FEATURE_X?: string; // optional
  }
}
```

Non-`NEXT_PUBLIC_` variables are only available in server context (Server Components, Server Actions, Route Handlers). They must not be prefixed or they become exposed to the client.

### next.config.ts

TypeScript config is first-class since Next.js 15:

```ts
// next.config.ts
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  // Stable in Next.js 16 — not experimental
  typedRoutes: true,

  // React Compiler — stable in Next.js 16
  reactCompiler: true,

  // Cache Components — stable in Next.js 16 (replaces unstable_cache)
  cacheComponents: true,

  // Turbopack is the default bundler in Next.js 16
  turbopack: {
    // Custom Turbopack rules if needed
  },

  // Standard Next.js image config
  images: {
    remotePatterns: [
      { protocol: 'https', hostname: 'cdn.example.com' },
    ],
  },

  // For Docker standalone output (S2 intersection)
  output: 'standalone',

  // Server external packages — not bundled, kept as node_modules
  serverExternalPackages: ['@shared'],
};

export default nextConfig;
```

**Key Next.js 16 config changes:**
- `webpack` config option is deprecated — use `turbopack` instead
- `experimental.typedRoutes` removed — use `typedRoutes` (stable)
- `experimental.appDir` removed
- `experimental.serverActions` removed
- `output: 'standalone'` unchanged — still used for Docker deployment (S2)

## Importing @shared types

### Server Component imports

```tsx
// app/users/page.tsx — Server Component
import type { User } from '@shared/types';        // Type-only — zero runtime cost
import { userSchema } from '@shared/schemas';      // Runtime Zod schema
import { UserRole } from '@shared/enums';          // Runtime enum/constants
```

### Client Component imports

```tsx
// app/ui/user-card.tsx — Client Component
'use client';

import type { User } from '@shared/types';         // Type-only — tree-shaken, no bundle hit
// import { userSchema } from '@shared/schemas';   // ⚠️ Runtime import — adds to client bundle
// Only import runtime schemas in client components if doing client-side validation
```

### Type-only imports

Use `import type` for type-only imports in client components — they're erased at compile time:

```tsx
import type { User, Post } from '@shared/types';
import type { CreatePostInput } from '@shared/schemas';
```

### Zod schemas in Server Actions

```ts
// app/actions.ts
'use server';

import { createPostSchema, updatePostSchema } from '@shared/schemas';
import type { CreatePostInput, UpdatePostInput } from '@shared/schemas';
import { revalidatePath } from 'next/cache';

export async function createPost(prevState: any, formData: FormData) {
  const result = createPostSchema.safeParse({
    title: formData.get('title'),
    content: formData.get('content'),
  });

  if (!result.success) {
    return { error: 'Validation failed', fields: result.error.flatten().fieldErrors };
  }

  // result.data is typed as CreatePostInput — inferred from the @shared schema
  const { title, content } = result.data;

  // ... persist ...
  revalidatePath('/posts');
}
```

### Architectural boundary enforcement (D7)

ESLint `no-restricted-imports` (from D7) prevents direct imports from `/apps/backend`:

```json
// .eslintrc — from D7
{
  "rules": {
    "no-restricted-imports": ["error", {
      "patterns": ["@/apps/backend/*", "apps/backend/*"]
    }]
  }
}
```

All shared types flow through `@shared` — the frontend never imports from the backend directly.

---

─── OPTION B: React + Vite (SPA) ───────────────────────

Option B is a traditional Single Page Application using React + Vite. Unlike Option A (Next.js), there is no server-side rendering, no RSC, and no Server Actions. All data fetching is client-side via TanStack Query. Forms use React Hook Form with Zod validation.

Key differences from Option A:
- No Server Components — everything runs in the browser
- No `'use client'` / `'use server'` directives
- Data fetching via TanStack Query, not RSC fetch
- Form handling via React Hook Form + Zod, not Server Actions
- Routing via a client-side router (e.g., TanStack Router or React Router), not file-based

## Project setup

### Vite config with @shared path alias

```ts
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'node:path';

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@shared': path.resolve(__dirname, '../../packages/shared/src'),
    },
  },
  server: {
    port: 5173,
    proxy: {
      '/api': 'http://localhost:3000', // Proxy to Fastify backend (D8)
    },
  },
});
```

### tsconfig alignment with D2

```jsonc
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": true,
    "noEmit": true,
    "baseUrl": ".",
    "paths": {
      "@shared/*": ["../../packages/shared/src/*"]
    }
  },
  "include": ["src"]
}
```

### Entry point

```tsx
// src/main.tsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { App } from './App';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 30_000,       // 30s before considered stale
      retry: 1,
      refetchOnWindowFocus: false,
    },
  },
});

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <QueryClientProvider client={queryClient}>
      <App />
    </QueryClientProvider>
  </StrictMode>,
);
```

## TypeScript component patterns

### Component props typing

Prefer explicit return types with function declarations over `React.FC`:

```tsx
// ✅ Recommended — explicit props interface + explicit return type
interface UserCardProps {
  user: User;
  onSelect?: (id: string) => void;
}

export function UserCard({ user, onSelect }: UserCardProps): React.ReactElement {
  return (
    <div onClick={() => onSelect?.(user.id)}>
      <h3>{user.name}</h3>
      <p>{user.email}</p>
    </div>
  );
}
```

**Why not `React.FC`?** `FC` implicitly includes `children` in the type even when the component doesn't accept children. Explicit declaration avoids this.

### Generic components

```tsx
interface ListProps<T> {
  items: T[];
  renderItem: (item: T, index: number) => React.ReactNode;
}

export function List<T>({ items, renderItem }: ListProps<T>): React.ReactElement {
  return <ul>{items.map((item, i) => <li key={i}>{renderItem(item, i)}</li>)}</ul>;
}

// Usage — T inferred as { id: string; name: string }
<List
  items={users}
  renderItem={(user) => <span>{user.name}</span>}
/>
```

### Hook typing

```tsx
import { useState, useReducer, useRef } from 'react';
import type { User } from '@shared/types';

// useState — explicit generic
const [user, setUser] = useState<User | null>(null);
const [count, setCount] = useState<number>(0);

// useReducer — typed reducer + initial state
type Action = { type: 'SET_USER'; payload: User } | { type: 'CLEAR' };

function userReducer(state: User | null, action: Action): User | null {
  switch (action.type) {
    case 'SET_USER': return action.payload;
    case 'CLEAR': return null;
  }
}

const [state, dispatch] = useReducer(userReducer, null);

// useRef — typed ref
const inputRef = useRef<HTMLInputElement>(null);   // RefObject<HTMLInputElement> — immutable .current
const countRef = useRef<number>(0);                // RefObject<number> — mutable .current

// In React 19: useRef ALWAYS requires an argument
```

### Context pattern

```tsx
// src/context/AuthContext.tsx
import { createContext, useContext, useState, type ReactNode } from 'react';
import type { User } from '@shared/types';

interface AuthContextType {
  user: User | null;
  login: (user: User) => void;
  logout: () => void;
}

const AuthContext = createContext<AuthContextType | null>(null);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);

  return (
    <AuthContext value={{
      user,
      login: setUser,
      logout: () => setUser(null),
    }}>
      {children}
    </AuthContext>
  );
}

export function useAuth(): AuthContextType {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error('useAuth must be used within AuthProvider');
  return ctx;
}
```

### Event handler typing

```tsx
import type { ChangeEvent, FormEvent, MouseEvent, KeyboardEvent } from 'react';

// Input change
function handleChange(e: ChangeEvent<HTMLInputElement>) {
  const value = e.target.value; // string
}

// Select change
function handleSelect(e: ChangeEvent<HTMLSelectElement>) {
  const value = e.target.value; // string
}

// Form submit
function handleSubmit(e: FormEvent<HTMLFormElement>) {
  e.preventDefault();
  const formData = new FormData(e.currentTarget);
}

// Button click
function handleClick(e: MouseEvent<HTMLButtonElement>) {
  e.preventDefault();
}

// Keyboard
function handleKeyDown(e: KeyboardEvent<HTMLInputElement>) {
  if (e.key === 'Enter') { /* ... */ }
}
```

## Data fetching — TanStack Query with @shared types

### Typed fetch wrapper

```ts
// src/api/client.ts
import type { User, Post } from '@shared/types';
import type { CreatePostInput } from '@shared/schemas';

const BASE_URL = '/api';

async function apiFetch<T>(path: string, init?: RequestInit): Promise<T> {
  const res = await fetch(`${BASE_URL}${path}`, {
    headers: { 'Content-Type': 'application/json', ...init?.headers },
    ...init,
  });

  if (!res.ok) {
    throw new Error(`API error: ${res.status} ${res.statusText}`);
  }

  return res.json() as Promise<T>;
}

export const api = {
  getUsers: () => apiFetch<User[]>('/users'),
  getUser: (id: string) => apiFetch<User>(`/users/${id}`),
  getPosts: () => apiFetch<Post[]>('/posts'),
  createPost: (input: CreatePostInput) =>
    apiFetch<Post>('/posts', { method: 'POST', body: JSON.stringify(input) }),
};
```

### Typed useQuery

```tsx
import { useQuery } from '@tanstack/react-query';
import { api } from '@/api/client';
import type { User } from '@shared/types';

export function useUser(id: string) {
  return useQuery<User, Error>({
    queryKey: ['user', id],
    queryFn: () => api.getUser(id),
    enabled: !!id,  // Don't fetch without an id
  });
}
```

### Typed useMutation

```tsx
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { api } from '@/api/client';
import type { Post, CreatePostInput } from '@shared/types';

export function useCreatePost() {
  const queryClient = useQueryClient();

  return useMutation<Post, Error, CreatePostInput>({
    mutationFn: (input) => api.createPost(input),
    onSuccess: (newPost) => {
      // Optimistically update the posts list
      queryClient.setQueryData<Post[]>(['posts'], (old) =>
        old ? [...old, newPost] : [newPost]
      );
    },
  });
}
```

## React Hook Form + Zod (SPA-specific)

### Setup with zodResolver and @shared schema

```tsx
// src/forms/CreatePostForm.tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { createPostSchema } from '@shared/schemas';  // From D4
import type { CreatePostInput } from '@shared/schemas'; // Inferred type
import { useCreatePost } from '@/hooks/useCreatePost';

export function CreatePostForm() {
  const { mutate, isPending } = useCreatePost();

  const {
    register,
    handleSubmit,
    formState: { errors },
  } = useForm<CreatePostInput>({
    resolver: zodResolver(createPostSchema),
    defaultValues: { title: '', content: '' },
  });

  const onSubmit = (data: CreatePostInput) => {
    mutate(data);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('title')} placeholder="Title" />
      {errors.title && <span>{errors.title.message}</span>}

      <textarea {...register('content')} placeholder="Content" />
      {errors.content && <span>{errors.content.message}</span>}

      <button type="submit" disabled={isPending}>
        {isPending ? 'Creating...' : 'Create'}
      </button>
    </form>
  );
}
```

**Key types:**
- `useForm<CreatePostInput>` — the generic matches the Zod-inferred type from `@shared`
- `register('title')` — field name is type-checked against the schema keys
- `errors.title?.message` — typed error message from Zod validation

### @hookform/resolvers compatibility with Zod

| Setup | Compatible? |
|---|---|
| `@hookform/resolvers@5.4.0` + `zod@3.25.x` | Fully compatible. Use `zodResolver`. |
| `@hookform/resolvers@5.2.2+` + `zod@4.x` | Compatible with caveats. May need to clear lockfile. |
| `@hookform/resolvers@5.x` + `zod@4.x` (alternative) | Use `standardSchemaResolver` from `@hookform/resolvers/standard-schema` for spec-compliant integration. |

D4 documents Zod 4.4.3. If using `zodResolver`, ensure `@hookform/resolvers` ≥ 5.2.2.

## Importing @shared types in SPA

Same pattern as Option A but without server/client boundary concerns — everything is bundled to the client:

```tsx
import type { User, Post } from '@shared/types';     // Type-only — zero bundle cost
import { createPostSchema } from '@shared/schemas';   // Runtime validation — bundled
import { UserRole } from '@shared/enums';             // Runtime constants — bundled
```

Use `import type` for type-only imports to ensure tree-shaking. Runtime Zod schemas are included in the bundle — this is expected since client-side validation needs them.

---

─── SHARED ─────────────────────────────────────────────

## Antipatterns and common mistakes

1. **Assuming fetch() is cached (Next.js 14 pattern).** In Next.js 15+, `fetch()` is NOT cached by default. Writing `await fetch('/api/data')` without explicit caching options fetches fresh data on every request. If caching is desired, add `{ cache: 'force-cache' }` or `{ next: { revalidate: N } }`.

2. **Using `useFormState` instead of `useActionState`.** React 19 replaced `useFormState` (was in `react-dom`) with `useActionState` (in `react`). The new hook returns `[state, dispatch, isPending]` — three elements, not two. Import from `'react'`, not `'react-dom'`.

3. **Using `forwardRef` when ref-as-prop suffices.** In React 19, `ref` is a regular prop. `forwardRef` is unnecessary but still works. Write new code without it.

4. **Writing Providers as `<Ctx.Provider>` instead of `<Ctx>`.** React 19 allows `<Context value={...}>` directly. The old `.Provider` syntax still works but is the older pattern.

5. **Defining types inline instead of importing from `@shared`.** Every type used by the frontend that also appears in the backend (DTOs, request bodies, response shapes) must come from `@shared`. Inline type definitions duplicate and diverge.

6. **Importing from `/apps/backend` directly.** The ESLint `no-restricted-imports` rule (D7) blocks this. Use `@shared` as the contract layer.

7. **Not awaiting params/searchParams.** In Next.js 15+, `params` and `searchParams` are `Promise<>` types. They must be awaited or unwrapped with the `use` hook.

8. **Not awaiting cookies() / headers().** These are async in Next.js 15+. Calling them without `await` returns a warning in dev.

9. **Sequential fetches when parallel is possible.** Two independent `fetch()` calls should use `Promise.all()`, not sequential `await`.

10. **Using `React.FC` in Option B.** It implicitly includes `children` even when the component doesn't accept them. Use explicit function declarations with typed props instead.

## Audit-flagged gaps — resolution

| # | Gap | Resolution | Source |
|---|---|---|---|
| 1 | fetch() caching defaults in 15/16 | **NOT cached by default.** Opt-in: `cache: 'force-cache'`. Opt-out: `cache: 'no-store'` (explicit). | [Next.js 14→15 Upgrade Guide](https://nextjs.org/docs/app/guides/upgrading/version-15) |
| 2 | Route handler caching defaults in 15/16 | **NOT cached by default.** Opt-in: `export const dynamic = 'force-static'`. | [Next.js 14→15 Upgrade Guide](https://nextjs.org/docs/app/guides/upgrading/version-15) |
| 3 | Client router cache TTL in 15/16 | **Page segments NOT reused** on `<Link>`/`useRouter` navigation. Opt-in via `staleTimes` experimental config. Layouts and loading states still cached. | [Next.js 14→15 Upgrade Guide](https://nextjs.org/docs/app/guides/upgrading/version-15) |
| 4 | useActionState — import, signature, stability | **Import from `'react'`.** Signature: `useActionState(action, initialState, permalink?)` → `[state, dispatch, isPending]`. **Stable** in React 19.0.0 (Dec 2024). Replaces `useFormState`. | [React 19 Release](https://react.dev/blog/2024/12/05/react-19) |
| 5 | ref as regular prop | **No longer needs `forwardRef`.** `ref` is a regular prop in React 19. `forwardRef` still works but will be deprecated. Codemod: `npx @next/codemod@canary react-19/replace-forwardref`. | [React 19 Release](https://react.dev/blog/2024/12/05/react-19) |
| 6 | typedRoutes stability | **STABLE in Next.js 16.** Use `typedRoutes: true` (not `experimental.typedRoutes`). Type-checks `<Link>` and `router.push()` paths. | [Next.js typedRoutes docs](https://nextjs.org/docs/app/api-reference/config/next-config-js/typedRoutes) |
| 7 | @hookform/resolvers + Zod compatibility | v5.4.0 + Zod 3.x: fully compatible. v5.2.2+ + Zod 4.x: compatible with tested combinations. Alternative: `standardSchemaResolver` for Zod 4.x. | npm releases, GitHub issues |
| 8 | unstable_cache stability | **Still named `unstable_cache` in Next.js 16** but **deprecated** in favor of `'use cache'` directive (Cache Components, stable since 16.0.0). `unstable_cache` still works with deprecation warning. | [Next.js unstable_cache docs](https://nextjs.org/docs/app/api-reference/functions/unstable_cache) |

All 8 gaps resolved. Zero unresolved.

## Reference snippets (Option A)

### app/layout.tsx (root layout, typed)

```tsx
import type { Metadata } from 'next';
import type { ReactNode } from 'react';

export const metadata: Metadata = {
  title: 'My App',
  description: 'A Next.js application',
};

export default function RootLayout({ children }: { children: ReactNode }) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  );
}
```

### app/[id]/page.tsx (typed params, RSC data fetching)

```tsx
import type { User } from '@shared/types';

type Params = Promise<{ id: string }>;

export default async function UserPage({ params }: { params: Params }) {
  const { id } = await params;

  // fetch() is NOT cached by default — fetches fresh per request
  const user: User = await fetch(`https://api.example.com/users/${id}`)
    .then((r) => r.json());

  return <h1>{user.name}</h1>;
}
```

### Server Action with Zod validation and useActionState

```ts
// app/actions.ts
'use server';

import { createPostSchema } from '@shared/schemas';
import type { z } from 'zod';
import { revalidatePath } from 'next/cache';

type CreatePostState = {
  error?: string;
  fieldErrors?: z.inferFlattenedErrors<typeof createPostSchema>['fieldErrors'];
  success: boolean;
};

export async function createPost(
  _prevState: CreatePostState,
  formData: FormData,
): Promise<CreatePostState> {
  const result = createPostSchema.safeParse({
    title: formData.get('title'),
    content: formData.get('content'),
  });

  if (!result.success) {
    return {
      error: 'Validation failed',
      fieldErrors: result.error.flatten().fieldErrors as any,
      success: false,
    };
  }

  try {
    await db.insert(posts).values(result.data);
    revalidatePath('/posts');
    return { success: true };
  } catch {
    return { error: 'Server error', success: false };
  }
}
```

### Client component consuming Server Action result

```tsx
// app/ui/create-post-form.tsx
'use client';

import { useActionState } from 'react';
import { createPost } from '@/app/actions';

export function CreatePostForm() {
  const [state, formAction, isPending] = useActionState(createPost, { success: false });

  return (
    <form action={formAction}>
      <input name="title" required />
      <textarea name="content" required />
      <button type="submit" disabled={isPending}>
        {isPending ? 'Saving...' : 'Create'}
      </button>
      {state.error && <p className="error">{state.error}</p>}
    </form>
  );
}
```

### env.d.ts for NEXT_PUBLIC typing

```ts
// env.d.ts
declare namespace NodeJS {
  interface ProcessEnv {
    NEXT_PUBLIC_API_URL: string;
    NEXT_PUBLIC_APP_NAME: string;
  }
}
```

### next.config.ts (annotated)

```ts
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  typedRoutes: true,
  reactCompiler: true,
  output: 'standalone',
  images: {
    remotePatterns: [
      { protocol: 'https', hostname: 'cdn.example.com' },
    ],
  },
};

export default nextConfig;
```

## Reference snippets (Option B)

### vite.config.ts with @shared alias

```ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'node:path';

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@shared': path.resolve(__dirname, '../../packages/shared/src'),
    },
  },
  server: {
    proxy: { '/api': 'http://localhost:3000' },
  },
});
```

### Typed component with props interface

```tsx
import type { User } from '@shared/types';

interface UserCardProps {
  user: User;
  onSelect?: (id: string) => void;
}

export function UserCard({ user, onSelect }: UserCardProps): React.ReactElement {
  return (
    <article onClick={() => onSelect?.(user.id)}>
      <h2>{user.name}</h2>
      <p>{user.email}</p>
    </article>
  );
}
```

### useQuery typed against @shared DTO

```tsx
import { useQuery } from '@tanstack/react-query';
import type { Post } from '@shared/types';

export function usePosts() {
  return useQuery<Post[], Error>({
    queryKey: ['posts'],
    queryFn: () => fetch('/api/posts').then((r) => r.json()),
  });
}
```

### React Hook Form with zodResolver and @shared schema

```tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { createPostSchema } from '@shared/schemas';
import type { CreatePostInput } from '@shared/schemas';

export function CreatePostForm() {
  const { register, handleSubmit, formState: { errors } } = useForm<CreatePostInput>({
    resolver: zodResolver(createPostSchema),
  });

  const onSubmit = (data: CreatePostInput) => {
    fetch('/api/posts', { method: 'POST', body: JSON.stringify(data) });
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('title')} />
      {errors.title && <span>{errors.title.message}</span>}
      <button type="submit">Create</button>
    </form>
  );
}
```

## Sources used

- [Next.js 14→15 Upgrade Guide](https://nextjs.org/docs/app/guides/upgrading/version-15) — official
- [Next.js Caching and Revalidating](https://nextjs.org/docs/app/guides/caching-without-cache-components) — official
- [Next.js Mutating Data (Server Actions)](https://nextjs.org/docs/app/getting-started/mutating-data) — official
- [Next.js typedRoutes](https://nextjs.org/docs/app/api-reference/config/next-config-js/typedRoutes) — official
- [Next.js unstable_cache](https://nextjs.org/docs/app/api-reference/functions/unstable_cache) — official
- [React 19 Release Post](https://react.dev/blog/2024/12/05/react-19) — official
- [React 19 Upgrade Guide](https://react.dev/blog/2024/04/25/react-19-upgrade-guide) — official
- [Next.js 16 Release Notes](https://nextjs.org/blog/next-16-1) — official
- [Vercel Changelog — May 2026 Security Release](https://vercel.com/changelog/next-js-may-2026-security-release) — official
