# Skill: TypeScript — D1: Language Core

## Summary

Covers the TypeScript type system fundamentals: generics, conditional types, mapped types, utility types, template literal types, type narrowing, declaration files, and the `satisfies` operator. Documented up to **TypeScript 6.0** (stable: 6.0.3, May 2026). Knowledge cutoff: 2026-05-25. Audit risk at time of research: **Moderate** (downgraded from original — all 4 flagged gaps resolved).

## Version landscape

| Version | Key type-system additions |

| **TS 5.2** | `using` declarations (Explicit Resource Management), decorator metadata |
| **TS 5.3** | Switch(true) narrowing, `instanceof` via `Symbol.hasInstance`, import attributes |
| **TS 5.4** | `NoInfer<T>` utility, preserved narrowing in closures after last assignment, `Object.groupBy` / `Map.groupBy` |
| **TS 5.5** | **Inferred type predicates**, **`isolatedDeclarations`**, control flow narrowing for constant indexed accesses, JSDoc `@import`, RegExp syntax checking, ES Set methods |
| **TS 5.6** | Iterator Helper methods types, arbitrary module identifiers, `noUncheckedSideEffectImports`, `--noCheck`, region-prioritized diagnostics |
| **TS 5.7** | Variable declarations from unreachable code, `--rewriteRelativeImportExtensions` |
| **TS 5.8** | `--erasableSyntaxOnly`, granular return-expression branch checks, `require()` of ESM in `nodenext`, `--module node18`, preserved computed property names in DTS |
| **TS 5.9** | `import defer`, `--module node20`, `tsc --init` modernization, expandable hovers |
| **TS 6.0** | `--module nodenext` contextual inference improvements, Node.js subpath imports, union type ordering stabilization with `--exactOptionalPropertyTypes`, Temporal API types, `Iterator.from`, `Map.emplace`, `RegExp.escape`, ES5 target deprecated, AMD/UMD/System dropped |

**Minimum baseline for this skill**: TypeScript 5.5+. Features from earlier versions (4.x) are included only when foundational.

## Essential knowledge

### Generics

**Constraints** (`extends`): Restrict a type parameter to a subset of types.

```ts
function loggingIdentity<Type extends { length: number }>(arg: Type): Type {
  console.log(arg.length);
  return arg;
}
```

**Defaults**: Type parameters can have defaults, usable when inference fails.

```ts
interface Container<T = string> { value: T; }
```

**Inference**: TypeScript infers type arguments from function call arguments. Generic classes are only generic over their *instance* side — static members cannot use the class type parameter.

**Variance**: TypeScript uses structural typing with *covariance* by default for most types. Function parameters are *contravariant* under `--strictFunctionTypes`. Return types are *covariant*.

### Conditional types

Form: `T extends U ? X : Y`. Power comes from combining with generics.

**Infer keyword** — extracts types within the true branch:

```ts
type ReturnType<T> = T extends (...args: never[]) => infer R ? R : never;
type ElementType<T> = T extends (infer E)[] ? E : never;
```

For overloaded functions, `infer` operates on the *last* (most permissive) call signature.

**Distributive behavior**: When `T` is a union, `T extends U ? X : Y` distributes over each union member. Wrap with tuples to disable: `[T] extends [U] ? X : Y`.

**Recursive conditional types** (TS 4.1+): can recurse, e.g., deep `Partial`, nested `Awaited`. TypeScript limits recursion depth (typically ~50 levels).

### Mapped types

Transform existing types property by property:

```ts
type OptionsFlags<Type> = { [Property in keyof Type]: boolean };
```

**Modifiers**: `+` (default) and `-` to add/remove `readonly` and `?`:
```ts
type Mutable<T> = { -readonly [P in keyof T]: T[P] };
type Required<T> = { [P in keyof T]-?: T[P] };
```

**Key remapping via `as`** (TS 4.1+): rename keys and filter via `never`:
```ts
type Getters<T> = { [P in keyof T as `get${Capitalize<string & P>}`]: () => T[P] };
type RemoveKind<T> = { [P in keyof T as Exclude<P, "kind">]: T[P] };
```

