---
type: domain
title: "Concepts"
subdomain_of: ""
page_count: 7
created: 2026-04-29
updated: 2026-04-29
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
- [[Repository Pattern]] — `Repository<T>` as the per-entity persistence entry point; reads delegate to `QueryBuilder`.
- [[Lazy Query Builder]] — `QueryBuilder<T>` accumulates clauses; SQL runs only on terminal calls.
- [[Conditions Proxy]] — the typed proxy of `FieldConditionBuilder`s passed to a `where` callback.
- [[Parameterized SQL]] — placeholders + bound parameters; the only allowed query form (no `sql.unsafe`).
- [[Dependency Injection Container]] — minimal singleton container for `@Service` / `@InjectRepository` wiring (planned, not yet implemented).

> [!gap] Still to come
> Deep-dive concepts on `Metadata storage layout`, `Schema migrations`, `Type narrowing`, `Class-table inheritance` — populated as the remaining `.raw/` notes are ingested.
