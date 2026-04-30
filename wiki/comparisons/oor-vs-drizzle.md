---
type: comparison
title: "OOR vs Drizzle"
subjects:
  - "[[order-of-relations]]"
  - "Drizzle ORM"
dimensions:
  - schema declaration
  - type derivation
  - entry-point API
  - query builder
  - SQL safety
  - indexes
  - inheritance
  - database scope
  - toolchain
verdict: "Drizzle proved that strong TypeScript inference is enough to make a SQL-shaped API feel safe. OOR makes a different bet: that the ergonomics of decorated classes — entity-as-type, Repository-as-entry-point, native STI — are worth keeping, and that they can be paired with the same strong inference and the same parameterized-SQL guarantees. The contribution is the OO ergonomic profile, modernized."
created: 2026-04-29
updated: 2026-04-30
tags:
  - comparison
  - drizzle
  - schema-builder
  - sql-first
status: developing
related:
  - "[[0001-stage-3-decorators]]"
  - "[[0002-repository-with-lazy-query-builder]]"
  - "[[0004-parameterized-sql-only]]"
  - "[[ECMAScript Stage-3 Decorators]]"
  - "[[Repository Pattern]]"
  - "[[Lazy Query Builder]]"
  - "[[Parameterized SQL]]"
  - "[[Single-Table Inheritance]]"
  - "[[support-user-indexes]]"
sources: []
---

# OOR vs Drizzle

## What OOR brings that's new

- **Class-as-schema, not value-as-schema.** In OOR, `User` is both the runtime entity class and the static type — methods and validation can hang off it. In Drizzle, `users` is a runtime value (a table descriptor) and the type is a separate inference step (`typeof users.$inferSelect`).
- **A `Repository<T>` entry-point with a clean simple-vs-composed boundary.** Drizzle has no Repository abstraction — there's the low-level core builder and an opt-in Relational Queries layer; users either reach for raw SQL-shaped helpers or learn the higher-level shape. OOR commits to one entry per entity. ([[0002-repository-with-lazy-query-builder]])
- **A typed predicate API rather than imported SQL helpers.** `cb.email.eq(value)` instead of `eq(users.email, value)` — column types are accessors on a typed proxy, not arguments to free functions. The user never imports `eq`, `and`, `or`, `gt`, `lt`. ([[Conditions Proxy]])
- **Native Single-Table Inheritance.** Drizzle has none — open feature request since the project's early days; the userland workaround is shared column-definition objects spread into multiple tables, which is not STI. OOR ships discriminator-based STI with an auto-emitted index and lazy resolution. ([[Single-Table Inheritance]])
- **Hard SQL safety, not just safe-by-default.** Drizzle's `` sql`...` `` is parameterized, but `sql.raw(...)` exists as the documented escape hatch. OOR has no equivalent escape hatch — the unsafe path is forbidden in library internals as well as user code. ([[0004-parameterized-sql-only]])

## Overview

Drizzle is the exemplar of a different ORM philosophy: bend the type system to fit SQL. Schemas are runtime table-builder objects (`pgTable("users", { ... })`); queries are SQL-shaped (`db.select().from(users).where(eq(users.id, 1))`); types are inferred from the schema rather than declared on a class. It's a coherent design and Drizzle executes it well.

OOR makes the opposite bet. Schemas are decorated TypeScript classes; queries flow through a `Repository<T>` and a typed predicate proxy; the class declaration *is* both the runtime metadata and the static type. The contribution isn't "Drizzle is wrong" — it's "the OO ergonomic profile (entity-as-type, methods on entities, single entry-point per entity, native polymorphism) was worth keeping, and it can be paired with the same modern guarantees Drizzle achieves."

> [!key-insight] Two coherent answers to the same question
> "How do we get type-safe access to PostgreSQL from TypeScript?" Drizzle answers it by typing SQL. OOR answers it by typing classes. Both work; the comparison is about which ergonomic profile a TypeScript codebase wants to live in.

## Comparison

> [!note] Equal-value rule for planned features
> Planned features with a fully-scoped open-question page (axes spelled out, decorator surface defined, change-surface described) are listed below as part of OOR's contribution.

