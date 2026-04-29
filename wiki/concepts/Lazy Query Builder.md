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

The builder is **mutable**, single-owner, and short-lived. Today it holds exactly one piece of internal state:

```ts
private conditions: Condition[] = [];
```

That's it. Where-conditions and the inheritance discriminator filter both reduce to entries in this array. Future fields (ordering, pagination) would each get their own dedicated property on the class.

The repository constructs a `QueryBuilder<T>`, calls `applyOptions(options?)` to install the user's `where` callback (and any other future option), then calls a terminal method. The builder is discarded afterwards. There is no scenario today where two consumers hold the same builder.

`applyOptions()` **replaces** the where-conditions wholesale on every call — last-call-wins, not additive. See [[apply-options-accumulation]] for the open question on whether to flip this to accumulation.

The terminal methods today are exactly two:

- `getMany(): Promise<T[]>` — every matching row. Empty result is `[]`, never `null`.
- `getOne(): Promise<T | null>` — first row of `getMany()` or `null`. **Slices client-side**; does not emit `LIMIT 1` today (see [[get-one-limit-1]]).

Notably absent: `getCount()`, `getExists()`, streaming. Future work — the shape `getX(): Promise<X>` is the pattern; new terminals slot in without disturbing the clause API.

Because SQL only runs at the terminal call, intermediate builders can be passed to helpers that conditionally add inheritance scope or extra conditions, and the helper doesn't have to re-issue or accept already-fetched rows. *That* is what laziness buys, regardless of the mutability stance.

> [!note] Refined 2026-04-29 (×2)
> 1. Earlier framing called the builder "conceptually immutable from the consumer's perspective." That was wrong — per `.raw/query-builder-design.md`, the builder is **concretely mutable**. The deferred-execution property (laziness) is what makes composition safe, not immutability. See [[QueryBuilder]] § Mutability for the rationale.
> 2. The terminal-methods list previously included `getCount()`. The source explicitly lists `getCount`, `getExists`, and streaming as **future work, not yet present**. Removed.

## Why It Matters

- **Composability**: filters, scopes, and "base queries" can be expressed as functions over `QueryBuilder<T>`. Without laziness, every layer would have to re-execute or cache rows.
- **Type narrowing**: each method can refine the result type. `select(["id", "name"])` can narrow `T` to `Pick<T, "id" | "name">` at the type level.
- **Predictability**: SQL runs at exactly one site (the terminal call). No surprise queries fired by `.toString()`-style accidents.

## Examples

Actual API per `.raw/architecture-overview.md` — `where` is a callback that receives a typed [[Conditions Proxy|conditions proxy]], not a plain object:

```ts
// `findMany` is a thin Repository wrapper that constructs a QueryBuilder,
// applies options, and calls getMany() internally. SQL fires once, on getMany.
const active = await userRepository.findMany({
  where: (u) => [u.active!.eq(true), u.deletedAt!.eq(null)],
});
```

> [!note] Refined 2026-04-29
> An earlier version of this page showed `where({ active: true })` taking a plain object. The actual API is a callback `(u) => [u.active!.eq(true)]` returning an array of `Condition` objects produced from a typed proxy of `FieldConditionBuilder`s. See [[Conditions Proxy]] for the proxy mechanism and [[query-lifecycle]] step 3 for where the proxy is built.

When composition / layering is needed, the builder returned by `Repository.find()` (when present in the API surface) accumulates the same way: each chained call appends state, terminal methods (`getMany`, `getOne`, `getCount`) compose and execute. The lazy-execution property is the same regardless of whether you reached the builder via `find()` or via a wrapper like `findMany`.

## Connections

- [[0002-repository-with-lazy-query-builder]] — the ADR.
- [[QueryBuilder]] — the concrete class realizing this concept, with the actual state field, mutability stance, and terminal methods.
- [[Repository Pattern]] — the entry point that wraps and delegates to this type.
- [[Conditions Proxy]] — the typed object the `where` callback receives.
- [[query-lifecycle]] — the six-step walkthrough of a `findMany` call.
- [[sqlJoin]] — the only sanctioned helper for joining SQL fragments at terminal time.
- [[Parameterized SQL]] — the safety property the builder must preserve at terminal compile time.

## Open Questions

- [[get-one-limit-1]] — `getOne()` slicing vs `LIMIT 1`.
- [[apply-options-accumulation]] — `applyOptions()` replace vs accumulate.

## Sources

- `.raw/welcome.md` § "Repository pattern with a lazy query builder"
