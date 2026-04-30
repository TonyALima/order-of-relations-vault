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

2026-04-30. **One new open question on the query builder: [[support-and-or-conditions]] (*high / L*).** Earlier today's filing of `support-date-operators` was retracted by the owner — the existing nine-method `FieldConditionBuilder` already covers date columns via the `T[K] = Date` value-parameter pathway, so any breakage there is a bug for code tests, not a wiki issue. Open-question count: 6 → 7. Wiki page count: 57 → 58.

## Open question addition (2026-04-30)

**[[support-and-or-conditions]]** — extend the `where` callback from a flat `(Condition | undefined)[]` AND-list to a real boolean tree (AND / OR / NOT, nested groups). Anchored in [[QueryBuilder]] § Scope: *"No OR / nested boolean trees. `where` is a flat AND list. A real boolean DSL is significant API surface; the source defers it until at least one concrete use case appears."* Three options sketched: `Or`/`And` combinator helpers (TypeORM/Sequelize-flavoured; lightest-touch); recursive object tree (Prisma-flavoured; replaces the proxy); method chaining (TypeORM legacy; conflicts with [[apply-options-accumulation]]). High impact because there is no `qb.raw(...)` escape hatch — without OR, complex predicates can't be expressed at all without leaving the builder.

## Convention reinforced (2026-04-30)

Open questions are for **genuine design decisions**, not for "is feature X working." If functionality is technically supported but might have bugs, that's a code-test concern, not a wiki issue. The retraction of `support-date-operators` is the canonical example: dates already flow through `FieldConditionBuilder<Date>` via existing comparison operators; no design choice is pending.

## Comparison convention (established 2026-04-30, still in scope)

Every page in `wiki/comparisons/` follows this skeleton:

1. Frontmatter `verdict:` — one-line contribution claim.
2. `## What OOR brings that's new` — 3–5 bullet contribution summary, top of page.
3. `## Overview` — what question this comparison answers about OOR's place in the field.
4. `## Comparison` — dimensions table (~10 rows). Equal-value rule applies.
5. `## OOR's contribution, dimension by dimension` — long-form prose, positive framing.
6. `## Why OOR matters in a crowded market` — closing argument.
7. `## Sources`.

Sections explicitly omitted: "When to reach for which", "Where OOR is behind", "Maturity" row, "Where they converge".

**Equal-value rule:** features with a fully-scoped open-question page (axes spelled out, decorator surface defined, change-surface described) are part of OOR's contribution. The new `support-and-or-conditions` does **not yet** clear the equal-value bar for the comparison pages — scoped enough to be tracked, not yet scoped enough to claim. Promote when an ADR closes it.

## Comparison contribution headlines (TCC-defense form)

- **vs TypeORM** — "TypeORM-shaped ergonomics with modern guarantees."
- **vs Drizzle** — "The OO ergonomic profile, modernized."
- **vs Prisma** — "TypeScript itself as the schema."
- **Stage-3 vs legacy** — "Right place, right time on the dialect transition."

## Summary matrix row-patterns ([[orms-summary]])

1. **OOR ✅ + all competitors ✅** — table-stakes (parameterized SQL by default, lazy chainable builder).
2. **OOR ✅ + TypeORM ✅ + Prisma/Drizzle ❌** — the OO ergonomic profile.
3. **OOR ✅ + Prisma/Drizzle ✅ + TypeORM ❌** — modern guarantees TypeORM is locked out of.
4. **OOR ✅ alone** — distinctive contributions (no `unsafe` SQL escape hatch in either user code OR library internals; auto-emitted STI discriminator index; FK demotion at the type level; compile-time rejection of partial entities on `create()`).

## Drift outcomes (still in scope)

- **D1 — `Repository.find()` does not exist.** Only `findOne` / `findMany` / `findById`.
- **D3 — `FindOptions.inheritance` is real.** `InheritanceSearchType` (`ALL` / `ONLY` / `SUBCLASSES`).
- **D4 — `FieldConditionBuilder` operator inventory.** Nine methods (`eq`, `ne`, `gt`, `gte`, `lt`, `lte`, `isNull`, `isNotNull`, `in`), uniform across `V` — date columns are covered through the value-parameter `V = Date` pathway.
- **D5 — implicit `idx_discriminator` index.** Three statements per STI root: `CREATE TABLE`, `ALTER ADD COLUMN discriminator`, `CREATE INDEX idx_discriminator`.

## Key facts (current state)

- **Repository surface = exactly 6 methods.** `findMany`, `findOne`, `findById`, `create`, `delete`, `update`. There is no `find()`.
- **`FindOptions<T>` carries TWO fields**: `where` (replaces conditions) and `inheritance` (pushes a discriminator predicate).
- **`FieldConditionBuilder<V>` exposes 9 methods, uniform across `V`** — operator surface is intentionally type-agnostic; covers dates through the value parameter.
- **`where` returns a flat `(Condition | undefined)[]`** joined by AND in SQL emission — flatness is what [[support-and-or-conditions]] proposes to break.
- **STI schema-create emits THREE statements per root**: `CREATE TABLE`, `ALTER ADD COLUMN discriminator`, `CREATE INDEX idx_discriminator`.
- **`COLUMN_TYPE` is a closed enum (~50 PG types)**; `toForeignKeyType` demotes SERIAL/SMALLSERIAL/BIGSERIAL → INTEGER/SMALLINT/BIGINT for FK columns.

## Open Questions (7 — tracked as issues — see [[issues|Issue tracker]])

- 🔓 [[support-one-to-many]] — *high / L* — implement `@OneToMany` / `@ManyToOne` (the `TO_MANY` enum member is currently dead).
- 🔓 [[support-and-or-conditions]] — *high / L* — boolean tree in `where` (AND / OR / NOT, nested groups). **NEW 2026-04-30.**
- 🔓 [[support-many-to-many]] — *medium / L* — implement `@ManyToMany` with a synthesized join table.
- 🔓 [[decorator-order-independence]] — *medium / S* — order-independent `@Column` / `@Nullable`.
- 🔓 [[support-user-indexes]] — *medium / M* — `@Index` / `@Unique` + `CREATE INDEX` emission. Folds in `idx_discriminator` naming fix.
- 🔓 [[get-one-limit-1]] — *low / S* — `getOne()` slicing vs `LIMIT 1`?
- 🔓 [[apply-options-accumulation]] — *low / S* — `applyOptions()` replace vs accumulate?

## Active threads

- **Comparison pages link to entity pages** `[[Drizzle ORM]]`, `[[Prisma]]`, and `[[TypeORM]]`. Light entity stubs would close those wikilinks; not yet filed.
- **Drift backlog:** `src/core/orm-error/`, `src/decorators/nullable/`, `src/decorators/relation/` still uncovered.
- **`examples/relations/` page is a stub** — files weren't opened in the drift pass; expand once read.
- **Suggested next action:** `lint the wiki` — one new question page added with cross-references on [[QueryBuilder]] and [[Conditions Proxy]]; quick sweep for orphans / dead links / frontmatter gaps.
