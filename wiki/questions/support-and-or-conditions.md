---
type: question
title: "Should QueryBuilder support AND / OR / nested boolean conditions?"
question: "Extend the where callback from a flat AND list to a real boolean tree (AND / OR / NOT, nested groups)?"
answer_quality: open
created: 2026-04-30
updated: 2026-04-30
tags:
  - question
  - open
  - query-builder
  - api-design
status: open
impact: high
effort: L
decided_by: ""
related:
  - "[[QueryBuilder]]"
  - "[[Conditions Proxy]]"
  - "[[Lazy Query Builder]]"
  - "[[query-lifecycle]]"
  - "[[Parameterized SQL]]"
sources:
  - "../.raw/query-builder-design.md"
  - "../.raw/architecture-overview.md"
---

# Should `QueryBuilder` support AND / OR / nested boolean conditions?

> [!note] Status: **open** (deferred by design — explicitly listed in [[QueryBuilder]] § Scope)

## Question

Today, the `where` callback returns a flat `(Condition | undefined)[]` and the SQL composer joins those entries with `AND`. There is no way to express:

```ts
// "active users with email ending in @paggo.ai, OR any admin"
userRepo.findMany({
  where: (u) => [
    /* (u.active.eq(true) AND u.email.like('%@paggo.ai')) OR u.role.eq('admin') */
  ],
});
```

Should OOR extend the builder to support real boolean trees — `AND`, `OR`, and (probably) `NOT` over nested groups of conditions?

## Why it matters

- **Real applications hit this on day one.** "Customers in São Paulo OR with > R$10k in lifetime spend" is a routine reporting query. Today the only path is to drop into raw SQL — but OOR has no `qb.raw(...)` escape hatch (see [[0004-parameterized-sql-only]]) and is firmly committed to keeping it that way. Without OR, users would either run two queries and merge in JS (a correctness hazard for ordering / dedup / pagination) or abandon the builder for direct `Database.query()` calls, which loses the type-safe identifier and value handling [[Conditions Proxy]] guarantees.
- **AND-only is an asymmetric API.** The "list = AND" convention is intuitive but lossy: it encodes *one* boolean operator structurally and forecloses the other. Users coming from TypeORM, Prisma, or Drizzle expect either an explicit `or:` field or `Op.or` / `Or(...)` combinators — see [[oor-vs-typeorm]], [[oor-vs-prisma]], [[oor-vs-drizzle]] for what the field has converged on.
- **STI discriminator coexistence.** [[Single-Table Inheritance|STI]] reads currently push a discriminator predicate that ANDs onto user conditions (see [[QueryBuilder]] § `inheritance`). Once OR exists at user level, the question becomes: does the discriminator filter wrap the user predicate as `(user_predicate) AND discriminator = ...`, or does it become a sibling node? The current "push onto array" trick stops working as soon as the array represents a tree.
- **Contribution claim, not gap.** [[oor-vs-typeorm]], [[oor-vs-prisma]], and [[oor-vs-drizzle]] currently lean on type-safe identifier handling and the no-`unsafe` invariant. A scoped boolean-tree design — preserving both — strengthens that argument; a permanent AND-only surface eventually undermines it.

## Current behavior (so the question doesn't decay)

From [[QueryBuilder]] § Scope:

> **No OR / nested boolean trees.** `where` is a flat AND list. A real boolean DSL is significant API surface; the source defers it until at least one concrete use case appears.

Mechanics today:

- `where: (conditions: Conditions<T>) => (Condition | undefined)[]` — the callback returns a flat array.
- `applyOptions()` runs the callback, validates each entry is non-`undefined` (rejects with `UndefinedWhereConditionError` carrying the offending index — see [[Conditions Proxy]] § "What goes wrong at runtime"), and assigns the result to `this.conditions: Condition[]`.
- `getMany()` walks `this.conditions` and joins fragments with the AND token via [[sqlJoin]], using `opFragments` as the closed-enum operator map and `sql(c.columnName)` for safe identifier interpolation.

