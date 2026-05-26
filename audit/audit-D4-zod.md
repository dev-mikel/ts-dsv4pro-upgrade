# Audit D4 — Zod Runtime Validation

## 1. Knowledge Level: Solid

I have thorough knowledge of Zod's core API: `z.object`, `z.string/number/boolean/date/enum/nativeEnum`, `z.array`, `z.tuple`, `z.union`, `z.discriminatedUnion`, `z.literal`, `z.record`, `z.optional`/`nullable`, `.refine`, `.superRefine`, `.transform`, `.pipe`, `.default`, `.catch`, `.brand`, `z.infer` and `z.input`/`z.output` type inference, `.partial`/`.deepPartial`, `.pick`/`.omit`, `.extend`/`.merge`, `.passthrough`/`.strict`, coercion types, and `preprocess`.

## 2. Versions Covered

Confident through Zod 3.22–3.23 (early–mid 2024). My knowledge ends there. If Zod 4.x shipped (which I recall was in beta/alpha), that represents a significant gap — Zod 4 was expected to be a major rewrite with API changes.

## 3. Identified Gaps

- **Zod 4.x**: This is the largest potential gap. Zod 4 was announced as a major version with a smaller bundle, different internal architecture, and possible API surface changes. I don't know if it shipped, what changed, or whether it's backward compatible.
- **`z.interface`**: I recall Zod 4 intended to introduce a new `z.interface` type for better TS integration — I don't know the final design.
- **Schema namespacing for shared packages**: In a monorepo, schemas from `@shared-types` need clear naming. I don't know the current best practice for Zod schema registration/naming in a multi-app context (e.g., for error messages that reference "which schema failed").
- **Zod + Fastify integration**: I know `fastify-type-provider-zod` exists for Fastify 4.x. I don't know if it's compatible with Fastify 5.x or if the preferred integration changed.
- **Zod + React Hook Form**: I know `@hookform/resolvers` supports Zod. I don't know if any Zod-to-RHF type bridges changed in recent versions.
- **`z.coerce` refinements**: The exact behavior of chaining `.refine` after `.coerce` may have edge cases I'm fuzzy on.
- **Performance**: I know Zod has known performance characteristics vs alternatives (valibot, arktype). I don't know the current benchmarks or if Zod 4 addressed these.

## 4. Hallucination Risk

Highest risk areas:
- Writing Zod 3.x API for a project on Zod 4.x, or vice versa, assuming backward compatibility that doesn't exist.
- Recommending `fastify-type-provider-zod` patterns that don't work with current Fastify versions.
- Inventing Zod 4 features that were proposed but never shipped.

## 5. Confidence: 7/10

If the project is on Zod 3.x, I'm confident. If on Zod 4.x, confidence drops to 3/10. The Zod 4 unknown is the dominant risk factor.

## Intersection Notes

- **D5 (shared types)**: Zod schemas in `@shared-types` are the single source of truth. `z.infer<typeof schema>` is the bridge to TypeScript types. If Zod 4 changed inference semantics, shared types packages are affected everywhere.
- **D8 (Fastify)**: Request/response schemas in Fastify use Zod via a type provider. Version mismatch between Zod, Fastify, and the type provider breaks the entire backend validation layer.
- **D9 (frontend)**: RHF+Zod validation in forms depends on `@hookform/resolvers`. Any Zod API change ripples to all forms.
- **D10 (workers)**: Job payload validation with Zod. Workers receive data serialized from Redis — Zod is the integrity boundary.

## Recommended Phase 2 Search Queries

1. `Zod 4.x release status changelog 2025`
2. `Zod 4 migration guide breaking changes`
3. `fastify-type-provider-zod Fastify 5 compatibility 2025`
4. `Zod React Hook Form resolver latest 2025`
5. `Zod vs Valibot vs ArkType benchmarks 2025`
