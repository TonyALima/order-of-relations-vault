---
type: comparison
title: "OOR vs Prisma"
subjects:
  - "[[order-of-relations]]"
  - "Prisma"
dimensions:
  - source of truth
  - codegen vs runtime metadata
  - query API
  - composition
  - SQL safety
  - inheritance
  - database scope
  - toolchain
verdict: "Prisma's success made schema-first + codegen the default expectation for typed ORMs. OOR proposes the alternative the field has under-explored: TypeScript itself as the schema, with no DSL, no codegen step, and no generated artifacts. The contribution is a fluent composable query builder over class-declared entities — a path Prisma's design closed off, that turns out to be worth keeping open."
created: 2026-04-29
updated: 2026-04-30
tags:
  - comparison
  - prisma
  - schema-first
  - codegen
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
sources: []
---

# OOR vs Prisma

## What OOR brings that's new

- **TypeScript as the single source of truth.** No `schema.prisma` DSL, no `prisma generate` step, no generated client folder to manage. The decorated TypeScript class is both the runtime metadata and the static type. ([[ECMAScript Stage-3 Decorators]])
- **A fluent, composable, lazy `QueryBuilder<T>`.** Half-built builders are typed values you can pass between functions and refine before terminal call. Prisma's `findMany({ where, include, orderBy })` is an object-literal argument with no chainable refinement.
- **Native Single-Table Inheritance.** Prisma documents STI as a userland pattern (manage the discriminator yourself, write your own queries); the polymorphic-relations RFC has been open since Prisma 1. OOR ships STI with decorator support, lazy resolution, and an auto-emitted discriminator index. ([[Single-Table Inheritance]])
- **Hard SQL safety.** Prisma exposes `$queryRawUnsafe` as a documented escape hatch; OOR has no equivalent and bans `sql.unsafe` codebase-wide. ([[0004-parameterized-sql-only]])
- **Zero build step in the development loop.** Add a `@Column`, run tests. No `prisma generate`, no regenerated client, no out-of-band file to commit-or-gitignore.

## Overview

Prisma is the most-installed JavaScript ORM and the dominant force in the "schema-first + generated client" category. Its 2025–2026 architectural pivot (Rust-free TypeScript Query Compiler in v7) retired the historical "Prisma is overweight at runtime" critique, which sharpens the comparison: what's left is a clean philosophical fork between *schema-as-DSL* and *schema-as-code*.

> [!key-insight] OOR explores the path Prisma's design closed off
> Prisma's choice to put the schema in its own DSL was a coherent answer to "where should the schema live?" It produced a strong product, and it foreclosed an alternative the TypeScript ecosystem has under-explored: classes-as-schema with a fluent composable builder. OOR is the long-form argument that this alternative was worth keeping open.

## Comparison

> [!note] Equal-value rule for planned features
> Planned features with a fully-scoped open-question page (axes spelled out, decorator surface defined, change-surface described) are listed below as part of OOR's contribution.