Implementation pointers (verify before changing):

- `../order-of-relations/src/query-builder/query-builder.ts` — `applyOptions`, `getMany`, the `Condition` type, and the `opFragments` map all live here today.

## Sketch of the design space

Three candidate API shapes. None is endorsed; the goal of this section is to make the trade-offs visible so a future ADR can choose deliberately.

### Option A — `Or` / `And` combinator helpers (TypeORM / Sequelize-flavoured)

Expose combinators that produce composite `Condition` nodes:

```ts
import { Or, And, Not } from '@order-of-relations/core';

userRepo.findMany({
  where: (u) => [
    Or(
      And(u.active!.eq(true), u.email!.like('%@paggo.ai')),
      u.role!.eq('admin'),
    ),
  ],
});
```

**Pros:**

- Smallest type-shape disturbance: `where` still returns `(Condition | undefined)[]`. The array's outer level remains AND; combinators introduce inner trees.
- Composes naturally — `Or(...conds)`, `And(...conds)` accept variadic children, including other combinator results.
- Plays well with the existing `UndefinedWhereConditionError` validator: each combinator is a single `Condition` node from the validator's perspective.

**Cons:**

- Imported helpers live outside the proxy, so the type-safety guarantee comes from `Condition` being well-typed at construction, not from the proxy itself. Acceptable, but worth pinning in tests.
- `Not` is a unary wrapper that needs separate composition rules in SQL emission (parenthesisation around the negated subtree).

### Option B — Object-shaped condition tree (Prisma-flavoured)

Replace the flat `(Condition | undefined)[]` with a recursive object structure:

```ts
userRepo.findMany({
  where: {
    OR: [
      { AND: [{ active: { eq: true } }, { email: { like: '%@paggo.ai' } }] },
      { role: { eq: 'admin' } },
    ],
  },
});
```

**Pros:**

- No callback indirection — the where clause is a serializable value, friendly to programmatic construction (e.g., building queries from URL parameters).
- Matches what Prisma / Drizzle have proven users can hold in their head.

**Cons:**

- Replaces the entire [[Conditions Proxy]] mechanism, not extends it. The current callback shape buys compile-time column-name validation that an object literal can't match without giving up structural typing of values. Recovering that type safety in object-shaped APIs requires deep mapped-type machinery; see what Prisma's generator produces.
- Semantically diverges from the current "where is a flat AND list" convention rather than generalising it. Higher migration cost for any future code already written against today's API.

### Option C — Method-chaining builder (TypeORM `.where().andWhere().orWhere()` flavour)

Expose `qb.andWhere(callback)` / `qb.orWhere(callback)` on the lazy builder:

```ts
userRepo.findMany()
  .where((u) => [u.active!.eq(true), u.email!.like('%@paggo.ai')])
  .orWhere((u) => [u.role!.eq('admin')])
  .getMany();
```

**Pros:**

- Mutational, fits the existing builder mutability stance ([[QueryBuilder]] § Mutability).

**Cons:**

- Order-dependent precedence is a footgun: `where(A).orWhere(B).andWhere(C)` is ambiguous between `(A OR B) AND C` and `A OR (B AND C)`. TypeORM's resolution is "left-to-right", which is exactly the surprising-at-PR-review pattern the [[decorator-order-independence]] discussion flagged elsewhere.
- Introduces multiple `applyOptions`-shaped paths, conflicting with the "applyOptions replaces wholesale" decision (see [[apply-options-accumulation]]). Two opens questions become one larger one.

## What would change in the codebase

Approximate change surface for **Option A** (the lightest-touch path):

