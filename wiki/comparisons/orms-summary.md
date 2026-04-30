---
type: comparison
title: "ORMs Summary"
subjects:
  - "[[order-of-relations]]"
  - "TypeORM"
  - "Prisma"
  - "Drizzle ORM"
dimensions:
  - foundation
  - API shape
  - SQL safety
  - modeling
  - toolchain
verdict: "At-a-glance feature matrix of OOR's contributions against the three dominant TypeScript ORMs. The pattern: OOR matches TypeORM on the OO ergonomic profile (Repository, lazy builder, native STI) while matching Drizzle/Prisma on modern guarantees (no polyfill, strict types) — and adds the contributions neither side delivers (no `unsafe` SQL escape hatch, predicate proxy without imported helpers, PG-specific type rules at the type level)."
created: 2026-04-30
updated: 2026-04-30
tags:
  - comparison
  - summary
  - matrix
status: developing
related:
  - "[[oor-vs-typeorm]]"
  - "[[oor-vs-prisma]]"
  - "[[oor-vs-drizzle]]"
  - "[[stage-3-vs-legacy-decorators]]"
  - "[[0001-stage-3-decorators]]"
  - "[[0002-repository-with-lazy-query-builder]]"
  - "[[0004-parameterized-sql-only]]"
  - "[[0005-no-any-type-driven-api]]"
sources: []
---

# ORMs Summary

A single feature matrix of OOR's contributions against the three dominant TypeScript ORMs. For the long-form thesis behind each row, see [[oor-vs-typeorm]], [[oor-vs-drizzle]], [[oor-vs-prisma]], and [[stage-3-vs-legacy-decorators]].

## Legend

