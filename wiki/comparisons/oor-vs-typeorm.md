---
type: comparison
title: "OOR vs TypeORM"
subjects:
  - "[[order-of-relations]]"
  - "TypeORM"
dimensions:
  - decorator dialect
  - entry-point API
  - query builder
  - SQL safety
  - type strictness
  - indexes
  - inheritance
  - database scope
  - toolchain
verdict: "OOR keeps everything that made TypeORM the default decorator-based ORM and tightens the contract on the four axes where TypeORM has been showing its age: decorator dialect, SQL safety, type strictness, and the simple-vs-composed boundary. The contribution is TypeORM-shaped ergonomics with modern guarantees."
created: 2026-04-29
updated: 2026-04-30
tags:
  - comparison
  - typeorm
  - decorators
  - ddd
status: developing
related:
  - "[[0001-stage-3-decorators]]"
  - "[[0002-repository-with-lazy-query-builder]]"
  - "[[0004-parameterized-sql-only]]"
  - "[[0005-no-any-type-driven-api]]"
  - "[[ECMAScript Stage-3 Decorators]]"
  - "[[Repository Pattern]]"
  - "[[Lazy Query Builder]]"
  - "[[Parameterized SQL]]"
  - "[[Single-Table Inheritance]]"
  - "[[support-user-indexes]]"
  - "[[decorator-order-independence]]"
sources: []
---

# OOR vs TypeORM

## What OOR brings that's new

- **A modern decorator dialect at the foundation.** Stage-3 decorators with no `reflect-metadata` polyfill, no `experimentalDecorators` flag — the dialect TypeScript treats as the default, on the standardization track. ([[0001-stage-3-decorators]])
- **SQL injection as a correctness property, not a discipline.** No `sql.unsafe`, no raw SQL fragments anywhere — including inside the library itself. The whole class of injection bugs that ship through TypeORM's `WHERE` strings is removed by construction. ([[0004-parameterized-sql-only]])
- **A typed predicate API instead of SQL strings.** `where(cb => cb.email.eq(value))` replaces TypeORM's `.where("user.email = :email", { email })`. Column types constrain the right-hand side at compile time; the user never authors a SQL fragment. ([[Conditions Proxy]])
- **Strict no-`any` from the public surface inward.** Compile-time rejection of partial entities and bad column references, not runtime `NOT NULL` violations. ([[0005-no-any-type-driven-api]])
- **A single-direction simple-vs-composed boundary.** Simple ops on `Repository<T>`; composition on the lazy `QueryBuilder<T>`. No parallel paths, no "did I call `find()` or `createQueryBuilder()`?" decision per query. ([[0002-repository-with-lazy-query-builder]])

## Overview

TypeORM is the obvious incumbent OOR is most often shaped *against*: both are decorator-driven, both expose `Repository<T>`, both ship a lazy query builder, both target relational databases. The genealogy is real. OOR's argument isn't that TypeORM is wrong — it's that TypeORM's design choices were made when the language and runtime context was different, and several of those choices look load-bearing in retrospect that don't have to be.

> [!key-insight] OOR is TypeORM, modernized
> Read this comparison as a thesis: the ORM most TypeScript developers reach for first carries six years of choices made under constraints (`reflect-metadata`, legacy decorators, raw SQL fragments) that no longer apply. OOR is a clean-room implementation of the same shape with those constraints lifted.

## Comparison

> [!note] Equal-value rule for planned features
> Planned features with a fully-scoped open-question page (axes spelled out, decorator surface defined, change-surface described) are listed below as part of OOR's contribution. The TCC's contribution is the design; the implementation is incremental work downstream of it.