| Dimension | OOR | Drizzle ORM (1.0-beta) |
|-----------|-----|------|
| **Schema declaration** | Decorated TypeScript classes. The class *is* the entity shape; methods and behavior can hang off it. ([[ECMAScript Stage-3 Decorators]]) | Runtime table-builder objects: `pgTable("users", { id: integer().primaryKey(), name: varchar().notNull() })`. Dialect-specific (`pgTable` vs `mysqlTable` vs `sqliteTable`). |
| **Type derivation** | The class declaration is both the runtime metadata and the static type — `User` works as both `class User` and `type User`. No inference step. | Static type is **derived** from the runtime schema via `typeof users.$inferSelect` / `$inferInsert`. No codegen, but the type is one inference step removed from the declaration. |
| **Entry-point API** | `Repository<T>` per entity — the single entry point. Simple ops on `Repository`, composition via `Repository.find()` returning a `QueryBuilder<T>`. ([[Repository Pattern]], [[0002-repository-with-lazy-query-builder]]) | No `Repository<T>`. Two coexisting layers: low-level core builder (`db.select().from(users).where(eq(users.id, 1))`) and the opt-in Relational Queries (RQB) API (`db.query.users.findMany({ with: { posts: true } })`). |
| **Query shape** | Typed [[Conditions Proxy]]: `where(cb => cb.firstName.eq("Tony"))`. No SQL strings, no operator helpers — just typed accessors per column. | SQL-shaped helpers imported from `drizzle-orm`: `where(eq(users.firstName, "Tony"))`, `and(...)`, `or(...)`, `gt(...)`, `lt(...)`, `inArray(...)`. |
| **Builder execution** | Mutable, single-owner. Lazy. Terminal call (`getMany()` / `getOne()`) executes. ([[Lazy Query Builder]]) | Lazy thenable — `await db.select()...` triggers execution. |
| **SQL safety** | `sql.unsafe` banned codebase-wide. No raw interpolation anywhere — no escape hatch even in library internals. ([[0004-parameterized-sql-only]]) | `` sql`...${value}` `` is parameterized. `sql.raw(...)` is the documented escape hatch — a deliberate keystroke required, but available. |
| **Indexes** | Property-level + class-level `@Index` and `@Unique`, with a uniform naming policy that also covers the STI discriminator index. Scoped in [[support-user-indexes]]. | First-class. Third arg of `pgTable` returns an array: `[index("name_idx").on(t.name), uniqueIndex("email_idx").on(t.email)]`. Composite, partial, functional, opclass, fillfactor — a near-complete PG index surface. |
| **Inheritance** | Single-Table Inheritance with discriminator column, decorator-driven, lazy resolution, auto-emitted discriminator index. ([[Single-Table Inheritance]]) | No native STI. Open feature request; userland workaround is shared column-definition objects spread into multiple tables. |
| **Database scope** | PostgreSQL, deeply. Closed [[modules/sql-types\|`COLUMN_TYPE` enum]] of ~50 PG types; PG-specific type rules (FK demotion) at the type level. | Multi-dialect: PostgreSQL, MySQL, SQLite, SingleStore, CockroachDB, MSSQL. Schemas are dialect-specific — same TypeScript file, but `pgTable` ≠ `mysqlTable`. |
| **Toolchain** | Bun-only. Stage-3 decorators native; `bun test` for TDD. ([[0007-bun-toolchain]]) | First-class Bun support; edge runtimes (Cloudflare Workers, Durable Objects, D1). `drizzle-kit` CLI for migrations. |

## OOR's contribution, dimension by dimension

### Class-as-schema

In Drizzle:

```ts
export const users = pgTable("users", {
  id: integer().primaryKey().generatedAlwaysAsIdentity(),
  name: varchar().notNull(),
});

export type User = typeof users.$inferSelect;
```

The schema (`users`) is a runtime value. The type (`User`) is one inference step away. There's no class to attach methods to — entity behavior, validation, computed fields all live elsewhere. That's a deliberate, FP-leaning choice that suits some codebases.

In OOR:

```ts
@Entity
class User {
  @PrimaryColumn({ type: COLUMN_TYPE.SERIAL })
  id!: number;

  @Column({ type: COLUMN_TYPE.TEXT })
  @NotNullable
  name!: string;

  // Methods, validation, computed fields all hang here naturally
  fullName() { return this.name; }
}
```

The class is both the runtime metadata and the static type. Behavior lives where the data does. For OO-leaning TypeScript codebases — which the language continues to support and the standard library continues to model — this is the ergonomic profile that "fits."

### Repository<T> as entry-point

Drizzle's core API is intentionally SQL-shaped:

```ts
const result = await db.select()
  .from(users)
  .where(and(eq(users.email, email), gt(users.createdAt, since)));
```

The Relational Queries layer (`db.query.users.findMany({ with: { posts: true } })`) is the closest analogue to OOR's `Repository.find()`, but it's an *additional* surface layered on top of the SQL builder, not the entry-point. A user has to choose per query which API shape to reach for.

