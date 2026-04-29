---
type: overview
title: "Wiki Index"
created: 2026-04-29
updated: 2026-04-29
tags:
  - meta
  - index
status: developing
page_count: 52
related: []
sources: []
---

# Wiki Index

Master catalog for the **Order of Relations (OOR)** vault. Update this file on every ingest. Pages are grouped by type, then by domain.

> [!key-insight] What is OOR?
> A TypeScript ORM for PostgreSQL, written as both a TCC (undergraduate thesis) and a publishable npm package. This vault is the *why* behind the code.

---

## Top-level

- [[brief]] — 30-second agent-facing project card (linked from `../order-of-relations/CLAUDE.md`)
- [[overview]] — executive summary of the wiki
- [[hot]] — recent context cache (~500 words)
- [[log]] — chronological record of every operation
- [[getting-started]] — how to use this vault

---

## Modules — `wiki/modules/`

One note per major module / package / service in the codebase. Per-leaf coverage is **partial** by design — the [[modules/_index]] explains which modules have dedicated pages and which lean on existing component / concept / flow pages.

- [[modules/sql-types]] — `COLUMN_TYPE` enum (closed, ~50 PG types); `getColumnTypeDefinition` (only DDL fragment producer); `toForeignKeyType` (SERIAL→INTEGER FK demotion).
- See [[modules/_index]] for the full source-tree map and the deferred-pages table.

## Components — `wiki/components/`

Reusable building blocks (decorators, query-builder fragments, container primitives).

- [[MetadataStorage]] — `Map<Constructor, EntityMetadata>` per `Database`.
- [[Database]] — connection + metadata host + schema lifecycle.
- [[QueryBuilder]] — mutable, single-owner; one `conditions[]` field; `getMany()` / `getOne()`.
- [[Repository]] — narrow by-key surface; `requirePrimaryKey` gate; single error type.
- See [[components/_index]] for the rolling index.

## Decisions — `wiki/decisions/`

Architecture Decision Records. Append-only, dated.

- [[0001-stage-3-decorators]] — Stage-3 decorators; no `reflect-metadata`.
- [[0002-repository-with-lazy-query-builder]] — `Repository<T>` entry point + lazy `QueryBuilder<T>`.
- [[0003-singleton-di-container]] — minimal singleton DI container.
- [[0004-parameterized-sql-only]] — `sql.unsafe` banned codebase-wide.
- [[0005-no-any-type-driven-api]] — strict no-`any`; type-driven public API.
- [[0006-tdd-rhythm]] — red-green-refactor with `bun test`, colocated tests.
- [[0007-bun-toolchain]] — Bun is the single toolchain.
- See [[decisions/_index]] for the rolling index.

## Dependencies — `wiki/dependencies/`

External packages, runtime deps, version pins, risk.

- _none yet — see [[dependencies/_index]]_

## Flows — `wiki/flows/`

Data flows and request paths (e.g., lifecycle of a `findMany` call).

- [[entity-registration]] — decorator evaluation → `@Entity` validation → `MetadataStorage.set` → lazy resolution.
- [[query-lifecycle]] — six-step walkthrough of a composed read.
- [[lifecycle-of-a-create]] — seven-step walkthrough of a write (`create()` end to end).
- [[schema-create-drop]] — two-pass `CREATE TABLE`; topologically reversed `DROP`.
- See [[flows/_index]] for the rolling index.

---

## Sources — `wiki/sources/`

One synthesis page per item in `.raw/`.

- [[sources/welcome|Welcome to the OOR Vault]] — manifesto: pillars + 7 ADR seeds (2026-04-29).
- [[sources/architecture-overview|Architecture Overview]] — five-layer stack, query lifecycle, schema create/drop; refines 3 claims from welcome.md (2026-04-29).
- [[sources/decorator-metadata-storage|Decorator Metadata Storage]] — storage shape, three-symbol scheme, lazy resolution, failure modes (2026-04-29).
- [[sources/query-builder-design|Query Builder Design]] — mutable builder, where-callback signature, SQL-composition safety; corrects two claims (2026-04-29).
- [[sources/repository-contract|Repository Contract]] — by-key operations, `requirePrimaryKey` gate, `create(entity: T)` type-shaped contract, autogeneration (2026-04-29).
- ✅ All five primary `.raw/` sources ingested.

**Drift corrections (2026-04-29):**