| Symbol | Meaning |
|---|---|
| ✅ | Delivered |
| ⏳ | Planned and well-scoped (open-question page with axes spelled out, decorator surface defined, change-surface described). Treated as equal-value per the [[_index\|comparisons convention]]. |
| 🟡 | Partial — covered for some cases but not as a deliberate first-class property |
| ❌ | Not delivered |
| — | Not applicable (the competitor uses a fundamentally different mechanism, so the question doesn't translate) |

## The matrix

| Feature                                                                              | OOR | TypeORM | Prisma | Drizzle |
| ------------------------------------------------------------------------------------ | :-: | :-----: | :----: | :-----: |
| **Foundation**                                                                       |     |         |        |         |
| Modern decorator dialect (ECMAScript Stage-3)                                        |  ✅  |    ❌    |   —    |    —    |
| No `reflect-metadata` polyfill required at runtime                                   |  ✅  |    ❌    |   —    |    —    |
| TypeScript-only source of truth (no separate schema DSL)                             |  ✅  |    ✅    |   ❌    |    ✅    |
| No codegen step in the development loop                                              |  ✅  |    ✅    |   ❌    |    ✅    |
| Class-as-schema (entity is the type and can carry methods)                           |  ✅  |    ✅    |   ❌    |    ❌    |
| Library-owned metadata (no global `Reflect` registry)                                |  ✅  |    ❌    |   ✅    |    ✅    |
| **API shape**                                                                        |     |         |        |         |
| `Repository<T>` per entity as the single entry-point                                 |  ✅  |   🟡    |   ❌    |    ❌    |
| Lazy chainable `QueryBuilder<T>`                                                     |  ✅  |    ✅    |   ❌    |    ✅    |
| Composable half-built queries (typed value, refinable across functions)              |  ✅  |    ✅    |   ❌    |    ✅    |
| One mental model — no parallel simple-vs-composed paths                              |  ✅  |    ❌    |   ✅    |    ❌    |
| `where()` without SQL strings                                                        |  ✅  |    ❌    |   ✅    |    ✅    |
| Predicate API as a column-namespaced proxy (no imported helpers)                     |  ✅  |    ❌    |   🟡   |    ❌    |
| Per-column operator constraint (rhs typed against the column type)                   |  ✅  |    ❌    |   ✅    |    ✅    |
| **SQL safety**                                                                       |     |         |        |         |
| Parameterized SQL by default                                                         |  ✅  |    ✅    |   ✅    |    ✅    |
| **No** `unsafe` SQL escape hatch in the public API                                   |  ✅  |    ❌    |   ❌    |    ❌    |
| **No** raw SQL strings in library internals either                                   |  ✅  |    ❌    |   ❌    |    ❌    |
| Strict no-`any` policy enforced via lint                                             |  ✅  |    ❌    |   🟡   |   🟡    |
| Compile-time rejection of partial entities on `create()`                             |  ✅  |    ❌    |   ✅    |    ✅    |
| **Modeling**                                                                         |     |         |        |         |
| Native Single-Table Inheritance with discriminator column                            |  ✅  |    ✅    |   ❌    |    ❌    |
| Auto-emitted index on the STI discriminator column                                   |  ✅  |    ❌    |   ❌    |    ❌    |
| `@Index` / `@Unique` user-declared indexes                                           |  ⏳  |    ✅    |   ✅    |    ✅    |
| Decorator-order independence (`@Column` / `@Nullable`)                               |  ⏳  |    —    |   —    |    —    |
| `@OneToMany` / `@ManyToMany` relations                                               |  ⏳  |    ✅    |   ✅    |    ✅    |
| `@ToOne` thunk pattern for circular entity graphs                                    |  ✅  |    ✅    |   —    |    —    |
| FK column type derived from referenced PK (SERIAL → INTEGER demotion)                |  ✅  |    ❌    |   ❌    |   🟡    |
| PostgreSQL-specific affordances as first-class types                                 |  ✅  |    ❌    |   ❌    |   🟡    |
| **Toolchain**                                                                        |     |         |        |         |
| Native Bun support (transpiles Stage-3 decorators, no `tsconfig` flags)              |  ✅  |    ❌    |   🟡   |    ✅    |
| Stage-3-default `tsconfig` (no `experimentalDecorators`, no `emitDecoratorMetadata`) |  ✅  |    ❌    |   —    |    —    |
| Co-located TDD with `bun test`                                                       |  ✅  |   🟡    |   🟡   |   🟡    |

## Notes on selected cells

Some cells need clarification — the matrix is compact by design, but a few rows hide a meaningful nuance.

### `Repository<T>` per entity (TypeORM 🟡)

TypeORM has a `Repository<T>` class, so the literal feature is present. The 🟡 reflects that it is *not* the single entry-point — `DataSource`, `EntityManager`, and `Repository<T>` coexist, and `Repository.find()` (executes immediately) is a different shape than `Repository.createQueryBuilder()` (lazy). The "single entry-point per entity with a one-directional simple-vs-composed boundary" property that ADR 0002 commits to is OOR-specific.

### One mental model (Prisma ✅)

Prisma scores ✅ here because there *is* one shape: `prisma.<modelName>.<verb>({ ... })`. There are no parallel simple-vs-composed paths because there is no composed path at all (queries are object literals, not chainable builders). It's a different way to satisfy the same property, not the same way.

### Predicate API as a column-namespaced proxy (Prisma 🟡)

Prisma's `where: { email: { contains: "..." } }` reads like a column-namespaced predicate object literal — the operator (`contains`) lives under the column key (`email`). It's structurally similar to OOR's `cb.email.eq(value)` proxy but lacks the `cb.<tab>` autocomplete affordance and the chainable `.and()` / `.or()` composition.

### Per-column operator constraint (TypeORM ❌)

TypeORM's `where("user.email = :email", { email })` is a SQL fragment with named parameters. The right-hand side of the equality is bound at runtime, not constrained at compile time against the column's type. Per-column constraint requires the predicate to be a typed expression, not a string.

### Strict no-`any` policy (Prisma / Drizzle 🟡)

Both ship typed public APIs in practice — generated client (Prisma) or inferred types (Drizzle). Neither documents a project-wide `@typescript-eslint/no-explicit-any` stance the way [[0005-no-any-type-driven-api]] does for OOR. The 🟡 reflects "strict in practice, not stated as a hard guarantee."

### FK demotion (Drizzle 🟡)

Drizzle's column types include `serial()` and `integer()` as first-class, and the user can write a manual FK reference using the right type. The 🟡 reflects that the demotion is *manual* — the user picks the FK column type. OOR derives it from the referenced PK at the type level ([[modules/sql-types]]), so a refactor of the referenced PK propagates to FK column types automatically.

### PostgreSQL affordances as first-class types (Drizzle 🟡)

Drizzle's `pgTable` exposes PG-specific column types and index features richly. The 🟡 reflects that the multi-dialect architecture means schemas are dialect-specific (`pgTable` ≠ `mysqlTable`); PG affordances are first-class within the PG schema, but the broader type system has to model the cross-dialect surface.

### Bun support (Prisma 🟡)

Community usage works. No authoritative "Bun is supported" statement in current Prisma docs. The 🟡 reflects "works in practice, not a documented target."

### Co-located TDD with `bun test` (TypeORM / Prisma / Drizzle 🟡)

All three are testable, and `bun test` will run their suites if they're in a Bun project. The 🟡 reflects that the *project's own* test suite isn't built around the fast `bun test` inner loop — they target other test runners (Jest, Vitest, Mocha) by default. OOR uses `bun test` as the canonical TDD loop ([[0006-tdd-rhythm]], [[0007-bun-toolchain]]).

## How to read this matrix

A row is **OOR's contribution** when the OOR cell is `✅` (or `⏳`) and at least one competitor cell is not. A few patterns:

- **Rows where OOR ✅ and all competitors ✅** (parameterized SQL by default, lazy chainable builder, etc.) — table-stakes the field has converged on. OOR doesn't lose anything by being on the modern side of these.
- **Rows where OOR ✅ and TypeORM ✅ but Prisma/Drizzle ❌** (Repository entry-point, native STI, class-as-schema) — the OO ergonomic profile OOR keeps and modernizes. ([[oor-vs-drizzle]], [[oor-vs-prisma]])
- **Rows where OOR ✅ and TypeORM ❌ but Prisma/Drizzle ✅** (no `reflect-metadata`, library-owned metadata, strict no-`any`) — the modern-guarantee profile OOR adopts that TypeORM is locked out of. ([[oor-vs-typeorm]], [[stage-3-vs-legacy-decorators]])
- **Rows where OOR ✅ and all three competitors ❌** — OOR's distinctive contributions:
  - No `unsafe` SQL escape hatch in the public API
  - No raw SQL strings in library internals either
  - Auto-emitted index on the STI discriminator column
  - FK demotion at the type level (SERIAL → INTEGER)
  - Compile-time rejection of partial entities on `create()` (against TypeORM)

That last bucket is the answer to "what does OOR bring that the field didn't already have?" The other buckets are the answer to "what does OOR keep that the field's been splitting in two?"

## Related

- [[oor-vs-typeorm]] — long-form thesis behind every row where TypeORM diverges
- [[oor-vs-prisma]] — long-form thesis behind every row where Prisma diverges
- [[oor-vs-drizzle]] — long-form thesis behind every row where Drizzle diverges
- [[stage-3-vs-legacy-decorators]] — long-form thesis behind the foundation rows on decorators / polyfill / metadata

## Sources

- TypeORM docs: <https://typeorm.io>
- Prisma docs: <https://www.prisma.io/docs>
- Drizzle docs: <https://orm.drizzle.team>
- TC39 proposal-decorators: <https://github.com/tc39/proposal-decorators>
- OOR ADRs: [[0001-stage-3-decorators]], [[0002-repository-with-lazy-query-builder]], [[0004-parameterized-sql-only]], [[0005-no-any-type-driven-api]], [[0006-tdd-rhythm]], [[0007-bun-toolchain]]