| Dimension | OOR | Prisma (v7) |
|-----------|-----|-------------|
| **Source of truth** | Decorated TypeScript classes — the class *is* the schema, in TypeScript. No DSL, no generated artifacts, no separate file to keep in sync. | `schema.prisma` — a separate DSL file. The TypeScript types are *generated from* the DSL. |
| **Schema language** | TypeScript + Stage-3 decorators. ([[ECMAScript Stage-3 Decorators]]) | Prisma Schema Language (PSL) — its own grammar with `model`, `datasource`, `generator`, attribute decorators (`@id`, `@unique`, `@@index`). |
| **Codegen** | None. Decorators populate runtime metadata via [[MetadataStorage]] at module-load time. | Mandatory `prisma generate` step. Produces a typed client into a configurable output folder. Schema changes require regenerating. |
| **Entry-point API** | `Repository<T>` per entity. Simple ops execute directly; `Repository.find()` returns a lazy `QueryBuilder<T>`. ([[Repository Pattern]], [[0002-repository-with-lazy-query-builder]]) | `prisma.<modelName>` — model-namespaced methods on a single client (`prisma.user.findMany`, `prisma.post.create`). No `Repository<T>` shape; per-entity logic is userland. |
| **Query shape** | Lazy chainable builder: `userRepo.find().where(cb => cb.createdAt.gt(since)).orderBy("createdAt", "DESC").limit(5).getMany()` | Object literal per call: `prisma.user.findMany({ where: { createdAt: { gt: since } }, orderBy: { createdAt: "desc" }, take: 5 })` |
| **Composition** | Half-built builders are typed values — pass them around, layer additional `where()` calls, commit on terminal call. ([[Lazy Query Builder]]) | A half-built query is `Prisma.UserFindManyArgs` (an object literal type) — spreadable, but not chainable or refinable. |
| **SQL safety** | `sql.unsafe` banned codebase-wide. No raw interpolation anywhere — no escape hatch in library internals or user code. ([[0004-parameterized-sql-only]]) | Two tiers: `$queryRaw` / `$executeRaw` (tagged templates, parameterized) and `$queryRawUnsafe` / `$executeRawUnsafe` (string concatenation, explicitly unsafe — documented but available). |
| **Type strictness** | Strict no-`any` enforced via lint. Public API uses generics + conditional types. ([[0005-no-any-type-driven-api]]) | Generated client is fully typed; no documented "no-`any`" stance, but the public surface is strict in practice. |
| **Indexes** | Property-level + class-level `@Index` and `@Unique`, with a uniform naming policy that also covers the STI discriminator index. Scoped in [[support-user-indexes]]. | First-class in PSL: `@unique`, `@@unique`, `@@index`. Composite, named, functional, direction, Postgres index types. |
| **Inheritance** | Single-Table Inheritance with discriminator column, decorator-driven, auto-emitted discriminator index. ([[Single-Table Inheritance]]) | No native STI. Userland patterns (manage discriminator yourself); polymorphic relations RFC open since v1. |
| **Database scope** | PostgreSQL, deeply. Closed [[modules/sql-types\|`COLUMN_TYPE` enum]]; PG-specific type rules at the type level. | PostgreSQL, MySQL, MariaDB, SQLite, SQL Server, MongoDB, CockroachDB via driver adapters. |
| **Toolchain** | Bun-only. ([[0007-bun-toolchain]]) | Node primary; edge runtimes (Cloudflare Workers, Vercel Edge, Lambda) first-class since v7. |

## OOR's contribution, dimension by dimension

### TypeScript as single source of truth

Prisma's design splits the schema across two languages:

```prisma
// schema.prisma
model User {
  id    Int    @id @default(autoincrement())
  email String @unique
}
```
…then `npx prisma generate`, then `import { PrismaClient } from "./generated/client"`.

This produces a strong product. It also ties every schema change to a build step, an out-of-band source file in a different language, and a generated artifact whose versioning ("commit it or gitignore it?") is a project decision the team has to make.

OOR puts the schema in TypeScript:

```ts
@Entity
class User {
  @PrimaryColumn({ type: COLUMN_TYPE.SERIAL })
  id!: number;

  @Column({ type: COLUMN_TYPE.TEXT })
  @NotNullable
  email!: string;
}
```

No DSL, no codegen step, no generated folder, no out-of-band file. The class works as a value (instantiate it, call methods on it) and as a type (`User` in a function signature). Schema evolution happens in the same loop as everything else: edit the file, run tests.

This is OOR's most consequential contribution against Prisma. The trade is real — the PSL has a clean grammar that some teams prefer for its purpose-built ergonomics — but the "TypeScript as schema" path was foreclosed by Prisma's design choice, and the TypeScript ecosystem has under-explored it.

### Composable, chainable queries

Prisma's query API is a single function per operation:

```ts
const recent = await prisma.user.findMany({
  where: { createdAt: { gt: since } },
  orderBy: { createdAt: "desc" },
  take: 5,
});
```

For a single trivial query, this is compact and pleasant. The cost shows up in **composition**: a half-built query is `Prisma.UserFindManyArgs` (an object literal type). You can spread partial `where` clauses across helper functions, but you can't return a "query refined this far" value and let the caller chain more conditions onto it. The query-as-data-structure model is awkward to refine incrementally.

OOR's lazy `QueryBuilder<T>` is built for incremental refinement:

```ts
function recentUsers(repo: Repository<User>) {
  return repo.find().orderBy("createdAt", "DESC").limit(5);
}

function recentUsersWithEmail(repo: Repository<User>, domain: string) {
  return recentUsers(repo).where(cb => cb.email.endsWith(domain));
  // still a QueryBuilder<User>, not yet executed
}
```

A `QueryBuilder<T>` is a typed value that flows between functions, accumulates clauses, and runs only on the terminal call (`getMany()` / `getOne()` / `getCount()`). Building queries across helpers, layered conditions, or per-context refinements is the path of least resistance — not an edge case.

This is the wedge ADR 0002 names: "lazy execution makes the builder safely composable — you can pass a half-built query around, layer additional `where()` calls on top, and only commit to running it at the call site that knows it's complete."

