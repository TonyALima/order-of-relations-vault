---
type: overview
title: "Wiki Index"
created: 2026-04-29
updated: 2026-04-29
tags:
  - meta
  - index
status: developing
page_count: 16
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

- _none yet — see [[modules/_index]]_

## Components — `wiki/components/`

Reusable building blocks (decorators, query-builder fragments, container primitives).

- _none yet — see [[components/_index]]_

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

- _none yet — see [[flows/_index]]_

---

## Sources — `wiki/sources/`

One synthesis page per item in `.raw/`.

- [[sources/welcome|Welcome to the OOR Vault]] — manifesto: pillars + 7 ADR seeds (2026-04-29).
- _pending ingest: `architecture-overview.md`, `decorator-metadata-storage.md`, `query-builder-design.md`, `repository-contract.md` — see [[sources/_index]]._

## Entities — `wiki/entities/`

People, organizations, products, repos, libraries that show up in the work.

- [[Bun]] — runtime + package manager + test runner + bundler; the single toolchain.
- [[PostgreSQL]] — sole target database.
- [[TypeScript]] — host language; strict no-`any`.
- See [[entities/_index]] for the rolling index.

## Concepts — `wiki/concepts/`

Ideas, patterns, frameworks (e.g., Stage-3 decorators, repository pattern, lazy query builder).

- [[ECMAScript Stage-3 Decorators]] — the standardized decorator dialect.
- [[Repository Pattern]] — `Repository<T>` as persistence entry point.
- [[Lazy Query Builder]] — clauses accumulate; SQL runs only on terminal calls.
- [[Parameterized SQL]] — placeholders + bound parameters; the only allowed query form.
- [[Dependency Injection Container]] — minimal singleton container.
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