**Homomorphic vs non-homomorphic**: Mapped types over `keyof T` are homomorphic — they preserve original modifiers (readonly/optional). Types like `Record<K, V>` are non-homomorphic.

### Utility types — built-in set

| Utility | Description | Min TS |
|---------|-------------|---------|
| `Partial<T>` | All properties optional | 2.1 |
| `Required<T>` | All properties required | 2.8 |
| `Readonly<T>` | All properties readonly | 2.1 |
| `Pick<T, K>` | Select subset of keys | 2.1 |
| `Omit<T, K>` | Remove subset of keys | 3.5 |
| `Record<K, V>` | Map keys to value type | 2.1 |
| `Exclude<T, U>` | Remove union members assignable to U | 2.8 |
| `Extract<T, U>` | Keep union members assignable to U | 2.8 |
| `NonNullable<T>` | Exclude `null` and `undefined` | 2.8 |
| `ReturnType<T>` | Extract return type of function | 2.8 |
| `Parameters<T>` | Extract parameter tuple of function | 3.0 |
| `Awaited<T>` | Unwrap Promise chain recursively | 4.5 |
| `NoInfer<T>` | Block inference on T | 5.4 |
| `InstanceType<T>` | Extract instance type of constructor | 2.8 |
| `ThisParameterType<T>` | Extract `this` parameter type | 3.3 |
| `OmitThisParameter<T>` | Remove `this` parameter | 3.3 |
| `ConstructorParameters<T>` | Extract constructor parameter tuple | 3.0 |
| `ThisType<T>` | Contextual `this` marker | 2.1 |

**Intrinsic string types** (compiler built-ins): `Uppercase<T>`, `Lowercase<T>`, `Capitalize<T>`, `Uncapitalize<T>`.

### `NoInfer<T>` (TS 5.4)

Blocks type inference for a type position while still checking assignability:

```ts
function createStreetLight<C extends string>(colors: C[], defaultColor?: NoInfer<C>) {}
createStreetLight(["red", "yellow", "green"], "blue"); // Error: "blue" not in C
```

Without `NoInfer`, TS would infer `C` as `"red" | "yellow" | "green" | "blue"`, silently widening the type.

### `satisfies` operator (TS 4.9)

Validates that an expression matches a type **without changing the inferred type**:

```ts
const palette = {
  red: [255, 0, 0],
  green: "#00ff00",
} satisfies Record<string, string | RGB>;
// palette.green is still typed as "#00ff00", not string | RGB
```

**When to use each:**
- **`satisfies`**: Validate shape while preserving exact inferred types
- **Type annotation (`: Type`)**: Declare an explicit type, widening the value
- **Type assertion (`as Type`)**: Force a type, no validation of the expression

**Antipattern**: Using `as` when `satisfies` would catch property typos. Using `: Type` when you want to preserve literal types.

### `const` assertions (TS 3.4) and `const` type parameters (TS 5.0)

**`as const`**: Marks an expression as deeply readonly with literal types:
```ts
const config = { mode: "auto", ports: [3000, 3001] } as const;
// type: { readonly mode: "auto"; readonly ports: readonly [3000, 3001] }
```

**`const` type parameters** (TS 5.0): Inline `as const` at the call site:
```ts
function get<T, const C extends readonly string[]>(obj: T, keys: C): Pick<T, C[number]> {}
get(person, ["name", "age"]); // C inferred as readonly ["name", "age"]
```

### Type narrowing

**Built-in type guards**: `typeof`, `instanceof`, `in`, equality checks (`===`, `!==`, `!=`, `==`), assignments, truthiness.

**Discriminated unions**: A common property (`kind`, `type`, `status`) with distinct literal values discriminates union members:
```ts
type Shape = { kind: "circle"; radius: number } | { kind: "square"; side: number };
function area(s: Shape) { if (s.kind === "circle") { /* s.radius available */ } }
```

**Type predicates** (explicit): `arg is Type` return type. TS trusts the predicate — it is not verified (same safety as `as`).

**`using` keyword narrowing** (TS 5.2+): Variables declared with `using` are disposed at end of scope in LIFO order. Types: `Disposable` (`Symbol.dispose`) and `AsyncDisposable` (`Symbol.asyncDispose`).

