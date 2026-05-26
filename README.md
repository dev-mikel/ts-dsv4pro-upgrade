# typescript-knowledge-system

> **Self-audited knowledge augmentation pipeline for DeepSeek V4 Pro on fast-moving TypeScript stacks.**
> The model audits its own knowledge gaps, researches exactly those gaps from official sources, and consumes the result — closing the training-cutoff gap for a specific stack configuration. Orchestrated with Claude Code + OpenRouter.

[![TypeScript](https://img.shields.io/badge/TypeScript-5.5+-3178C6?style=flat&logo=typescript&logoColor=white)](https://typescriptlang.org)
[![DeepSeek V4 Pro](https://img.shields.io/badge/DeepSeek-V4_Pro-4D6BFF?style=flat)](https://deepseek.com)
[![OpenRouter](https://img.shields.io/badge/OpenRouter-Routing-FF6B6B?style=flat)](https://openrouter.ai)
[![Claude Code](https://img.shields.io/badge/Claude_Code-Orchestration-blueviolet?style=flat)](https://claude.ai/code)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

> **Audit date:** 23 May 2026 — skills are calibrated to the knowledge gaps DeepSeek V4 Pro reported about itself on this date.

---

## Table of Contents

- [The Problem](#the-problem)
- [The Solution](#the-solution)
- [Why This Project](#why-this-project)
- [Architecture](#architecture)
- [Stack Coverage](#stack-coverage)
- [Methodology](#methodology)
- [Output Structure](#output-structure)
- [Skill Routing System](#skill-routing-system)
- [Usage](#usage)
- [Token Efficiency](#token-efficiency)
- [Adapting to Your Stack](#adapting-to-your-stack)
- [Tools Used](#tools-used)
- [Version Reference](#version-reference)
- [License](#license)

---

## The Problem

Modern TypeScript stacks move faster than LLM training cycles. By deployment, key parts of a model's knowledge are already stale:

- **Turborepo 2.x** replaced `pipeline` with `tasks` in `turbo.json` — a silent breaking change
- **Next.js 15** inverted the default fetch caching model — a mental-model-level gap, not just syntax
- **ESLint v9** switched to flat config (`eslint.config.js`) — legacy `.eslintrc` silently applies no rules
- **Zod 4.x** introduced breaking changes to the inference API
- **Fastify 5.x**, **Drizzle 0.45.x**, **React 19** — each with their own breaking changes

The failures these gaps produce are not always visible. They generate plausible, compilable code that fails at runtime or silently applies wrong behavior. Standard RAG approaches retrieve documents but don't reason about *what the model already knows* versus *what it needs to update*.

---

## The Solution

A structured pipeline where **the same model that will consume the knowledge first audits its own gaps**.

DeepSeek V4 Pro:

1. **Audits** its own knowledge per domain — identifying what is solid, stale, or high hallucination risk — with no internet access
2. **Researches** only the gaps it flagged, from official sources, in urgency order

Because the auditing model is the consuming model, the resulting skill documents are not generic documentation — they are precisely the knowledge DeepSeek V4 Pro lacked as of **23 May 2026**. They load on demand during coding sessions, replacing stale training knowledge with version-correct content for the specific stack.

---

## Why This Project

**Self-targeted augmentation.** Most knowledge-injection approaches dump generic docs into context. This pipeline fills *exactly* the gaps the model itself reported — no more, no less. The model is both auditor and beneficiary, making the augmentation maximally relevant and minimally wasteful.

**Cost and iteration.** The entire pipeline runs on DeepSeek V4 Pro — strong technical reasoning at a fraction of frontier-model cost. A 12-domain audit → research cycle is cheap enough to re-run whenever a major dependency ships a breaking change. On a frontier model, that economics would discourage iteration.

**Accessible for cost-sensitive contexts.** For developers in regions like LATAM, where frontier-model API costs are prohibitive relative to local budgets, this project is built around that reality. Advanced agentic pipelines should not require a frontier-model budget to be useful.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│              CLAUDE CODE  +  OpenRouter                 │
│        (orchestration → routes to DeepSeek V4 Pro)      │
└───────────────────────┬─────────────────────────────────┘
                        │
          ┌─────────────▼─────────────┐
          │  PHASE 1: SELF-AUDIT      │
          │  DeepSeek V4 Pro          │
          │  Audits own knowledge     │
          │  per domain — no web      │
          └─────────────┬─────────────┘
                        │ gap map + urgency ranking
          ┌─────────────▼─────────────┐
          │  PHASE 2: RESEARCH        │
          │  DeepSeek V4 Pro          │
          │  Official docs only       │
          │  One domain at a time     │
          └─────────────┬─────────────┘
                        │ skill documents
          ┌─────────────▼─────────────┐
          │  SKILL ROUTING            │
          │  SKILL.md index           │
          │  On-demand loading        │
          │  Consumed by DeepSeek V4  │
          └───────────────────────────┘
```

---

## Stack Coverage

12 skill documents covering a **multi-app monorepo with independently deployable units**:

```
/apps/frontend   → Next.js App Router        → own Docker container
/apps/backend    → Fastify                   → own Docker container
/apps/workers    → BullMQ background jobs    → own Docker container
/packages/shared-types                       → internal, not deployed
/packages/db     → Drizzle ORM               → internal, not deployed
```

### Domain Map

| Layer | ID | Domain | Key Technologies |
|---|---|---|---|
| Foundational | D1 | TypeScript Language Core | Generics, mapped types, `satisfies`, inferred predicates |
| Foundational | D2 | Compiler & Tooling | tsconfig, project references, tsup, esbuild, swc |
| Foundational | D3 | Multi-App Monorepo | Turborepo 2.x, pnpm workspaces, turbo prune |
| Cross-cutting | D4 | Runtime Validation | Zod schemas, inference, transforms |
| Cross-cutting | D5 | Shared Types & Contracts | @shared design, DTOs, job payloads, barrel exports |
| Cross-cutting | D6 | Testing | Vitest workspace, Fastify inject, expect-type |
| Cross-cutting | D7 | Linting & Formatting | ESLint v9 flat config, typescript-eslint v8, Prettier |
| App | D8 | Backend (Fastify) | Plugin-per-module, Zod type provider, graceful shutdown |
| App | D9 | Frontend (Next.js / Vite) | App Router, Server Actions, React 19, caching model |
| App | D10 | Workers (BullMQ) | Typed queues, job payloads, drain semantics |
| Support | S1 | Data Layer (Drizzle) | pgTable, InferSelectModel, drizzle-kit migrations |
| Support | S2 | Containerization | Multi-stage Dockerfiles, turbo prune in Docker, Compose |

---

## Methodology

### Phase 1 — Self-Knowledge Audit

DeepSeek V4 Pro audits its own knowledge for each domain before any web access. Pure self-assessment.

Each domain audit produces:

- **Knowledge level**: Solid / Partial / Outdated / Unknown
- **Versions covered**: what the model reliably knows and until when
- **Identified gaps**: specific APIs, flags, or patterns likely to be wrong
- **Hallucination risk**: which question types are most likely to produce plausible-but-wrong answers
- **Confidence score**: 1–10 with justification

The audit also evaluates **intersection points** — where two domains interact (e.g. Zod schemas shared between Fastify and workers) — because compound gaps cause the most silent damage.

Final synthesis produces:
- A ranked list of domains by urgency of external research
- Claims the model must NOT make without web verification
- An overall confidence score for the entire stack

> In the audit run on **23 May 2026**, DeepSeek V4 Pro reported an overall confidence of **4/10** for this stack, flagging Turborepo 2.x and Next.js 15/16 as TIER 1 CRITICAL.

---

### Phase 2 — Targeted Research

Each domain gets a focused research prompt with:

- **Explicit version targets** derived from the audit (e.g. "document Turborepo 2.x — DO NOT document 1.x")
- **Mandatory sources** — official docs, changelogs, GitHub releases
- **Audit-flagged gaps** as explicit tasks to resolve
- **Scope boundaries** — what belongs in this skill vs adjacent domains
- **Propagation locks** — certain skills cannot be written until their dependencies are verified (e.g. D5 cannot be written until D1, D3, and D4 are confirmed accurate)

Research runs **one domain at a time** in priority order, using Claude Code's native web fetch to retrieve content from official docs.

Each skill document includes a **Sources Used** section listing every URL that contributed a fact, labeled as official, expert, or secondary.

---

## Output Structure

```
.
├── CLAUDE.md                          ← Claude Code integration (skill loading policy)
├── SKILL.md                           ← Skill routing index (~400 tokens, read first)
├── typescript-updated/
│   ├── D1-typescript-language-core.md
│   ├── D2-typescript-compiler-tooling.md
│   ├── D3-multi-app-monorepo.md
│   ├── D4-zod.md
│   ├── D5-shared-types-contracts.md
│   ├── D6-testing.md
│   ├── D7-linting-formatting.md
│   ├── D8-backend-app-fastify.md
│   ├── D9-frontend-app.md
│   ├── D10-workers-app.md
│   ├── S1-data-layer-drizzle-orm.md
│   └── S2-containerization.md
└── Audit/                             ← Phase 1 audit files (reference only)
    ├── audit-D1-ts-core.md
    ├── audit-D2-compiler-tooling.md
    └── ...
```

---

## Skill Routing System

Loading all 12 skill documents on every request would be wasteful (~36,000–60,000 tokens). A two-level routing system loads only what is needed.

**`CLAUDE.md`** — read at session start, ~50 tokens about skills:

```markdown
## TypeScript Stack Skills

This project has updated, research-backed skill documents for the full stack.
They take precedence over training knowledge when they conflict.

Before working on any TypeScript, architecture, or infrastructure task:
1. Read `SKILL.md` — lightweight index that tells you which skill files are relevant.
2. Read ONLY the skill file(s) the index points to — do not load all 12.
3. If a task spans multiple domains, read the lowest-layer skill first.

Skill files live in /typescript-updated/.
Do not load skill files proactively on session start.
```

**`SKILL.md`** — read on demand, ~400 tokens, routes to specific files:

```markdown
| D8 | typescript-updated/D8-backend-app-fastify.md | Fastify routes, plugins, Zod type provider... |
| D9 | typescript-updated/D9-frontend-app.md        | Next.js App Router, Server Actions, React 19... |
...
```

Multi-skill rules handle cross-domain tasks:

```
Typed Fastify route using @shared Zod schema  → D4 + D8
Server Action with Zod validation             → D4 + D9
Job payload shared between backend/workers    → D5 + D8 + D10
```

---

## Usage

### Prerequisites

- [Claude Code](https://claude.ai/code) as the orchestration environment
- [OpenRouter](https://openrouter.ai) account with DeepSeek V4 Pro access, routed into Claude Code

### Running the Pipeline

**Step 1 — Self-Audit (no internet required)**

In a DeepSeek V4 Pro session, run the Phase 1 audit prompt. The model audits its own knowledge across all 12 domains and produces a gap map and urgency ranking. Record the audit date.

Save outputs to `Audit/audit-DX-<domain>.md` per domain.

**Step 2 — Research (one domain at a time)**

In a DeepSeek V4 Pro session, run the Phase 2 prompt for the target domain. Start with TIER 1 CRITICAL domains from the urgency ranking.

```bash
# Example: research D3 (Turborepo — TIER 1 CRITICAL)
# In your DeepSeek V4 Pro session, paste the D3 research prompt.
# The model fetches from:
#   turbo.build/docs/reference/configuration
#   github.com/vercel/turbo (CHANGELOG)
#   turbo.build/docs/handbook/deploying-with-docker
```

Save each output to `typescript-updated/DX-<domain>.md`.

**Step 3 — Activate**

Place `CLAUDE.md` and `SKILL.md` at the root of your project. Claude Code (running DeepSeek V4 Pro via OpenRouter) will use them automatically in all subsequent sessions.

---

## Token Efficiency

| Loading strategy | Tokens | When used |
|---|---|---|
| All 12 skills loaded | ~36,000–60,000 | Never (avoided by design) |
| SKILL.md index only | ~400 | Default on any technical task |
| SKILL.md + 1 domain | ~2,500–5,000 | Single-domain task |
| SKILL.md + 2–3 domains | ~5,000–12,000 | Cross-domain task |

The routing system reduces skill-related token cost by ~85–95% compared to loading all documents.

---

## Adapting to Your Stack

The methodology is stack-agnostic and model-agnostic.

1. **Run the self-audit on your consuming model** — the gaps must be its own. Record the audit date.
2. **Redefine the architecture context** — replace the monorepo description with your actual deployment topology.
3. **Rebuild the domain map** — identify your stack's technology layers. Each technology with a breaking-change history deserves its own domain.
4. **Apply the urgency ranking** — the audit surfaces which domains have the highest hallucination risk for your specific versions. Research those first.
5. **Update SKILL.md triggers** — rewrite the "Read when the task involves..." column for your technologies.
6. **Re-run on model updates** — if the consuming model receives an internal update, re-run Phase 1. The new baseline may have different gaps.

---

## Tools Used

| Tool | Role | Phase |
|---|---|---|
| Claude Code | Orchestration environment | All |
| OpenRouter | Model routing to DeepSeek V4 Pro | All |
| DeepSeek V4 Pro | Self-audit, research, generation | All |

**Why DeepSeek V4 Pro for the whole pipeline?**
The model that consumes the skills should be the one that audits its own gaps — otherwise the augmentation targets the wrong knowledge state. Running every phase on a single model keeps the gap map, research, and consumption coherent. DeepSeek V4 Pro makes this economically viable at 12-domain scale.

---

## Version Reference

Skills were generated against these versions (audit date 23 May 2026):

| Technology | Version |
|---|---|
| TypeScript | 5.5+ |
| Turborepo | 2.x |
| pnpm | 9.x |
| Zod | see D4 skill |
| Vitest | 2.x |
| ESLint | v9 flat config |
| typescript-eslint | v8 |
| Fastify | 5.x |
| Next.js | 15/16 |
| React | 19 |
| BullMQ | current stable |
| Drizzle ORM | 0.45.x |
| drizzle-kit | 0.45.x |
| Node.js | LTS |

Regenerate skills when a major version change occurs in a covered domain, or when the consuming model receives an internal update that shifts its baseline knowledge.

---

## Future Work

### Phase 3 — Verification (not yet implemented)

The pipeline design includes a verification phase where DeepSeek V4 Pro reviews each skill document in a **fresh session** across 8 dimensions:

| # | Dimension | What it checks |
|---|---|---|
| 1 | Version integrity | Versions match audit targets; no outdated syntax |
| 2 | Technical accuracy | Every API and pattern matches documented behavior |
| 3 | Hallucination detection | Claims without source citations are flagged |
| 4 | Scope compliance | No content from adjacent domains leaked in |
| 5 | Source integrity | Citations actually support their claims |
| 6 | Code quality | Snippets are syntactically valid and idiomatic |
| 7 | Gap honesty | Audit-flagged gaps are resolved or declared explicitly |
| 8 | Architecture fit | Patterns apply to this specific stack, not generically |

Intended verdicts: **PASS**, **CONDITIONAL** (specific fixes required), or **FAIL** (reject and rewrite). Running the verifier in a fresh session — independent of the writing context — reduces confirmation bias in the review.

This phase is designed but not yet executed for the current skill set. It is the recommended next step before using these skills in production.

---

## Acknowledgements

This project is built on the work of several outstanding tools and platforms:

- **[Claude Code](https://claude.ai/code)** by Anthropic — the orchestration environment that drives the entire pipeline
- **[OpenRouter](https://openrouter.ai)** — model routing layer that makes multi-model workflows practical
- **[DeepSeek V4 Pro](https://deepseek.com)** — the model that audits, researches, and generates its own knowledge-gap documents
- **[Visual Studio Code](https://code.visualstudio.com)** by Microsoft — editor and development environment
- **[Linux Mint](https://linuxmint.com)** — the OS this project was built and tested on
- **[Zsh](https://www.zsh.org)** — shell powering the CLI tooling and automation glue

---

**@dev-mikel**
Electronics Engineer | Full Stack AI Developer

---

## License

[MIT](LICENSE) © 2025