### Repository<T> as entry-point

Prisma's per-entity surface is `prisma.<modelName>.<verb>()`. There's no `Repository<T>` abstraction; if a project wants per-entity service methods, it builds a userland repository layer on top of the Prisma client — a recurring recipe across the Prisma ecosystem.

OOR makes that layer the primary API. `Repository<T>` is the entry point per entity, with a closed surface of six methods plus `find()` for composition. Per-entity logic has a natural home; it doesn't have to be re-invented per project.

### Native Single-Table Inheritance

Prisma's official "Table inheritance" docs page recommends three userland patterns: STI via discriminator (the user manages it), class-table via 1:1 relations, and "delegated types." None of these is native — each is a pattern the user implements on top of the generated client. The polymorphic-relations RFC has been open since Prisma 1.

OOR ships STI as first-class: `@Entity({ inheritance: { type: "STI", discriminatorValue: "..." } })`, lazy resolution, auto-emitted discriminator index, and a `FindOptions.inheritance` parameter (`InheritanceSearchType`: `ALL` / `ONLY` / `SUBCLASSES`) controlling which subclasses a query covers.

For applications that model polymorphic hierarchies, this is a feature with no Prisma equivalent — not "Prisma does it differently," but "the user has to build it themselves."

### Hard SQL safety

Prisma's safety story is two-tier and explicit:

```ts
await prisma.$queryRaw`SELECT * FROM "User" WHERE email LIKE ${pattern}`     // safe
await prisma.$queryRawUnsafe(`SELECT * FROM "${tableName}" LIMIT 10`)          // unsafe
```

The boundary is well-marked, and Prisma's documentation tells users when to reach for `Unsafe`. That's better than libraries with implicit unsafe paths — but the unsafe path exists, and it ships through production Prisma code regularly.

OOR's stance ([[0004-parameterized-sql-only]]) is that even a documented unsafe path is too much trust: the next maintainer copies the pattern without the warning. There is no `$queryRawUnsafe` equivalent in OOR. Dynamic identifiers go through allowlists; library internals can't use the unsafe path either, by lint and by convention. SQL injection is treated as a non-negotiable correctness property, not a discipline.

### Zero build step

`prisma generate` is fast, but it's still a step. For a TypeScript developer used to the inner loop of "edit a class, run tests, see results," every additional step compounds: tooling has to know when to regenerate, IDEs have to refresh, generated artifacts have to be kept consistent across team members and CI.

OOR has no codegen. Decorators populate runtime metadata at module-load time. Edit the class, run `bun test`, see results. The development loop is the same loop the rest of the TypeScript code uses. ([[0007-bun-toolchain]])

### PostgreSQL, deeply

Prisma's adapter architecture flattens to the lowest common denominator across seven databases. PG-specific affordances (expression indexes, partial indexes, `JSONB` operators) become escape hatches or are not supported at all (partial indexes are not expressible in PSL).

OOR's [[modules/sql-types|`COLUMN_TYPE`]] enum is closed and PG-shaped. PG-specific affordances are first-class types. The narrowing of scope is the contribution: PG type rules become part of the contract instead of adapter glue.

## Why OOR matters in a crowded market

Prisma's design choice — schema-as-DSL, codegen-as-generation, model-namespaced calls — produced a category-defining product. It also defined the path TypeScript ORMs are *expected* to take. The path it didn't take — schema-as-code, runtime metadata, fluent composable builder over class entities — has been under-served in the ecosystem since Prisma's rise.

OOR's contribution is taking that path seriously. The class-as-schema model, the lazy `QueryBuilder<T>`, native STI, and the no-codegen development loop aren't refinements of Prisma's design — they're an alternative the field has had room for and not delivered. The TCC's claim is that the alternative is worth delivering.

## Sources

- Prisma docs: <https://www.prisma.io/docs>
- Prisma 7 announcement: <https://www.prisma.io/blog/announcing-prisma-orm-7-0-0>
- Rust-to-TypeScript migration: <https://www.prisma.io/blog/from-rust-to-typescript-a-new-chapter-for-prisma-orm>
- Table inheritance docs: <https://www.prisma.io/docs/orm/prisma-schema/data-model/table-inheritance>
- Repo: <https://github.com/prisma/prisma>
- OOR ADRs: [[0001-stage-3-decorators]], [[0002-repository-with-lazy-query-builder]], [[0004-parameterized-sql-only]], [[0005-no-any-type-driven-api]]
