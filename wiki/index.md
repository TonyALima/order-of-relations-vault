---
type: overview
title: "Wiki Index"
created: 2026-04-29
updated: 2026-04-29
tags:
  - meta
  - index
status: developing
page_count: 28
related: []
sources: []
---

# Wiki Index

Master catalog for the **Order of Relations (OOR)** vault. Update this file on every ingest. Pages are grouped by type, then by domain.

> [!key-insight] What is OOR?
> A TypeScript ORM for PostgreSQL, written as both a TCC (undergraduate thesis) and a publishable npm package. This vault is the *why* behind the code.

---

## Top-level

- [[overview]] — executive summary of the wiki
- [[hot]] — recent context cache (~500 words)
- [[log]] — chronological record of every operation
- [[getting-started]] — how to use this vault

---

## Modules — `wiki/modules/`

One note per major module / package / service in the codebase.

- _Module pages not yet filed; the [[modules/_index]] holds the source-tree map (8 directories cataloged with their concerns and layer assignments)._

## Components — `wiki/components/`

Reusable building blocks (decorators, query-builder fragments, container primitives).

- [[MetadataStorage]] — `Map<Constructor, EntityMetadata>` per `Database`.
- [[Database]] — connection + metadata host + schema lifecycle.
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
- [[schema-create-drop]] — two-pass `CREATE TABLE`; topologically reversed `DROP`.
- See [[flows/_index]] for the rolling index.

---

## Sources — `wiki/sources/`

One synthesis page per item in `.raw/`.

- [[sources/welcome|Welcome to the OOR Vault]] — manifesto: pillars + 7 ADR seeds (2026-04-29).
- [[sources/architecture-overview|Architecture Overview]] — five-layer stack, query lifecycle, schema create/drop; refines 3 claims from welcome.md (2026-04-29).
- [[sources/decorator-metadata-storage|Decorator Metadata Storage]] — storage shape, two-symbol scheme, lazy resolution, failure modes (2026-04-29).
- _pending ingest: `query-builder-design.md`, `repository-contract.md` — see [[sources/_index]]._

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
- [[Dependency Injection Container]] — minimal singleton container (planned, not yet implemented).
- See [[concepts/_index]] for the rolling index.

## Domains — `wiki/domains/`

Top-level topic areas (e.g., ORM design, type systems, PostgreSQL).

- _none yet — see [[domains/_index]]_

## Comparisons — `wiki/comparisons/`

Side-by-side analyses (e.g., OOR vs TypeORM vs Drizzle).

- _none yet — see [[comparisons/_index]]_

## Questions — `wiki/questions/`

Filed answers to user queries.

- _none yet — see [[questions/_index]]_

## Meta — `wiki/meta/`

Dashboards, lint reports, conventions.

- _none yet — see [[meta/_index]]_