| Dimension | OOR | TypeORM (v0.3.28) |
|-----------|-----|-------------------|
| **Decorator dialect** | ECMAScript **Stage-3** decorators only. No `reflect-metadata`, no `experimentalDecorators`. Default in `tsc --init` since TS 5.0. ([[0001-stage-3-decorators]]) | Legacy TypeScript decorators. Requires `experimentalDecorators: true`, `emitDecoratorMetadata: true`, and `import "reflect-metadata"` at app entry. |
| **Metadata storage** | Library-owned: `MetadataStorage: Map<Constructor, EntityMetadata>` per `Database` instance. Three symbols (`COLUMNS_KEY`, `NULLABLE_KEY`, `RELATIONS_KEY`) joined by `@Entity`. Storage shape can evolve without colliding with anything else in the host app. ([[MetadataStorage]]) | Global `MetadataArgsStorage` populated via `Reflect.metadata`. Shares the global registry with whatever else the host app uses `Reflect.metadata` for. |
| **Entry-point API** | `Repository<T>` is the **single** entry point per entity. Simple ops execute SQL; `Repository.find()` returns a `QueryBuilder<T>` (lazy). One mental model: simple → `Repository`, composed → `QueryBuilder`. ([[0002-repository-with-lazy-query-builder]]) | Three coexisting surfaces: `DataSource`, `EntityManager`, `Repository<T>`. `Repository.find()` executes immediately; composition requires `Repository.createQueryBuilder()` — a separate path. |
| **Query builder shape** | Mutable, single-owner. Typed [[Conditions Proxy]] in `where(cb => cb.field.eq(value))` — no SQL strings, no operator helpers. Terminal: `getMany()` / `getOne()`. ([[Lazy Query Builder]]) | Mutable `SelectQueryBuilder`. Where-clauses are raw SQL fragments with named parameters: `.where("user.firstName = :firstName", { firstName })`. Nested predicates via `Brackets` callback class. |
| **SQL safety** | `sql.unsafe` banned codebase-wide. No raw interpolation anywhere — neither in user code nor library internals. ([[0004-parameterized-sql-only]]) | Where-clauses are SQL fragments routinely; `repository.query(sql, params)` is a first-class raw API. Injection safety relies on the user choosing the named-parameter form. |
| **Type strictness** | Strict no-`any` enforced via `@typescript-eslint/no-explicit-any`. `create(entity: T)` rejects partial entities at compile time. ([[0005-no-any-type-driven-api]]) | Public methods are generic, but parameter binding goes through `ObjectLiteral = { [key: string]: any }`. No project-wide no-`any` stance. |
| **Indexes** | Property-level + class-level `@Index` and `@Unique`, with auto-generated names and a uniform naming policy that also covers the STI discriminator index. Scoped in [[support-user-indexes]]. | `@Index` is both property-level and class-level. Composite, named, `unique: true`, partial via `where`. `@Unique` exists as a separate class-level decorator. |
| **Decorator order** | `@Column` / `@Nullable` order-independent — both decorators write their bucket; `@Entity` joins them. Scoped in [[decorator-order-independence]]. | Order-dependent in the legacy dialect; `Reflect.metadata` papers over some of the timing issues. |
| **Inheritance** | Single-Table Inheritance via `@Entity({ inheritance: { type: "STI", discriminatorValue: "..." } })`. Lazy resolution; auto-emits the discriminator index. ([[Single-Table Inheritance]]) | STI via `@TableInheritance({ column: { ... } })` + `@ChildEntity()`. Concrete-table inheritance also supported via abstract base class. |
| **Relations** | `@ToOne(() => User)` thunk pattern for circular graphs. FK column type derived from referenced PK with **SERIAL → INTEGER demotion** at the type level. ([[Relation Target Thunk]], [[modules/sql-types]]) | `@ManyToOne`, `@OneToMany`, `@ManyToMany`, `@OneToOne` — full relational matrix; lazy/eager loading via `relations` option. |
| **Database scope** | PostgreSQL, deeply. Closed [[modules/sql-types\|`COLUMN_TYPE` enum]] of ~50 PG types; PG-specific type rules (FK demotion) are part of the contract, not adapter glue. | Adapter-shaped abstraction over 11 databases. Type rules are flattened to the lowest common denominator. |
| **Toolchain** | Bun-only. Native Stage-3 support, no `tsconfig` decorator flags, `bun test` for TDD. ([[0007-bun-toolchain]]) | Node.js primary; Bun is not mentioned in docs or getting-started. |

## OOR's contribution, dimension by dimension

### Decorator dialect

TypeORM ships under three transitive constraints: the `experimentalDecorators` flag, the `emitDecoratorMetadata` flag, and a runtime `reflect-metadata` polyfill that has to load before any decorated module evaluates. Each was reasonable in 2018; in 2026 each has costs the original authors couldn't avoid:

- The flags pin a `tsconfig` shape that is no longer the default and is moving toward "transitional" status.
- The polyfill patches a global. In a process with multiple decorator-using libraries (TypeORM + NestJS DI + Inversify is a common stack), all of them write to the same `Reflect` registry.
- The dialect is on the deprecation track relative to TC39's standardization pipeline. Whatever runtimes ship natively in the next decade, it won't be this one.

OOR's Stage-3 stance lifts all three constraints. The cost is one decision: column types are explicit (`@Column({ type: COLUMN_TYPE.TEXT })`) rather than inferred from `design:type`. The [[modules/sql-types|`COLUMN_TYPE`]] enum is closed and small enough that this isn't friction in practice — and it doubles as documentation at the call site.

