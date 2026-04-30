---
type: meta
title: 'OOR — 30-Second Brief'
created: 2026-04-29
updated: 2026-04-29
tags:
  - meta
  - brief
  - agent-context
status: evergreen
related:
  - '[[overview]]'
  - '[[hot]]'
  - '[[index]]'
sources: []
---

# Order of Relations — 30-Second Brief

> [!note] Audience
> This page is the **agent-facing single-pager** for code agents working inside `../order-of-relations` (the sibling code repo). It is referenced from that repo's `CLAUDE.md` so every agent picks up project context in seconds without paying for the full wiki. It evolves with the wiki — treat the body as stable, the _facts_ as living.

## What OOR is

An opinionated TypeScript ORM for PostgreSQL. Decorated classes → metadata → `Repository<T>` → fluent `QueryBuilder<T>` → parameterized SQL via Bun's `sql` driver. Doubles as a UNIFEI undergraduate thesis (TCC) and a publishable npm package.

## Hard rules (do not violate)

- **ECMAScript Stage-3 decorators only.** No `reflect-metadata`. → [[0001-stage-3-decorators]]
- **Parameterized SQL only.** `sql.unsafe` is banned codebase-wide. No `qb.raw()` escape hatch. → [[0004-parameterized-sql-only]]
- **Strict no-`any`.** Public API is type-driven; required-on-the-type means required-at-the-call-site. → [[0005-no-any-type-driven-api]]
- **TDD with `bun test`.** Red → green → refactor. Unit tests colocated under `src/`; integration under top-level `tests/`. → [[0006-tdd-rhythm]]
- **Bun is the single toolchain.** Runtime, package manager, test runner, bundler. No npm/yarn/pnpm. → [[0007-bun-toolchain]]
- **`@Nullable` must be the inner decorator** when stacked with `@Column`. `@PrimaryColumn` is exempt.
- **`sqlJoin` is the only sanctioned fragment joiner.** Hand-rolled `reduce` over fragments is rejected as a footgun.

## Architecture in five layers

Downward dependencies only. Decorators are the only writers of metadata.

1. **Decorators** — `@Entity`, `@Column`, `@PrimaryColumn`, `@Nullable`, relation decorators. Write to three symbol keys on `context.metadata`: `COLUMNS_KEY`, `RELATIONS_KEY`, `NULLABLE_KEY`.
2. **Metadata** — [[MetadataStorage]] is **per-`Database`** (not library-global). `Map<Constructor, EntityMetadata>`, lazy-resolved, idempotent on late additions.
3. **Repository** — narrow by-key surface, six public methods total. `create` / `findById` / `update` / `delete` funnel through one `requirePrimaryKey` gate. `findOne` / `findMany` are the composition entry points (each builds a one-shot `QueryBuilder` internally; there is no caller-facing `find()`). Single error type: `IncompletePrimaryKeyError`. Driver errors propagate unwrapped.
4. **QueryBuilder** — mutable, single-owner. State is one field: `conditions: Condition[] = []`. Two terminals: `getMany()`, `getOne()`. `applyOptions()` replaces (not accumulates).
5. **Driver** — Bun's `sql` (PostgreSQL only).

## Method-shape facts worth knowing up front

- `create(entity: UnbrandedT<T>)` — signature is `T` (modulo brand-stripping on input), not `Partial<T>`. Returns `PKOutput<T>` with **only PK fields** (branded via `PrimaryKey<V>`; not a hydrated entity).
- `findById(key: PKInput<T>)` / `delete(key: PKInput<T>)` — strict PK shape, every key required, all unbranded. `findById({})` and `findById({ name: 'x' })` are compile errors. → [[PrimaryKey Brand]]
- `update(entity: UnbrandedT<T> & PKInput<T>)` — full entity, plus PK keys required regardless of `T`'s optional modifier. Closes the silent-`update({ name: 'x' })` bug on autogen entities. → [[0008-pk-aware-compile-time]]
- Autogeneration is **explicit-only**: two strategies (`clientSide` / `dbSide`); a caller-supplied value always wins. → [[Autogeneration]]
- `where` callback shape: `(conditions: Conditions<T>) => (Condition | undefined)[]`. Missing-column entries throw `UndefinedWhereConditionError` carrying the offending **index**.
- `FindOptions<T>` also accepts `inheritance: InheritanceSearchType` (`ALL` / `ONLY` / `SUBCLASSES`) — opts a read into / out of subclass-scoped discriminator filtering. See [[Single-Table Inheritance]] § Reading Across the Hierarchy.
- DI container is **decided but not yet implemented**; current code uses direct `new Repository(User, db)`. → [[0003-singleton-di-container]]
- **`COLUMN_TYPE` is a closed enum.** Decorators can only declare types from this enum — no string escape hatch. To support a new PostgreSQL type, add a member and a branch in `getColumnTypeDefinition`. → [[modules/sql-types]]
- **FK columns referencing a SERIAL PK are emitted as `INTEGER`, not `SERIAL`.** Because `SERIAL` = `INTEGER` + sequence + `DEFAULT nextval(...)`, an FK that copied the type would create a bogus second sequence. `toForeignKeyType()` demotes `SERIAL`→`INTEGER`, `SMALLSERIAL`→`SMALLINT`, `BIGSERIAL`→`BIGINT` automatically; identity for every other type. Hand-written migrations against entity declarations need to mirror this. → [[modules/sql-types]] § Foreign-key type demotion

## Where to dig deeper

| When you need…                       | Read                                                                                                   |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------ |
| Full architectural map               | [[overview]] · [[Layered Architecture]]                                                                |
| Most-recent facts / churn            | [[hot]]                                                                                                |
| Master catalog of every page         | [[index]]                                                                                              |
| End-to-end walks                     | [[query-lifecycle]] · [[lifecycle-of-a-create]] · [[entity-registration]] · [[schema-create-drop]]     |
| Decision rationale                   | [[decisions/_index]]                                                                                   |
| Repository contract                  | [[Repository]] · [[Repository Pattern]]                                                                |
| Builder mechanics                    | [[QueryBuilder]] · [[Lazy Query Builder]] · [[Conditions Proxy]] · [[Parameterized SQL]] · [[sqlJoin]] |
| Entity registration & metadata shape | [[MetadataStorage]] · [[Database]]                                                                     |
| Working call sites                   | `examples/basic-crud/` · `examples/inheritance/` · `examples/relations/` — see [[examples/_index]]     |

## Open questions (do not assume resolved)

- 🔓 [[decorator-order-independence]] — `@Column` / `@Nullable` order independence?
- 🔓 [[get-one-limit-1]] — `getOne()` slicing vs `LIMIT 1`?
- 🔓 [[apply-options-accumulation]] — `applyOptions()` replace vs accumulate?

This brief stays small on purpose. Anything that grows belongs in a dedicated wiki page; only load-bearing rules and pointers stay here.