OOR collapses this. Per entity there is one `Repository<T>`. Simple ops are methods on it (`findOne`, `findMany`, `findById`, `create`, `update`, `delete`); composition is `Repository.find()` returning a typed builder. The user never has to choose a layer.

### Typed predicate proxy

```ts
// OOR
qb.where(cb => cb.email.eq(email).and(cb.createdAt.gt(since)));

// Drizzle
db.select().from(users).where(and(eq(users.email, email), gt(users.createdAt, since)));
```

Both compile to the same parameterized SQL. The contribution is **where the operator names live**: in OOR they're methods on a typed proxy (so `cb.email` knows the column type and constrains the right-hand side); in Drizzle they're free functions imported from `drizzle-orm` and called with column references.

OOR's reading: a column knows its own operators. The proxy is auto-generated from the entity's column metadata; the user never imports `eq`, `and`, `or`, `gt`, `lt`. That keeps the surface area discoverable through `cb.<tab>` autocomplete instead of through documentation, and it constrains type-incompatible operations at the call site.

### Native Single-Table Inheritance

This is OOR's clearest feature contribution in the comparison. Drizzle has no native STI, no native CTI, and no polymorphic relations — the open feature request has been pending since the project's early days. The userland workaround spreads shared column definitions across multiple tables, which is *table-per-class with code reuse*, not STI.

OOR ships STI as a first-class feature: `@Entity({ inheritance: { type: "STI", discriminatorValue: "..." } })`, lazy resolution, auto-emitted discriminator index, and a `FindOptions.inheritance` field with `InheritanceSearchType` (`ALL` / `ONLY` / `SUBCLASSES`) controlling which subclasses a query covers. ([[Single-Table Inheritance]], [[schema-create-drop]])

For applications that model polymorphic hierarchies — admins/users, base-product/specialized-product, role-based actors — this isn't an opinion preference; it's a feature Drizzle simply doesn't have.

### Hard SQL safety

Drizzle's safety story is "safe by default, unsafe by deliberate keystroke":

```ts
// safe — parameterized
sql`SELECT * FROM users WHERE id = ${id}`

// explicit unsafe escape hatch
sql.raw(userInput)
```

That's a real, documented improvement over libraries with implicit unsafe paths. OOR goes further: there is no `sql.raw` equivalent, and `sql.unsafe` is banned codebase-wide. Library internals can't reach for it either. The class of injection bugs that ship through `sql.raw` calls in production Drizzle code can't ship through OOR by construction. ([[0004-parameterized-sql-only]])

### PostgreSQL, deeply

Drizzle's multi-dialect promise is at the runtime level (one Drizzle codebase can target multiple DBs with parallel schema files), not at the schema level (one schema file targeting many DBs). `pgTable` and `mysqlTable` are different functions with different available column types.

OOR doesn't try to be portable. The [[modules/sql-types]] module is PG-shaped — `SERIAL`, `JSONB`, `TIMESTAMPTZ` — and the FK demotion logic (`SERIAL → INTEGER`) is PG-specific behavior encoded at the type level. The narrowing of scope is the contribution: PG affordances become first-class types instead of adapter glue.

## Why OOR matters in a crowded market

Drizzle's success showed that strong TypeScript inference can carry a SQL-shaped API to the same destination ORMs reach by other means. That doesn't render decorator-based ORMs obsolete — it raises the bar for what they have to deliver. Specifically: they have to match Drizzle's SQL safety, type strictness, and toolchain alignment, while also justifying the OO ergonomic profile they bring.

OOR's argument is that the OO ergonomic profile *is* worth bringing. Class-as-schema, methods-on-entities, single-entry-point-per-entity, native polymorphism — these are not just preferences; they're load-bearing in OO-leaning codebases the way value-as-schema and free helpers are load-bearing in FP-leaning ones. The contribution is meeting Drizzle's bar on the modern guarantees while keeping (and modernizing) the ergonomic profile decorator ORMs were always good at.

## Sources

- Drizzle docs: <https://orm.drizzle.team>
- Schema declaration: <https://orm.drizzle.team/docs/sql-schema-declaration>
- Indexes & constraints: <https://orm.drizzle.team/docs/indexes-constraints>
- Relational Queries: <https://orm.drizzle.team/docs/rqb>
- SQL safety: <https://orm.drizzle.team/docs/sql>
- Repo: <https://github.com/drizzle-team/drizzle-orm>
- STI feature request: <https://github.com/drizzle-team/drizzle-orm/issues/900>
- OOR ADRs: [[0002-repository-with-lazy-query-builder]], [[0004-parameterized-sql-only]]