See [[stage-3-vs-legacy-decorators]] for the dialect-level deep-dive.

### SQL safety as a correctness property

This is OOR's strongest contribution. TypeORM's primary composition path *is* SQL strings — even simple `.where("user.id = :id", { id })` is a fragment. Safety relies on the user choosing the named-parameter form rather than string-concatenation. The library also ships `repository.query()` as a first-class raw API.

OOR forbids the unsafe path entirely. The query builder uses a typed [[Conditions Proxy]] — `cb.email.eq(value)` instead of `"email = :email"` — so the user never authors a SQL fragment in the first place. Dynamic identifiers (table/column names) go through allowlists; `sql.unsafe` is banned codebase-wide, including in library internals. The class of injection bugs that ship through ORM-using applications can't ship through OOR by construction.

For an academic ORM, this is the contribution that survives the longest. Performance numbers and decorator ergonomics are version-bound; "you cannot write a SQL injection through this library" is a property of the design.

### The Repository / QueryBuilder boundary

TypeORM offers two parallel paths: `Repository.find(options)` (executes immediately, simple cases) and `Repository.createQueryBuilder()` (lazy, composed cases). A consumer has to know which path they're on for any given task, and the two have overlapping but non-identical capabilities.

OOR collapses the two: `Repository.find()` returns a builder. Simple methods (`findOne`, `findMany`, `findById`, `create`, `update`, `delete`) are on `Repository`; composition is on `QueryBuilder`. The boundary is one-directional with no overlap. A half-built `QueryBuilder<T>` is a typed value you can pass around, layer additional `where()` calls onto, and run when ready.

This is the design ADR 0002 commits to and what the [[Lazy Query Builder]] page documents in detail.

### Strict no-`any` from the public surface inward

`ObjectLiteral = { [key: string]: any }` is in TypeORM's parameter-binding seam by design — the library accepts arbitrary parameter objects so that user-authored SQL fragments can reference them by name. That's coherent, but it means `any` is in the type graph at a load-bearing point.

OOR's strict `@typescript-eslint/no-explicit-any` policy pushes errors to the IDE. `create(entity: T)` rejects partial entities at compile time, not at runtime via a `NOT NULL` violation. Renaming a column on the entity class breaks every miswritten call site.

### PostgreSQL, deeply

TypeORM's headline is "supports more databases than any other JS/TS ORM." That's a real value proposition for teams with heterogeneous stacks, and a real cost for teams that don't have one — every column-type rule has to flatten to the lowest common denominator. PG-specific affordances (`SERIAL`, `JSONB`, `TIMESTAMPTZ`, partial indexes, expression indexes) become escape hatches rather than first-class types.

OOR's [[modules/sql-types|`COLUMN_TYPE`]] enum is closed, PG-shaped, and ~50 entries. The FK demotion rule (`SERIAL → INTEGER` for a column referencing a SERIAL PK) is encoded at the type level. PG-specific affordances become first-class instead of escape hatches.

This is an opinionated trade — the universe of teams that benefit narrows — but it's the trade that lets the library be sharp instead of generic. For a TCC arguing that ORM design has been over-generalized, it's a load-bearing choice.

## Why OOR matters in a crowded market

TypeORM proved decorator-based TypeScript ORMs work. It also accumulated the constraints of being early: a transitional decorator dialect, a global metadata registry, raw SQL strings as the primary `where()` API, `any` at the parameter seam, and a wide-but-shallow database matrix. Every one of those is a reasonable answer to a 2018 question. None of them is a load-bearing answer to a 2026 question.

OOR's contribution is the same shape with the constraints lifted: Stage-3 dialect, library-owned metadata, typed predicates, no-`any` public surface, PG-specific type rules. Each is incremental on its own; together they're a meaningfully different ergonomic profile. That's the case the TCC defends.

## Sources

- TypeORM docs: <https://typeorm.io>
- TypeORM repo: <https://github.com/typeorm/typeorm>
- Getting started: `https://github.com/typeorm/typeorm/blob/master/docs/docs/getting-started.md`
- Indexes: `https://github.com/typeorm/typeorm/blob/master/docs/docs/indexes.md`
- Inheritance: `https://github.com/typeorm/typeorm/blob/master/docs/docs/entity/3-entity-inheritance.md`
- Query builder: `https://github.com/typeorm/typeorm/blob/master/docs/docs/query-builder/1-select-query-builder.md`
- OOR ADRs: [[0001-stage-3-decorators]], [[0002-repository-with-lazy-query-builder]], [[0004-parameterized-sql-only]], [[0005-no-any-type-driven-api]]
