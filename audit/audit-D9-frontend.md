# Audit D9 — Frontend App (Next.js / React)

## 1. Knowledge Level: Partial (high velocity, high drift)

I understand the Next.js App Router concepts: layouts, pages, `loading.tsx`, `error.tsx`, `not-found.tsx`, route groups, Server Components (RSC) vs Client Components (`"use client"`), Server Actions (`"use server"`), `revalidatePath`/`revalidateTag`, `cache`/`unstable_cache`, `generateStaticParams`, `generateMetadata`, middleware, `next.config` options. I know React: hooks (`useState`, `useEffect`, `useMemo`, `useCallback`, `useRef`, `useId`, `useTransition`, `useActionState`, `useOptimistic`), React Hook Form with Zod via `@hookform/resolvers`.

However, the Next.js release cadence is extremely aggressive (major versions every ~6 months), and the caching, rendering, and routing behavior has changed significantly across versions.

## 2. Versions Covered

- **Next.js**: Confident through 14.x (ending ~mid 2024). **Next.js 15.x and 16.x are the largest gaps across all D domains.** 15.x changed caching defaults, introduced React 19, `useActionState`, and possibly `after()` / `forbidden`/`unauthorized` APIs. 16.x may have shipped with further changes.
- **React**: Confident through 18.x. **React 19 is a major gap** — it introduces Server Components as stable, new hooks, and changes to Suspense/hydration behavior.
- **React Hook Form**: Confident through v7.x. RHF's integration with Zod (`@hookform/resolvers`) is relatively stable.
- **Vite** (if using React+Vite instead of Next.js): Confident through v5.x.

## 3. Identified Gaps

- **Next.js 15.x caching changes**: The biggest practical gap. Next.js 15 changed default caching behavior — `fetch` requests are no longer cached by default, `GET` route handlers are no longer cached, and client router cache behavior changed. I know the high-level changes but not the exact details or migration path.
- **React 19**: New hooks: `useActionState` (replacing/renaming `useFormState`), `useOptimistic` updates, `use` hook for reading promises/context in render, ref as a prop (no more `forwardRef`), `Context.Provider` simplified. I know *about* these but don't have deep practical knowledge of the new patterns.
- **Next.js 15.x/16.x new APIs**: `after()` for post-response work, `forbidden()`/`unauthorized()` for auth errors. I don't know if these shipped or what the exact API is.
- **Partial Prerendering (PPR)**: I know the concept (static shell + dynamic holes). I don't know if it's stable or still experimental.
- **Server Components to Server Actions data flow**: The exact patterns for form handling, optimistic updates, and `revalidatePath` placement have evolved. I don't know the current "idiomatic" patterns as of 2025.
- **`next.config` vs `next.config.ts`**: I don't know if TypeScript config is supported or what the current config format changes are.
- **Turbopack**: I know it exists as a webpack replacement. I don't know if it's the default in Next.js 15/16, what its limitations are, or how it interacts with Turborepo.
- **`@tanstack/react-query`** (if used): I don't know if the frontend uses a server-state library alongside or instead of RSC data fetching.
- **RHF + Server Actions**: The integration pattern for React Hook Form with Next.js Server Actions (as opposed to REST endpoints) — I don't know the current best practice.

## 4. Hallucination Risk

Highest risk areas:
- **Explaining Next.js caching behavior based on v14 defaults when v15 changed them. This is the most probable hallucination source — the mental model I have is v14-biased.**
- Writing `"use server"` patterns that don't align with v15/v16 RSC conventions.
- Using `forwardRef` when React 19 makes it unnecessary.
- Confusing `useFormState` for `useActionState` (the rename).
- Assuming `revalidatePath` behavior that changed across versions.
- Claiming SSR/SSG/ISR patterns work as they did in Pages Router-era Next.js.

## 5. Confidence: 4/10

This is my lowest confidence domain. The Next.js + React release velocity means even being 12 months behind means my knowledge is significantly stale on defaults, caching, and idiomatic patterns. The concepts are correct but the details are likely wrong.

## Intersection Notes

- **D4 (Zod)**: Form validation with Zod via RHF is the frontend validation story. If the project uses Server Actions, Zod runs on both client (for immediate feedback) and server (for security).
- **D5 (shared types)**: API route response types and form input types should come from `@shared-types`. If the frontend imports directly from a backend package, that's an architectural boundary violation.
- **D8 (Fastify)**: The frontend calls the backend API. API response types shared via `@shared-types` provide end-to-end type safety.
- **D3 (Turborepo)**: Next.js build in a Turborepo pipeline. `next build` output and caching in Turbo 2.x is an intersection I don't fully know.
- **S2 (Docker)**: Next.js Dockerfile needs the standalone output. `output: "standalone"` in `next.config` — I don't know if this changed in 15/16.
- **D7 (linting)**: Next.js has its own ESLint plugin (`@next/eslint-plugin-next`). Flat config support is unknown.

## Recommended Phase 2 Search Queries

1. `Next.js 15 caching changes migration guide defaults 2025`
2. `Next.js 16 release notes new features 2025`
3. `React 19 useActionState use hook ref as prop migration`
4. `Next.js Server Actions React Hook Form Zod pattern 2025`
5. `Next.js App Router partial prerendering status 2025`
6. `Next.js standalone output Docker 15 16 2025`
7. `Turbopack default Next.js 15 16 limitations 2025`
8. `Next.js revalidatePath revalidateTag behavior 15 16`
