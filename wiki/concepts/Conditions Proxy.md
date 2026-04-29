---
type: concept
title: "Conditions Proxy"
complexity: intermediate
domain: "Query building"
aliases:
  - "where callback proxy"
  - "FieldConditionBuilder proxy"
created: 2026-04-29
updated: 2026-04-29
tags:
  - concept
  - query-builder
  - typing
status: seed
related:
  - "[[Lazy Query Builder]]"
  - "[[query-lifecycle]]"
  - "[[sources/architecture-overview]]"
sources:
  - "../.raw/architecture-overview.md"
---

# Conditions Proxy

## Definition

The **conditions proxy** is the typed object passed to a `QueryBuilder<T>`'s `where` callback. It exposes one `FieldConditionBuilder` per column on `T`, each carrying the operators valid for that column's type. The user's callback consumes the proxy and returns an array of `Condition` objects.

```ts
userRepo.findMany({
  where: (u) => [u.email!.eq('a@b.com')],
});
```

Here `u` is the conditions proxy for `User`; `u.email` is a `FieldConditionBuilder<string>`; `.eq(...)` produces a `Condition`.

## How It Works

The query builder constructs the proxy from the entity's [[MetadataStorage|column metadata]]:

- For each `ColumnMetadata`, attach a `FieldConditionBuilder` keyed by the column property name.
- Each builder carries the operators valid for the column's type — `.eq`, `.neq`, `.in`, `.lt`, `.gt`, etc. — pre-tokenised from a fragment table.
- The proxy's *type* is derived from the entity class itself: TypeScript's mapped types take `T`'s columns and produce `{ [K in keyof Columns<T>]: FieldConditionBuilder<Columns<T>[K]> }`.

The user's `where` callback runs, returns an array, and the array contents are validated (see [[query-lifecycle]] step 4) before SQL composition.

## Why It Matters

- **Type-safe column references.** Misspelling a column name doesn't pass typecheck; the proxy doesn't expose properties that aren't columns.
- **Type-safe operators.** A `Date` column's `FieldConditionBuilder` exposes `.before` / `.after` but not `.matches`; the type system rejects the wrong operator at compile time.
- **No string concatenation.** Operators are pre-tokenised into SQL fragments; values stay as parameters. The proxy is the user-facing layer that makes [[Parameterized SQL]] enforceable in composed reads.
- **No `any`.** The whole construction relies on conditional / mapped types — exactly the pattern [[0005-no-any-type-driven-api]] commits to.

## Pitfalls

The non-null assertion (`u.email!`) is required when the column is typed as nullable on the entity (`email?: string`) but the query author knows the predicate should still apply. The proxy preserves nullability from the column metadata, so the assertion is a deliberate narrowing the user opts into.

If the user writes `u.foo?.eq(...)` against a column that doesn't exist (typo, refactor lag), `u.foo` is `undefined`, the optional-chained `.eq` returns `undefined`, and the condition slips into the array as `undefined`. **The query builder catches this in step 4** and throws `UndefinedWhereConditionError` rather than silently dropping the predicate.

## Examples

```ts
// Equality
userRepo.findMany({ where: (u) => [u.email!.eq('a@b.com')] });

// Multiple conditions (AND)
userRepo.findMany({
  where: (u) => [
    u.active!.eq(true),
    u.createdAt!.after(new Date('2026-01-01')),
  ],
});
```

## Connections

- [[Lazy Query Builder]] — owns the proxy and consumes the returned `Condition[]`.
- [[query-lifecycle]] — step 3 builds the proxy, step 4 validates the returned array.
- [[Parameterized SQL]] — the safety property the proxy preserves.
- [[0005-no-any-type-driven-api]] — the strictness ADR this construction depends on.

## Sources

- `.raw/architecture-overview.md` §§ "Lifecycle of a Query" (steps 3–4)
