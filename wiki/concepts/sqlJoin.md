---
type: concept
title: "sqlJoin"
complexity: foundational
domain: "SQL composition"
aliases:
  - "sql-join"
  - "fragment joiner"
created: 2026-04-29
updated: 2026-04-29
tags:
  - concept
  - sql
  - utility
  - core
status: stable
related:
  - "[[Parameterized SQL]]"
  - "[[QueryBuilder]]"
  - "[[Repository Pattern]]"
  - "[[sources/query-builder-design]]"
sources:
  - "../.raw/query-builder-design.md"
  - "../.raw/architecture-overview.md"
---

# sqlJoin

## Definition

`sqlJoin` is the **only sanctioned way** in the OOR codebase to combine an array of SQL fragments into a single fragment with a separator. It lives in `src/core/utils/` and is used by every site that composes SQL out of pieces — `Repository.create`, `Repository.update`, `Repository.delete`, and `QueryBuilder.getMany`.

```ts
sqlJoin({ sql, items, map, separator })
```

Where `items` is the array, `map` produces a fragment per element, and `separator` defaults to `, ` (often overridden to `AND` for `WHERE` clauses).

## How It Works

`sqlJoin` returns a single tagged-template fragment that:

1. Maps each `item` to its own fragment via `map(item)`.
2. Joins the fragments with `separator` between them.
3. Preserves all parameter bindings — the underlying parameterization survives the join.

The function is intentionally small. Its value isn't the implementation; it's that it **exists at all** as the single sanctioned join helper.

## Why It Matters

The pattern this replaces — hand-rolled `reduce` over an array of fragments — is documented as a known footgun:

- Easy to skip a separator (the head element is special; people reach for `if (i === 0)` and get it wrong).
- Easy to drop the head element (off-by-one in the accumulator).
- Easy to mess up parameter ordering (concatenating raw strings instead of fragments).

`sqlJoin` exists precisely so that "hand-roll a reduce" never has to be a code-review judgement call. **Every place in the codebase that needs to join SQL fragments uses it** — that's an enforceable property at PR review.

It also reinforces [[0004-parameterized-sql-only]] from a different angle. The ADR forbids `sql.unsafe`; `sqlJoin` removes the *temptation* to reach for unsafe string-level operations when a join is needed.

## Examples

The canonical use, inside `QueryBuilder.getMany()` for a `WHERE` clause:

```ts
sqlJoin({
  sql,
  items: this.conditions,
  map: (c) => sql`${sql(c.columnName)} ${opFragments[c.op]} ${c.value}`,
  separator: sql` AND `,
})
```

In `Repository.create()`, joining the column list and the values list is two `sqlJoin` calls — one for `(col1, col2, col3)`, another for `($1, $2, $3)`.

## Connections

- [[Parameterized SQL]] — the safety property `sqlJoin` preserves under composition.
- [[0004-parameterized-sql-only]] — the codebase-wide ADR that `sqlJoin` operationalizes for "joining" specifically.
- [[QueryBuilder]] — the heaviest user of `sqlJoin` (every `WHERE` clause goes through it).
- [[Repository Pattern]] — `create`, `update`, `delete` all use it for column lists, value lists, and `SET` clauses.
- [[Layered Architecture]] — `sqlJoin` lives in the `src/core/utils/` shared layer.

## Sources

- `.raw/query-builder-design.md` § "SQL composition safety"
- `.raw/architecture-overview.md` § "Source Tree, by Concern" (`src/core/utils/`)