- `src/query-builder/condition.ts` (or wherever `Condition` lives) — extend the `Condition` type from a flat `{ columnName, op, value }` to a discriminated union: `{ kind: 'leaf', ... } | { kind: 'and' | 'or', children: Condition[] } | { kind: 'not', child: Condition }`.
- `src/query-builder/combinators.ts` — new module exporting `And`, `Or`, `Not`. Each returns a `Condition` of the corresponding `kind`.
- `src/query-builder/query-builder.ts` — `getMany()`'s SQL composition becomes recursive: walk the tree, parenthesise on group nodes, use [[sqlJoin]] with `' AND '` / `' OR '` per node, prefix `NOT ` on negation nodes. The `opFragments` table stays as-is — it covers leaves only.
- `src/query-builder/query-builder.ts` — STI discriminator push (`setConcreteClassDiscriminator` / `setSubClassesDiscriminator`) changes from "append to flat array" to "wrap user tree in `And(userTree, discriminatorLeaf)`". This is the [[QueryBuilder]] § `inheritance` interaction noted above.
- Tests — multiple parens layouts pinned: simple OR of two leaves, AND-of-OR, OR-of-AND, NOT around a group, and the combination with `inheritance: ONLY` / `SUBCLASSES`.
- `wiki/components/QueryBuilder.md` § Scope — flip the "No OR / nested boolean trees" line.
- `wiki/concepts/Conditions Proxy.md` § Examples — add a multi-condition OR example, document combinators alongside the per-field methods.
- `wiki/concepts/Lazy Query Builder.md` — note the tree shape so the laziness explanation matches.

## Things to verify before deciding

- **Concrete user demand.** The source defers this *"until at least one concrete use case appears."* Has it? A search of `examples/` and the OOR npm consumer surface (issues, draft examples) for OR-shaped queries should precede the design call.
- **Prior art mapping.** Quick read of TypeORM's `Or` / `Brackets`, Prisma's `OR` / `AND` / `NOT`, and Drizzle's `or()` / `and()`. The combinator pattern (Option A) is the convergent answer in libraries that started typed-callback; the object pattern (Option B) is convergent in schema-DSL libraries.
- **NOT semantics.** `NOT` is the cheap part syntactically but the expensive part for null-handling: in SQL, `NOT (x = 5)` and `x != 5` differ when `x IS NULL`. Do we expose `Not` directly, or only sugar like `notIn` / `notEq` (already partially there as `ne`)? See [[Conditions Proxy]] for the operator inventory today.
- **Validator update.** `UndefinedWhereConditionError` carries the *index* in the flat array. Once the array represents a tree, "index" needs a richer locator (path? dot-notation? `[0].children[1]`?). The error UX is part of the API.
- **Type-checking the tree.** Each combinator should preserve `T` end-to-end so misuse like `Or(u.email!.eq('a'), o.total!.gt(0))` (mixing entity proxies) is rejected at compile time. The typing exercise mirrors [[Conditions Proxy]]'s mapped-type approach but for combinators.

## What would close this question

A decision in one of three directions:

- **"Option A — combinators."** File an ADR locking the surface (`Or`, `And`, `Not`), the `Condition` discriminated union, and the STI-tree wrapping rule. Implement.
- **"Option B — object tree."** A more invasive ADR; would also need to address how `Conditions<T>` mapped-type becomes a `Where<T>` mapped-type with the same column-name guarantees.
- **"Hold."** Document why a flat AND list is sufficient for the targeted use cases — i.e., affirm the deferral as the long-term answer rather than a transient one.

## Confidence

**Open** — the source explicitly defers this; no decision logged. Higher impact than [[get-one-limit-1]] / [[apply-options-accumulation]] because the absence is structural, not a performance tweak.

## Related Questions

- [[apply-options-accumulation]] — the accumulator-vs-replace question becomes coupled if Option C is ever entertained. Otherwise independent.

## Sources

- `.raw/query-builder-design.md` § "Trade-offs and open questions" (deferral statement)
- `.raw/architecture-overview.md` § "Lifecycle of a Query"
- `../order-of-relations/src/query-builder/query-builder.ts` (current AND-only implementation, verified 2026-04-30)
