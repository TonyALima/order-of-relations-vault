---
type: concept
title: "Lazy Query Builder"
complexity: intermediate
domain: "ORM design"
aliases:
  - "QueryBuilder"
  - "QueryBuilder<T>"
  - "Deferred query"
created: 2026-04-29
updated: 2026-04-29
tags:
  - concept
  - query-builder
  - orm
status: seed
related:
  - "[[0002-repository-with-lazy-query-builder]]"
  - "[[Repository Pattern]]"
  - "[[sources/welcome]]"
sources:
  - "../.raw/welcome.md"
---

# Lazy Query Builder

## Definition

A **lazy query builder** accumulates query state — `WHERE` predicates, joins, ordering, pagination — without executing SQL. SQL is generated and run only when a terminal method is called. In OOR the type is `QueryBuilder<T>`, returned by `Repository<T>.find()`.

"Lazy" here means *deferred until a terminal call*, not "evaluated only on demand by a consumer of the rows." The rows are fetched eagerly once the terminal method runs; what is deferred is the entire act of running.

## How It Works

The builder holds an internal state object:

```ts
type QueryState = {
  wheres: Predicate[];
  joins: Join[];
  orderBy?: { column: string; direction: "ASC" | "DESC" };
  limit?: number;
  offset?: number;
};
```

Each chained method (`where`, `andWhere`, `join`, `orderBy`, `limit`, `offset`) returns a `QueryBuilder<T>` (or a narrowed type) with the new state appended. The builder is conceptually immutable from the consumer's perspective: chaining produces a new state.

Terminal methods compile the state to a parameterized SQL string + parameter array, hand it to the [[Bun]] `SQL` driver, and resolve with rows:

- `getMany(): Promise<T[]>`
- `getOne(): Promise<T | null>`
- `getCount(): Promise<number>`

Because SQL only runs at the terminal call, intermediate builders can be passed around and layered on. A function can accept a `QueryBuilder<User>`, add `where({ active: true })`, and return it without ever having issued a query.

## Why It Matters

- **Composability**: filters, scopes, and "base queries" can be expressed as functions over `QueryBuilder<T>`. Without laziness, every layer would have to re-execute or cache rows.
- **Type narrowing**: each method can refine the result type. `select(["id", "name"])` can narrow `T` to `Pick<T, "id" | "name">` at the type level.
- **Predictability**: SQL runs at exactly one site (the terminal call). No surprise queries fired by `.toString()`-style accidents.

## Examples

```ts
// Build, then layer, then run.
const baseQuery = userRepo.find().where({ deletedAt: null });

const activeQuery = baseQuery.where({ active: true }).orderBy("createdAt", "DESC");

const recent = await activeQuery.limit(10).getMany(); // <-- SQL fires here
const total = await activeQuery.getCount();           // <-- and here, separately
```

## Connections

- [[0002-repository-with-lazy-query-builder]] — the ADR.
- [[Repository Pattern]] — the entry point that returns this type.
- [[Parameterized SQL]] — the safety property the builder must preserve at terminal compile time.
- [[Query Builder Design]] — clause accumulation, lazy execution, type narrowing details (page populated by a future ingest of `.raw/query-builder-design.md`).

## Sources

- `.raw/welcome.md` § "Repository pattern with a lazy query builder"
