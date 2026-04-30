---
type: domain
title: "Concepts"
subdomain_of: ""
page_count: 13
created: 2026-04-29
updated: 2026-04-30
tags:
  - meta
  - concepts
status: developing
related:
  - "[[index]]"
sources: []
---

# Concepts

Ideas, patterns, and frameworks. A concept page defines what something is and why it matters. Distinct from `components/` — a concept is the *idea*, a component is the *named implementation*.

## Concept pages

- [[Layered Architecture]] — the five-layer stack and the downward-only dependency rule.
- [[ECMAScript Stage-3 Decorators]] — the standardized decorator dialect OOR uses; replaces `experimentalDecorators` + `reflect-metadata`.
- [[Single-Table Inheritance]] — discriminator-based inheritance resolved lazily on first read of `MetadataStorage`.
- [[Relation Target Thunk]] — `() => User` closures defer constructor lookup past the temporal dead zone.
- [[Repository Pattern]] — `Repository<T>` as the per-entity persistence entry point; reads delegate to `QueryBuilder`.
- [[Lazy Query Builder]] — `QueryBuilder<T>` accumulates clauses; SQL runs only on terminal calls.
- [[Conditions Proxy]] — the typed proxy of `FieldConditionBuilder`s passed to a `where` callback.
- [[Parameterized SQL]] — placeholders + bound parameters; the only allowed query form (no `sql.unsafe`).
- [[sqlJoin]] — the only sanctioned helper for joining SQL fragments; replaces hand-rolled `reduce` patterns.
- [[Autogeneration]] — explicit-only PK auto-generation with `clientSide` / `dbSide` strategies; commit `3aa354b` removed implicit SERIAL.
- [[PrimaryKey Brand]] — `PrimaryKey<V>` structural brand; brand asymmetry keeps call sites brand-free; powers compile-time PK enforcement on `findById` / `delete` / `update`.
- [[Schema Migrations]] — *seed* — placeholder concept page; no migration system implemented yet. Lists the load-bearing design choices a future implementation would face.
- [[Dependency Injection Container]] — minimal singleton container for `@Service` / `@InjectRepository` wiring (planned, not yet implemented).

> [!gap] Still to come
> Deeper concepts on `Type narrowing in the query builder`, and `Class-table inheritance details` — populated when future sources or code spot-checks land.
