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

The **conditions proxy** is the typed object passed to a `QueryBuilder<T>`'s `where` callback. It exposes one `FieldConditionBuilder` per column on `T`, each carrying the operators valid for that column's type. The user's callback consumes the proxy and returns an array of `Condition` (or `undefined`) entries.

The exact callback signature:

```ts
where: (conditions: Conditions<T>) => (Condition | undefined)[]
```

And the proxy type:

```ts
export type Conditions<T> = {
  [K in keyof T]?: FieldConditionBuilder<T[K]>;
};
```

A `Partial` mapped type over the entity's keys. Each key maps to a per-field builder parameterized by the *field's* type — `u.age.gt(18)` accepts `number`, `u.name.eq('Alice')` accepts `string`, and a typo on the property name is a compile error.

```ts
userRepo.findMany({
  where: (u) => [u.email!.eq('a@b.com')],
});
```

Here `u` is the conditions proxy for `User`; `u.email` is a `FieldConditionBuilder<string>`; `.eq(...)` produces a `Condition`.

## How It Works

The query builder constructs the proxy from the entity's [[MetadataStorage|column metadata]]:

- For each `ColumnMetadata`, attach a `FieldConditionBuilder` keyed by the column property name.
- Each `FieldConditionBuilder<V>` exposes **the same nine methods regardless of `V`**:
  - **Comparison:** `eq(v)`, `ne(v)`, `gt(v)`, `gte(v)`, `lt(v)`, `lte(v)` — produce `=`, `!=`, `>`, `>=`, `<`, `<=` respectively.
  - **Null tests:** `isNull()`, `isNotNull()` — produce `IS NULL` / `IS NOT NULL`.
  - **Set membership:** `in(values: V[])` — produces `IN (...)`. An empty array is a valid no-match `IN ()`; the test suite pins this.
- The proxy's *type* is derived from the entity class itself: TypeScript's mapped types take `T`'s columns and produce `{ [K in keyof T]?: FieldConditionBuilder<T[K]> }`.

The user's `where` callback runs, returns an array, and the array contents are validated (see [[query-lifecycle]] step 4) before SQL composition.

## Why It Matters

- **Type-safe column references.** Misspelling a column name doesn't pass typecheck; the proxy doesn't expose properties that aren't columns.
- **Type-safe values, uniform operator surface.** The compile-time guarantee is on the **value parameter** — `gt(v: V)` rejects a value whose type doesn't match `V`. There is **no per-column-type narrowing on operator availability**: a `Date` column and a `number` column expose the same nine methods. (A future API could narrow further — e.g. exposing string-specific `like` or date-specific `between` only on the right column types — but today the surface is uniform.)
- **No string concatenation.** Operators are pre-tokenised into SQL fragments; values stay as parameters. The proxy is the user-facing layer that makes [[Parameterized SQL]] enforceable in composed reads.
- **No `any`.** The whole construction relies on conditional / mapped types — exactly the pattern [[0005-no-any-type-driven-api]] commits to.

## Pitfalls

### Why the mapped type is `Partial`

The `?` on `Conditions<T>` is **load-bearing**, not stylistic.

It exists so that `u.name?.eq(...)` calls type-check against entities where some properties live on a base class, are inherited via single-table inheritance, or are simply optional in the type but might not exist in the metadata. TypeScript stays quiet; the runtime layer (the proxy + the `UndefinedWhereConditionError`) covers the gap between *"TypeScript thinks this might be undefined"* and *"actually, the column doesn't exist."*

This is the inverse of the decorator-order constraint pattern: there, the runtime catches what the compiler couldn't reject; here, the runtime catches what the compiler **deliberately** allowed.

### What goes wrong at runtime

If the user writes `u.foo?.eq(...)` against a column that doesn't exist (typo, refactor lag), `u.foo` is `undefined`, the optional-chained `.eq` returns `undefined`, and the condition slips into the array as `undefined`. **The query builder catches this in [[query-lifecycle]] step 4** and throws `UndefinedWhereConditionError`, carrying the **offending index** (so the user can immediately locate the bad entry in their `where` array) rather than silently dropping the predicate.

### The non-null assertion (`u.email!`)

The non-null assertion is required when the column is typed as nullable on the entity (`email?: string`) but the query author knows the predicate should still apply. The proxy preserves nullability from the column metadata, so the assertion is a deliberate narrowing the user opts into.

## Examples

```ts
// Equality
userRepo.findMany({ where: (u) => [u.email!.eq('a@b.com')] });

// Multiple conditions (AND)
userRepo.findMany({
  where: (u) => [
    u.active!.eq(true),
    u.createdAt!.gt(new Date('2026-01-01')),
  ],
});
```

> [!note] Refined 2026-04-29 (drift D4)
> An earlier version of this page listed `.neq` (the actual method is `ne`) and `.before` / `.after` / `.matches` (none exist) as if they were per-column-type narrowed operators. The real surface is the nine uniform methods documented above; the `.raw/query-builder-design.md` source had it right and the synthesis introduced the drift. See [[sources/drift-d4-conditions-proxy-operators]].

## Connections

- [[Lazy Query Builder]] — owns the proxy and consumes the returned `(Condition | undefined)[]`.
- [[QueryBuilder]] — the concrete class that builds and validates the proxy at runtime.
- [[query-lifecycle]] — step 3 builds the proxy, step 4 validates the returned array (rejects `undefined` with the offending index).
- [[Parameterized SQL]] — the safety property the proxy preserves.
- [[0005-no-any-type-driven-api]] — the strictness ADR this construction depends on.

## Sources

- `.raw/architecture-overview.md` §§ "Lifecycle of a Query" (steps 3–4)