**Control flow analysis**: Tracks reassignments, early returns. From TS 5.5, narrows `obj[key]` when both `obj` and `key` are effectively constant.

### Inferred type predicates (TS 5.5)

TS 5.5 infers type predicate return types for functions meeting these conditions:
1. No explicit return type or type predicate annotation
2. Single `return` statement, no implicit returns
3. Does not mutate its parameter
4. Returns a boolean expression tied to a refinement on the parameter

```ts
// Inferred as: const isNumber: (x: unknown) => x is number
const isNumber = (x: unknown) => typeof x === "number";

// filters now get precise types automatically:
const nums = [1, 2, null, 3].filter(x => x !== null);
// nums: number[] (previously (number | null)[])
```

Truthiness checks (`!!score`) do NOT infer predicates for primitive types (the `0` ambiguity). Explicit type predicates continue to work, but TS does not check whether it "would have" inferred the same predicate.

If the inferred predicate is too narrow, use an explicit type annotation on the variable to override.

### `isolatedDeclarations` (TS 5.5)

**What it enforces**: Every module export must be sufficiently annotated so that declaration files (`.d.ts`) can be generated by tools *without a full type-checker* — per-file, without cross-module inference.

**What it forbids**:
- Unannotated export return types: `export function foo() { return x; }` → error
- Export types that depend on cross-file inference

**What it does NOT require**:
- Annotations on non-exported locals
- Annotations on "trivial" expressions: `export let x = 10;` (literal type), `export function y() { return 20; }` (simple return expression)

**Configuration**: Must be paired with `declaration: true` or `composite: true`.

**Monorepo impact**: Enables parallel DTS generation — shared packages can generate declarations independently without waiting for dependencies. However, it constrains coding style: every exported function needs an explicit return type annotation.

**When to enable**: When you want parallel builds, or when using non-TSC tools for declaration emit. Not needed if you're using standard TSC builds serially.

### `using` / `await using` (TS 5.2)

ECMAScript Explicit Resource Management proposal (Stage 3 at time of writing).

```ts
class TempFile implements Disposable {
  #path: string;
  [Symbol.dispose]() { fs.unlinkSync(this.#path); }
}

function process() {
  using file = new TempFile(".temp");
  // file disposed here on scope exit, return, or throw
}
```

Key behavior:
- Disposal at end of containing scope (or early return/throw), in **first-in-last-out (LIFO)** order
- Exception resilient: if body throws and disposal throws → `SuppressedError` (with `error` and `suppressed` properties)
- `AsyncDisposable` + `await using` for async cleanup flows
- Types: `Disposable`, `AsyncDisposable`, `DisposableStack` (TS 5.2)

### Declaration files (`.d.ts`)

**When to write**: For libraries consumed by TypeScript users. For monorepo packages that export types.

**Module augmentation**: Extend existing types via declaration merging:
```ts
// augment a library's types
declare module "library-x" { interface Options { newProp: boolean; } }
```

**Ambient declarations**: `declare` keyword for types that exist at runtime but are unknown to TS:
```ts
declare const API_KEY: string;
declare function track(event: string): void;
```

**Global augmentation**: Use `declare global {}` within a module to extend the global scope.

### Template literal types (TS 4.1)

Construct string literal types from unions:
```ts
type Lang = "en" | "ja";
type IDs = `page_${Lang}_header`; // "page_en_header" | "page_ja_header"
```

**Inference**: Extract substrings back from template literals using `infer`:
```ts
type GetKey<T extends `${string}Changed`> = T extends `${infer K}Changed` ? K : never;
// GetKey<"nameChanged"> = "name"
```

Unions in interpolated positions produce the **cross product** — grows factorially. Avoid large union combinations; at ~100K members TSC will error.

**Intrinsic manipulation**: `Uppercase<T>`, `Lowercase<T>`, `Capitalize<T>`, `Uncapitalize<T>`.

### Variadic tuple types (TS 4.0)

Rest/spread in tuple types:
```ts
type Concat<T extends unknown[], U extends unknown[]> = [...T, ...U];
type Tail<T extends unknown[]> = T extends [unknown, ...infer Rest] ? Rest : never;
```

