---
type: component
title: "QueryBuilder"
component_kind: class
module: "src/query-builder/"
exports_as: "QueryBuilder"
created: 2026-04-29
updated: 2026-04-29
tags:
  - component
  - query-builder
  - core
status: stable
related:
  - "[[Lazy Query Builder]]"
  - "[[Conditions Proxy]]"
  - "[[Repository Pattern]]"
  - "[[sqlJoin]]"
  - "[[query-lifecycle]]"
  - "[[sources/query-builder-design]]"
sources:
  - "../.raw/query-builder-design.md"
---

# QueryBuilder

## Purpose

`QueryBuilder<T>` is the concrete class that composes a SQL `SELECT` against entity `T`. It is the layer that [[Repository Pattern|Repository]] reads delegate to, the home of the `where`-callback machinery, and the only place in OOR that turns runtime user input into a parameterized SQL statement targeting the wire.

For the *idea* of laziness, see [[Lazy Query Builder]]. For the *call signature* of `where`, see [[Conditions Proxy]]. This page documents the *class*: its state, its mutability stance, its terminal methods, and its current scope.

## Internal State

A single mutable field, today:

```ts
private conditions: Condition[] = [];
```

That's it. Where-conditions and the inheritance discriminator filter both reduce to entries in this array. Future fields (ordering, pagination) would each get their own.

The class is **mutable, not immutable-by-copy** — see § Mutability.

## Lifecycle

`QueryBuilder<T>` is a short-lived, single-owner object:

1. The repository constructs it: `new QueryBuilder<T>(...)`, passing the entity constructor and the `Database`.
2. `applyOptions(options?: FindOptions<T>): this` — the entry point repositories use to install the user's `where` callback (and any other future option). Returns `this` for chaining.
3. A terminal method (`getMany()` / `getOne()`) is called. SQL is composed, parameterized, awaited against [[Bun]]'s `SQL` driver, and rows return.
4. The builder is discarded.

There is no scenario today where two consumers hold the same builder and expect divergent state.

## `applyOptions()` — what it consumes

`FindOptions<T>` carries two fields today:

```ts
export interface FindOptions<T> {
  where?: (conditions: Conditions<T>) => (Condition | undefined)[];
  inheritance?: InheritanceSearchType;
}
```

### `where` — replace, not accumulate

Calling `applyOptions()` twice **replaces** the where-conditions wholesale (`this.conditions = results`). Deliberate "last call wins."

This may surprise consumers who expect classic builder semantics where every `.where()` call ANDs onto the previous. The current decision reflects that:

- The repository's `findMany(options?)` / `findOne(options?)` are the dominant entry points; they call `applyOptions` exactly once.
- Partial composition is not a use case today; introducing it implicitly via accumulation could mask bugs (forgotten reset, surprising AND with stale state).

See [[apply-options-accumulation]] — open question on whether to flip to additive.

### `inheritance` — discriminator-scoped reads

`InheritanceSearchType` is a closed enum with three values controlling how reads scope across a [[Single-Table Inheritance|single-table-inheritance]] hierarchy:

| Value | What `applyOptions` does | Emitted SQL |
|---|---|---|
| `ALL` (default) | Nothing — no branch in `applyOptions` for `ALL`. | None added; reads every row in the inherited table regardless of subclass. |
| `ONLY` | Calls `setConcreteClassDiscriminator()`. | Pushes `discriminator = <self>` onto `this.conditions`. |
| `SUBCLASSES` | Calls `setSubClassesDiscriminator()`. | Pushes `discriminator IN (...)` whose value list is `T` and every descendant in the prototype chain. |

The two helpers (`setSubClassesDiscriminator`, `setConcreteClassDiscriminator`) are method-private but actually-public in effect: they **push** onto `this.conditions` rather than replacing. The order in `applyOptions` is `where` first (replaces), then inheritance (pushes), so a single `applyOptions` call cleanly ANDs the discriminator predicate onto the user's conditions.

> [!key-insight] Discriminator-only-when-needed
> The discriminator filter is only added when `meta.discriminator` is truthy — and `MetadataStorage.resolveInheritance` *wipes* the discriminator on entities alone in their table (see [[Single-Table Inheritance]]). So `userRepo.findMany({ inheritance: ONLY })` against a `User` with no subclasses is a no-op at SQL level. As soon as a sibling registers and resolution re-runs, the same call starts emitting the predicate. The option's effect depends on whether siblings exist, not just on what the caller passes.

> [!warning] Footgun: double-applyOptions
> If `applyOptions` is called twice — once with `inheritance`, once with `where` — the second call's `where` clobbers the discriminator (because `where` *replaces*, and `inheritance` was on the first call only). Always pass both fields together in a single call.

Subclass discovery uses a pure prototype-chain walk against the live metadata `Map`:

```ts
for (const [t, m] of this.db.getMetadata()) {
  if (t === this.entity ||
      Object.prototype.isPrototypeOf.call(this.entity.prototype, t.prototype)) {
    subclasses.push(m);
  }
}
```

No name strings, no manual registry. See `examples/inheritance/services/UserHierarchyService.ts` for the canonical use of `SUBCLASSES`, and [[sources/drift-d3-find-options-inheritance]] for the full documentation gap this section closes.

## Terminal Methods

Two today:

| Method | Returns | Notes |
|---|---|---|
| `getMany()` | `Promise<T[]>` | Every matching row. Empty result is `[]`, never `null`. |
| `getOne()` | `Promise<T \| null>` | First row of `getMany()` or `null`. **Slices client-side** — does **not** emit `LIMIT 1` today. See [[get-one-limit-1]] for the open question. |

Notably absent and acknowledged as future work: `getCount()`, `getExists()`, streaming. The shape `getX(): Promise<X>` is the pattern; new terminals slot in without disturbing the clause API.

## Mutability

> [!key-insight] Mutable, not immutable-by-copy — by design
> The builder is a short-lived, single-owner object. There is no scenario today where two consumers hold the same builder and expect divergent state, so the safety that immutable-by-copy would buy is irrelevant — and the per-clause allocations would be paid for nothing. If a sharing use case appears later (e.g. caching a partial query), the plan is to introduce a `clone()` rather than retrofit copy-on-write into every method.

This refines an earlier framing on [[Lazy Query Builder]] that called the builder "conceptually immutable" — that was wrong. The builder is concretely mutable; the *deferred-execution* property is what makes the laziness work, not immutability.

## SQL Composition

`getMany()` is where the conditions array becomes SQL. The composition uses three safety mechanisms (see [[Parameterized SQL]] for the underlying property):

- **Operators come from a closed enum.** A static `opFragments` map maps each `Condition['op']` value to a pre-built `sql\`=\``-style fragment. The operator token is *never* built from user input.
- **Column names go through `sql(c.columnName)`** — Bun's identifier form. Safe even when the column metadata's `columnName` was derived from user code (decorator argument).
- **Values go through normal parameter binding.** `${value}` in the tagged template becomes a placeholder.
- **The fragment-joiner is `[[sqlJoin]]`** — never hand-rolled `reduce`.

The `IN` operator deserves a callout: ``sql`${col} IN ${sql(c.value)}` `` — `sql(array)` binds each element as a parameter. An **empty array** produces a valid `IN ()` that matches nothing rather than crashing. The test suite pins this.

## Scope

What `QueryBuilder<T>` does **not** do today, by deliberate scope:

- **No OR / nested boolean trees.** `where` is a flat AND list. A real boolean DSL is significant API surface; the source defers it until at least one concrete use case appears.
- **No `orderBy` / `limit` / `offset`.** Trivial to add as new fields + new clauses; held back because none of the current examples need them.
- **No joins.** Relations are loaded eagerly through repository helpers, not through builder-level joins.
- **No raw escape hatch (`qb.raw(...)`).** Reaffirms [[0004-parameterized-sql-only]] — the typed surface gets extended; the parameterized invariant is not weakened.

## Type Flow

`QueryBuilder<T>` carries `T` end-to-end:

- `Repository<T>` constructor takes `new () => T`.
- `findMany(options?: FindOptions<T>)` / `findOne(options?: FindOptions<T>)` → `new QueryBuilder<T>(...)`.
- `applyOptions(options?: FindOptions<T>): this` → preserves `T` for chaining.
- `getMany(): Promise<T[]>` / `getOne(): Promise<T | null>` → results land typed as `T`.

The mapped type that does the heavy lifting:

```ts
export type Conditions<T> = {
  [K in keyof T]?: FieldConditionBuilder<T[K]>;
};
```

The `?` is load-bearing — see [[Conditions Proxy]] for the rationale.

## Connections

- [[Lazy Query Builder]] — the concept this class realizes.
- [[Conditions Proxy]] — the typed object passed to `where`.
- [[Repository Pattern]] — the entry point that owns and discards the builder.
- [[sqlJoin]] — the only sanctioned fragment joiner.
- [[query-lifecycle]] — the per-call walkthrough.
- [[Parameterized SQL]] / [[0004-parameterized-sql-only]] — the safety property the builder must preserve.
- [[Layered Architecture]] — `QueryBuilder` is layer 4 in the five-layer stack.

## Open Questions

- [[get-one-limit-1]] — `getOne()` slicing vs. `LIMIT 1`.
- [[apply-options-accumulation]] — `applyOptions()` replace vs. accumulate.

## Sources

- `.raw/query-builder-design.md` (the pin-down for everything on this page)
- `.raw/architecture-overview.md` (the layer-4 placement and the source-tree note for `src/query-builder/`)
