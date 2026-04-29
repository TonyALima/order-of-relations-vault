---
type: decision
title: "ADR 0002 — Repository as the entry point, lazy QueryBuilder underneath"
status: accepted
date: 2026-04-29
deciders:
  - "Tony Albert"
supersedes: []
superseded_by: ""
context: "Need a clean boundary between simple persistence calls and composed queries."
created: 2026-04-29
updated: 2026-04-29
tags:
  - decision
  - adr
  - repository
  - query-builder
related:
  - "[[Repository Pattern]]"
  - "[[Lazy Query Builder]]"
  - "[[sources/welcome]]"
sources:
  - "../.raw/welcome.md"
---

# ADR 0002 — Repository as the entry point, lazy QueryBuilder underneath

## Context

An ORM has to expose two very different ergonomics:

- **Trivial cases** — `repo.findOne({ id })`, `repo.create({ ... })` — should read like a typed key/value store.
- **Composed cases** — joins, dynamic filters, ordering, pagination — should read like SQL but typed.

If a single API tries to cover both, it bloats. If two APIs are exposed at the same level, consumers don't know which to reach for.

## Decision

**`Repository<T>` is the single entry point for an entity's persistence.** Simple operations (`findOne`, `create`, `update`, `delete`) execute SQL directly. The composed entry point — `Repository.find()` — does **not** execute SQL; it returns a `QueryBuilder<T>` that accumulates clauses lazily and runs only on terminal calls (`getMany()`, `getOne()`, `getCount()`, …).

## Consequences

### Positive

- The mental model is one-directional: simple methods on `Repository`, composition on `QueryBuilder`. Consumers always know where they are.
- Lazy execution makes the builder safely composable — you can pass a half-built query around, layer additional `where()` calls on top, and only commit to running it at the call site that knows it's complete.
- Decouples the read-vs-write surface naturally: writes go through `Repository`, reads scale up through `QueryBuilder`.

### Negative / trade-offs

- Two types to learn. Consumers must understand that `find()` returns *a builder, not a result*.
- Risk of footgun: a developer who forgets the terminal call gets a `QueryBuilder` instead of a row array. Type-narrowing in the API helps, but not perfectly.

### Neutral

- The boundary forces a discipline: anything that needs to execute SQL implicitly belongs on `Repository`; anything that should compose lives on `QueryBuilder`.

## Alternatives Considered

- **Single-class fluent API (TypeORM-style `Repository.createQueryBuilder()`)** — rejected: muddies the boundary; simple and composed cases share too much surface.
- **Eager execution from `find()`** — rejected: can't compose, and forces every filter to be passed up-front.
- **No repositories, raw query builder only (Drizzle-style)** — rejected: trivial CRUD becomes verbose.

## References

- `.raw/welcome.md` § "Repository pattern with a lazy query builder"
- [[Repository Pattern]]
- [[Lazy Query Builder]]