### Class field behavior with `strict` mode

Under `--strict`:
- `strictPropertyInitialization`: Fields must be initialized in constructor or have definite assignment assertion (`!`)
- `noImplicitOverride`: Requires `override` keyword for overridden methods
- `strictNullChecks`: Fields with undefined in union must be guarded before use
- TC39 decorators (TS 5.0+) replaced legacy experimental decorators
- `accessor` keyword (TS 4.9+) for auto-accessor fields (de-sugars to get/set + private `#` field)

## Patterns for a multi-app monorepo

### Typing across module boundaries

Shared types in `/packages/shared-types` should export explicit interfaces, not inferred types from implementation code. This prevents accidental coupling to internal implementation details.

```ts
// packages/shared-types/src/api.ts
export interface User { id: string; name: string; }
// DO NOT: export type User = ReturnType<typeof someInternalFunction>;
```

### Declaration files in `/packages`

- Each package should have `declaration: true` and emit `.d.ts` files.
- If using `isolatedDeclarations`, every export must have an explicit return type annotation.
- `composite: true` enables project references and incremental DTS generation across packages.

### Generic patterns for shared types

Standard CRUD response wrapper:
```ts
type ApiResponse<T> = { data: T; error?: never } | { data?: never; error: string };
```

Discriminated unions for cross-package event/action types:
```ts
type AppEvent = { source: "frontend"; type: "click"; target: string }
              | { source: "backend"; type: "log"; message: string };
```

## Antipatterns and common mistakes

1. **Type assertions instead of narrowing**: `(x as Type)` bypasses checks. Prefer type guards, `satisfies`, or `if` narrowing.

2. **`any` instead of `unknown`**: `any` disables all checking; `unknown` forces narrowing before use.

3. **`satisfies` vs annotation confusion**: Using `: Record<string, string>` when you want `satisfies Record<string, string>` — the annotation widens and loses literal types.

4. **`isolatedDeclarations` without explicit return types**: If you enable this flag, every exported function needs an explicit return type. Not doing so produces errors for every unannotated export.

5. **Distributive conditional type surprises**: `T extends U ? X : Y` distributes over union `T`. When this isn't desired, wrap: `[T] extends [U] ? X : Y`.

6. **Template literal type combinatorial explosion**: `A | B` × `C | D | E` × `F | G` = 12 combinations. Deeply nested generics can produce union types exceeding 100K members.

7. **Assuming `implements` changes inference**: `class C implements I` does NOT infer parameter types from `I` — each parameter must be independently annotated.

## Audit-flagged gaps — resolution

### 1. `isolatedDeclarations` — RESOLVED

**Source**: TS 5.5 release notes (official). Confirmed behavior: requires `declaration` or `composite`; errors on unannotated exports; locals and trivial expressions exempt. Not yet supported for computed property declarations in classes/object literals (known limitation documented in release notes). Parallel DTS generation is the primary monorepo benefit.

### 2. Inferred type predicates (TS 5.5) — RESOLVED

**Source**: TS 5.5 release notes (official), authored by Dan Vanderkam who implemented the feature. Confirmed: 4 conditions for inference; truthiness checks don't infer for primitives due to falsy ambiguity (`0`, `""`); explicit type predicates are still unverified (same trust level as `as`). Can break existing code if inferred predicate is too narrow — fix with explicit type annotation.

### 3. `NoInfer<T>` — RESOLVED

**Source**: TS 5.4 release notes (official), TS utility types handbook page. Confirmed: intrinsic utility type introduced in 5.4. Primary use case is preventing unwanted widening of inferred type parameters. Still assignable/checkable — only inference is blocked, not assignability.

### 4. `using` / `await using` — RESOLVED

**Source**: TS 5.2 release notes (official). Confirmed: Stage 3 TC39 proposal, shipped in TS 5.2. Works with `Disposable` and `AsyncDisposable` global types. LIFO disposal order. `SuppressedError` for nested exceptions. Not downleveled — requires runtime support or polyfill. Does NOT require a specific `lib` target beyond standard ES2022+ (uses `Symbol.dispose` which is available as a well-known symbol).

## Reference snippets

