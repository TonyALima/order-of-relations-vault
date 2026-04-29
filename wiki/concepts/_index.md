---
type: domain
title: "Concepts"
subdomain_of: ""
page_count: 5
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

- [[ECMAScript Stage-3 Decorators]] — the standardized decorator dialect OOR uses; replaces `experimentalDecorators` + `reflect-metadata`.
- [[Repository Pattern]] — `Repository<T>` as the per-entity persistence entry point.
- [[Lazy Query Builder]] — `QueryBuilder<T>` accumulates clauses; SQL runs only on terminal calls.
- [[Parameterized SQL]] — placeholders + bound parameters; the only allowed query form (no `sql.unsafe`).
- [[Dependency Injection Container]] — minimal singleton container for `@Service` / `@InjectRepository` wiring.

> [!gap] Still to come
> `Metadata storage`, `Schema migrations`, `Type narrowing in the query builder`, `TDD` — populated as the remaining `.raw/` notes are ingested.
