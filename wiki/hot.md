---
type: meta
title: "Hot Cache"
updated: 2026-04-30T00:00:00
created: 2026-04-29
tags:
  - meta
  - cache
status: evergreen
related: []
sources: []
---

# Recent Context

## Last Updated

2026-04-30. **Comparison pages reframed as TCC defense material + new at-a-glance summary matrix.** All four head-to-head pages ([[oor-vs-typeorm]], [[oor-vs-drizzle]], [[oor-vs-prisma]], [[stage-3-vs-legacy-decorators]]) rewritten end-to-end with the contribution frame. New [[orms-summary]] page provides the single-table scoreboard across all three competitors, ~30 rows grouped under Foundation / API shape / SQL safety / Modeling / Toolchain. Cell vocabulary: ✅ ⏳ 🟡 ❌ — (the `—` marks "different mechanism, question doesn't translate"). Page count: 57 wiki pages.

## Comparison convention (new — established 2026-04-30)

Every page in `wiki/comparisons/` follows this skeleton:

1. Frontmatter `verdict:` — one-line contribution claim (not a hedged consumer-decision conclusion).
2. `## What OOR brings that's new` — 3–5 bullet contribution summary, top of page.
3. `## Overview` — what question this comparison answers about OOR's place in the field.
4. `## Comparison` — dimensions table (~10 rows). Equal-value rule applies: planned-and-well-scoped features are listed as OOR contributions.
5. `## OOR's contribution, dimension by dimension` — long-form prose, positive framing.
6. `## Why OOR matters in a crowded market` — closing argument.
7. `## Sources`.

**Sections explicitly omitted:**

- "When to reach for which" — concedes use-cases.
- "Where OOR is behind (by feature, not by design)" — gap callouts work against contribution claim.
- "Maturity" row in dimensions tables — TCC vs established project is meaningless on this axis.
- "Where they converge" — neutral observation that doesn't sell.

**Equal-value rule:** features with a fully-scoped open-question page (axes spelled out, decorator surface defined, change-surface described) are part of OOR's contribution. Seed-only concepts are not claimed. Promoted to feature-equal so far: `@Index` / `@Unique` ([[support-user-indexes]]), decorator-order independence ([[decorator-order-independence]]). Not claimed: [[Schema Migrations]] (still seed).

## Comparison contribution headlines (TCC-defense form)

- **vs TypeORM** — "TypeORM-shaped ergonomics with modern guarantees." Lifts the constraints (legacy decorators, `reflect-metadata`, raw SQL fragments, `any` at parameter seam) that TypeORM accumulated as the early default.
- **vs Drizzle** — "The OO ergonomic profile, modernized." Drizzle bends types to fit SQL; OOR keeps class-as-schema, single-entry-point-per-entity, native STI — the path Drizzle deliberately walked away from.
- **vs Prisma** — "TypeScript itself as the schema." Explores the path Prisma's design closed off: schema-as-code with a fluent composable builder over class-declared entities. Prisma 7's Rust-free architecture retired the runtime objections, sharpening the comparison to schema-DSL-vs-class.
- **Stage-3 vs legacy** — "Right place, right time on the dialect transition." The major libraries (TypeORM, NestJS, Inversify) are stuck on legacy by architectural lock-in (`design:paramtypes` for DI auto-wiring). A clean-room library has zero migration cost; the bet is structurally favorable.

## Summary matrix row-patterns ([[orms-summary]])

The "How to read this matrix" section names four row-patterns worth memorizing:

1. **OOR ✅ + all competitors ✅** — table-stakes (parameterized SQL by default, lazy chainable builder).
2. **OOR ✅ + TypeORM ✅ + Prisma/Drizzle ❌** — the OO ergonomic profile: Repository entry-point, native STI, class-as-schema.
3. **OOR ✅ + Prisma/Drizzle ✅ + TypeORM ❌** — modern guarantees TypeORM is locked out of: no `reflect-metadata`, library-owned metadata, strict no-`any`.
4. **OOR ✅ alone** — distinctive contributions: no `unsafe` SQL escape hatch in either user code OR library internals; auto-emitted STI discriminator index; FK demotion at the type level (SERIAL → INTEGER); compile-time rejection of partial entities on `create()`.

Bucket #4 is the answer to "what does OOR bring that the field didn't already have?" Buckets #2 and #3 are the answer to "what does OOR keep that the field's been splitting in two?"

## Drift outcomes (still in scope)

- **D1 — `Repository.find()` does not exist.** Removed `find()` from [[brief]], [[Repository]], [[Repository Pattern]], [[Lazy Query Builder]]. New framing: `findOne` / `findMany` *are* the composition entry points.
- **D3 — `FindOptions.inheritance` is real.** `InheritanceSearchType` (`ALL` / `ONLY` / `SUBCLASSES`) documented in [[QueryBuilder]], [[Single-Table Inheritance]], [[query-lifecycle]], [[brief]].
- **D4 — `FieldConditionBuilder` operator inventory.** Fixed [[Conditions Proxy]]: nine methods (`eq`, `ne`, `gt`, `gte`, `lt`, `lte`, `isNull`, `isNotNull`, `in`).
- **D5 — implicit `idx_discriminator` index.** Documented in [[MetadataStorage]], [[schema-create-drop]], [[Single-Table Inheritance]], [[Database]]. Latent collision risk now folded into [[support-user-indexes]] naming policy.

## Key facts (current state)

- **Repository surface = exactly 6 methods.** `findMany`, `findOne`, `findById`, `create`, `delete`, `update`. There is no `find()`.
- **`FindOptions<T>` carries TWO fields**: `where` (replaces conditions) and `inheritance` (pushes a discriminator predicate).
- **`FieldConditionBuilder<V>` exposes 9 methods, uniform across `V`**.
- **STI schema-create emits THREE statements per root**: `CREATE TABLE`, `ALTER ADD COLUMN discriminator`, `CREATE INDEX idx_discriminator`.
- **`COLUMN_TYPE` is a closed enum (~50 PG types)**; `toForeignKeyType` demotes SERIAL/SMALLSERIAL/BIGSERIAL → INTEGER/SMALLINT/BIGINT for FK columns.

## Open Questions (tracked as issues — see [[issues|Issue tracker]])

- 🔓 [[support-one-to-many]] — *high / L* — implement `@OneToMany` / `@ManyToOne` (the `TO_MANY` enum member is currently dead).
- 🔓 [[support-many-to-many]] — *medium / L* — implement `@ManyToMany` with a synthesized join table.
- 🔓 [[decorator-order-independence]] — *medium / S* — order-independent `@Column` / `@Nullable`.
- 🔓 [[support-user-indexes]] — *medium / M* — `@Index` / `@Unique` + `CREATE INDEX` emission. Folds in `idx_discriminator` naming fix.
- 🔓 [[get-one-limit-1]] — *low / S* — `getOne()` slicing vs `LIMIT 1`?
- 🔓 [[apply-options-accumulation]] — *low / S* — `applyOptions()` replace vs accumulate?

## Active threads

- **Comparison pages link to entity pages** `[[Drizzle ORM]]`, `[[Prisma]]`, and `[[TypeORM]]` (the latter as a plain string after the earlier fix). Light entity stubs would close those wikilinks; not yet filed.
- **Drift backlog:** `src/core/orm-error/`, `src/decorators/nullable/`, `src/decorators/relation/` still uncovered. File when leverage warrants.
- **`examples/relations/` page is a stub** — files weren't opened in the drift pass; expand once read.
- **Suggested next action:** `lint the wiki` — comparison rewrites changed many wikilinks and added new ones; sweep for orphans / dead links / frontmatter gaps.