### Generics — constrained identity (all versions)

```ts
function identity<T extends { length: number }>(arg: T): T {
  console.log(arg.length);
  return arg;
}
```

### Conditional type — deep readonly (TS 4.1+)

```ts
type DeepReadonly<T> = T extends object
  ? { readonly [P in keyof T]: DeepReadonly<T[P]> }
  : T;
```

### Mapped type — pick by value type (TS 4.1+)

```ts
type PickByType<T, V> = {
  [P in keyof T as T[P] extends V ? P : never]: T[P];
};
```

### satisfies — validate object shape (TS 4.9+)

```ts
const routes = {
  home: "/",
  profile: "/user/:id",
} satisfies Record<string, `/${string}`>;
```

### `const` type parameter (TS 5.0+)

```ts
declare function pick<const T extends readonly string[]>(keys: T): T[number][];
const k = pick(["a", "b"]); // ("a" | "b")[]
```

### Inferred type predicate (TS 5.5+)

```ts
const isDefined = <T>(x: T): x is NonNullable<T> => x != null;
// Or let TS 5.5 infer:
const isDefined = <T,>(x: T) => x != null;
// inferred as: <T>(x: T) => x is NonNullable<T>
```

### `NoInfer` — block inference (TS 5.4+)

```ts
declare function createStreetLight<C extends string>(
  colors: C[],
  defaultColor?: NoInfer<C>,
): void;
```

### `using` — resource management (TS 5.2+)

```ts
class Connection implements Disposable {
  [Symbol.dispose]() { this.close(); }
  private close() { /* ... */ }
}
function query() {
  using conn = new Connection();
  // conn disposed on scope exit
}
```

### Template literal — event type inference (TS 4.1+)

```ts
type EventOf<T> = {
  on<K extends string & keyof T>(event: `${K}Changed`, cb: (val: T[K]) => void): void;
};
```

### Variadic tuple — concat (TS 4.0+)

```ts
type Concat<A extends unknown[], B extends unknown[]> = [...A, ...B];
type Result = Concat<[1, 2], [3, 4]>; // [1, 2, 3, 4]
```

### Discriminated union — exhaustive check (all versions)

```ts
type Shape = { kind: "circle"; radius: number } | { kind: "square"; side: number };
function area(s: Shape): number {
  switch (s.kind) {
    case "circle": return Math.PI * s.radius ** 2;
    case "square": return s.side ** 2;
  }
}
```

### `isolatedDeclarations` — compliant export (TS 5.5+)

```ts
// Must have explicit return type annotation
export function add(a: number, b: number): number {
  return a + b;
}
```

## Sources used

- [TypeScript 5.2 Release Notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-2.html) — official
- [TypeScript 5.3 Release Notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-3.html) — official
- [TypeScript 5.4 Release Notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-4.html) — official
- [TypeScript 5.5 Release Notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-5.html) — official
- [TypeScript 5.6 Release Notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-6.html) — official
- [TypeScript 5.8 Release Notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-8.html) — official
- [TypeScript 5.9 Release Notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-9.html) — official
- [TypeScript 6.0 Release Notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html) — official
- [TypeScript Handbook — Generics](https://www.typescriptlang.org/docs/handbook/2/generics.html) — official
- [TypeScript Handbook — Conditional Types](https://www.typescriptlang.org/docs/handbook/2/conditional-types.html) — official
- [TypeScript Handbook — Mapped Types](https://www.typescriptlang.org/docs/handbook/2/mapped-types.html) — official
- [TypeScript Handbook — Template Literal Types](https://www.typescriptlang.org/docs/handbook/2/template-literal-types.html) — official
- [TypeScript Handbook — Narrowing](https://www.typescriptlang.org/docs/handbook/2/narrowing.html) — official
- [TypeScript Handbook — Utility Types](https://www.typescriptlang.org/docs/handbook/utility-types.html) — official
- [TypeScript Handbook — Classes](https://www.typescriptlang.org/docs/handbook/2/classes.html) — official
- [TypeScript 4.9 Release Notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-9.html) — official
- [npm: typescript version](https://www.npmjs.com/package/typescript) — version verification (6.0.3)
