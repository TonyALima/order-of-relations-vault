# Welcome to the OOR Vault

**Order of Relations (OOR)** is a TypeScript ORM library for PostgreSQL. It is simultaneously two things:

- A **TCC** (undergraduate thesis) — the academic artifact.
- A **publishable npm package** — the engineering artifact.

This vault is where I capture the *why*: the ideas, the trade-offs, and the decisions that don't fit cleanly inside source comments or the public `README.md`.

---

## What OOR Is

A small, opinionated ORM that maps decorated TypeScript classes to PostgreSQL tables and exposes a fluent, type-safe query builder on top of a generic `Repository<T>`.

The pillars:

- **Decorators** describe entities, columns, and relations.
- **Repositories** are the entry point for persistence operations.
- **A query builder** composes `WHERE` / `JOIN` / `ORDER BY` clauses lazily.
- **A DI container** wires services and injects repositories.
- **Schema-based migrations** evolve the database over time.

For the API surface (decorator list, column type mapping, usage snippets), see the project's `README.md`. This vault is for everything that sits *behind* that surface.

---

## High-Level Decisions

### Stage-3 decorators, no `reflect-metadata`

OOR uses [[ECMAScript Stage-3 Decorators]] rather than the legacy TypeScript `experimentalDecorators` + `reflect-metadata` combination.

- Stage-3 is the path forward — it's already in the language proposal pipeline, supported natively by modern runtimes, and avoids a runtime dependency on the global `Reflect.metadata` polyfill.
- Metadata is stored in a custom `metadataStorage` `Map`, owned by the library, instead of piggy-backing on `Reflect`.
- Trade-off: the decorator API is slightly more verbose than the legacy form, and the ecosystem of helpers is smaller. Acceptable for a greenfield project.

### Repository pattern with a lazy query builder

`Repository<T>` is the single entry point for an entity's persistence. `Repository.find()` does **not** execute SQL — it returns a `QueryBuilder<T>` that accumulates clauses and runs only on terminal calls (`getMany()`, `getOne()`).

This keeps the boundary clean: repositories handle simple `findOne` / `create` cases; the query builder handles anything composed.

### DI container as a singleton

A single `Container` holds service singletons. `@Service` wraps a constructor so that `@Inject` and `@InjectRepository` fields are populated automatically.

The container is intentionally minimal — it exists to make `Repository<T>` injection ergonomic, not to compete with serious DI frameworks.

### SQL safety: parameterized only

A hard rule: **no `sql.unsafe` anywhere in the codebase.** Every query goes through parameterized templates or the query builder. SQL injection is treated as a non-negotiable correctness property, not a hardening pass.

### Type-driven API: no `any`

`@typescript-eslint/no-explicit-any` is enforced strictly. The library leans on generics, conditional types, and `unknown` to give consumers compile-time guarantees — for example, the `create()` signature requires non-optional fields and rejects partial entities at compile time.

### TDD as the implementation rhythm

Every feature lands by writing a failing test next to the source file, then making it green, then refactoring. Test files live alongside source (`foo.ts` / `foo.test.ts`) rather than in a separate `tests/` tree.

Run with `bun test`.

### Bun as the toolchain

Bun replaces `node` + `ts-node` + `npm` + `jest` + `webpack`. Single binary, native TypeScript, native test runner, native `.env` loading. Reduces toolchain surface area to one thing.

---

## Vault Map

A few notes that should grow over time. Links may resolve to empty pages today — those are intentional placeholders.

- [[Architecture Overview]] — how the pieces fit together end to end.
- [[Decorator Metadata Storage]] — the `metadataStorage` Map and how decorators write to it.
- [[Query Builder Design]] — clause accumulation, lazy execution, type narrowing.
- [[Repository Contract]] — what `create()`, `findOne()`, and `find()` guarantee.
- [[Schema Migrations]] — how schema evolution is modelled.
- [[Decisions Log]] — short, dated entries for design calls worth remembering.
- [[Open Questions]] — things I'm still unsure about and want to revisit.

---

## Project Status

Active development. The TCC angle pulls toward a clean conceptual model and good written justification; the npm angle pulls toward stability, ergonomics, and a public API I won't regret. Both pulls are usually compatible — when they aren't, this vault is where I work it out.