- [[sources/drift-d1-repository-find]] — `Repository.find()` does not exist; rewrite read-API framing.
- [[sources/drift-d3-find-options-inheritance]] — document `FindOptions.inheritance` and the `InheritanceSearchType` enum.
- [[sources/drift-d4-conditions-proxy-operators]] — fix `FieldConditionBuilder` operator inventory (no `neq` / `before` / `after`).
- [[sources/drift-d5-discriminator-index]] — document the implicit `idx_discriminator` index emitted by schema-create.
- [[sources/drift-m3-module-pages]] — partial close of the `wiki/modules/` backlog.
- [[sources/drift-m5-sql-types-module]] — file `src/core/sql-types/` (the closed enum + DDL helpers + FK demotion).
- [[sources/drift-m6-examples-pointer]] — make `examples/` visible via `wiki/examples/` front-doors.

## Entities — `wiki/entities/`

People, organizations, products, repos, libraries that show up in the work.

- [[order-of-relations]] — **the codebase**. Local: `../order-of-relations`. Remote: <https://github.com/TonyALima/order-of-relations>.
- [[Bun]] — runtime + package manager + test runner + bundler; the single toolchain.
- [[PostgreSQL]] — sole target database.
- [[TypeScript]] — host language; strict no-`any`.
- See [[entities/_index]] for the rolling index.

## Concepts — `wiki/concepts/`

Ideas, patterns, frameworks (e.g., Stage-3 decorators, repository pattern, lazy query builder).

- [[Layered Architecture]] — five-layer stack; downward-only dependencies.
- [[ECMAScript Stage-3 Decorators]] — the standardized decorator dialect.
- [[Single-Table Inheritance]] — discriminator-based inheritance, lazily resolved.
- [[Relation Target Thunk]] — `() => User` closure pattern for circular entity graphs.
- [[Repository Pattern]] — `Repository<T>` as persistence entry point.
- [[Lazy Query Builder]] — clauses accumulate; SQL runs only on terminal calls.
- [[Conditions Proxy]] — the typed proxy passed to a `where` callback.
- [[Parameterized SQL]] — placeholders + bound parameters; the only allowed query form.
- [[sqlJoin]] — the only sanctioned fragment joiner.
- [[Autogeneration]] — explicit-only PK auto-generation; `clientSide` / `dbSide` strategies.
- [[Schema Migrations]] — *seed* — placeholder concept page; no migration system is built yet.
- [[Dependency Injection Container]] — minimal singleton container (planned, not yet implemented).
- See [[concepts/_index]] for the rolling index.

## Domains — `wiki/domains/`

Top-level topic areas (e.g., ORM design, type systems, PostgreSQL).

- _none yet — see [[domains/_index]]_

## Examples — `wiki/examples/`

Wiki front-doors for the runnable scenarios in `examples/`.

- [[examples/basic-crud]] — minimum-viable CRUD (one entity, three Repository methods).
- [[examples/inheritance]] — STI scenario; **the only place `InheritanceSearchType` is shown in use**.
- [[examples/relations]] — `@ToOne` + FK auto-derivation + SERIAL→INTEGER demotion (stub).
- See [[examples/_index]] for the rolling index.

## Comparisons — `wiki/comparisons/`

Side-by-side analyses (e.g., OOR vs TypeORM vs Drizzle).

- _none yet — see [[comparisons/_index]]_

## Questions — `wiki/questions/`

Filed answers to queries, plus **open questions** kept visible until resolved. Open questions carry `impact` + `effort` frontmatter and surface in the [[issues|Issue tracker]] Base.

- 🔓 [[decorator-order-independence]] — *open · medium / S* — could `@Column` / `@Nullable` work regardless of decoration order?
- 🔓 [[support-user-indexes]] — *open · medium / M* — add `@Index` / `@Unique` so users can declare indexes?
- 🔓 [[get-one-limit-1]] — *open · low / S* — `getOne()` slicing vs. `LIMIT 1`?
- 🔓 [[apply-options-accumulation]] — *open · low / S* — `applyOptions()` replace vs. accumulate?
- See [[questions/_index]] for the rolling index.

## Meta — `wiki/meta/`

Dashboards, lint reports, conventions.

- [[issues|Issue tracker]] — Bases view over open questions, sorted by impact × effort score.
- See [[meta/_index]] for the rolling index.
