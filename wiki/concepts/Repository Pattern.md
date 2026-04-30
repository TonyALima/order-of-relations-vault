---
type: concept
title: "Repository Pattern"
complexity: intermediate
domain: "ORM design"
aliases:
  - "Repository"
  - "Repository<T>"
created: 2026-04-29
updated: 2026-04-30
tags:
  - concept
  - persistence
  - orm
status: seed
related:
  - "[[0002-repository-with-lazy-query-builder]]"
  - "[[Lazy Query Builder]]"
  - "[[PrimaryKey Brand]]"
  - "[[sources/welcome]]"
  - "[[sources/pk-aware-repository-methods]]"
sources:
  - "../.raw/welcome.md"
  - "../.raw/pk-aware-repository-methods.md"
---

# Repository Pattern

## Definition

A **Repository** is the single object responsible for persistence of one entity type. It mediates between the domain (typed entity classes) and the storage (PostgreSQL rows), exposing CRUD operations and a composition entry point for reads.

In OOR the type is `Repository<T>`, parameterized by the entity class. One repository per entity; one entity per repository.

## How It Works

`Repository<T>` exposes a **narrow** surface, split by axis:

- **By-key (single-row) operations.** `create(entity: T)`, `findById(key)`, `update(entity)`, `delete(key)`. All four funnel through one private `requirePrimaryKey` gate that decides "complete" based on each PK column's `autogeneration` metadata. The same gate, the same error (`IncompletePrimaryKeyError`).
- **Composed reads.** `findOne(options?)` and `findMany(options?)` are the composition entry points: each accepts a `FindOptions<T>`, builds a `QueryBuilder<T>` internally, applies options, and calls the terminal (`getOne()` / `getMany()`) in one shot. `findById` is the one by-key reader; the other read methods own the builder for exactly one call.

The headline rule: **single-row by-key operations on `Repository`; anything composed via `QueryBuilder`.**

> [!note] Refined 2026-04-29 (×4)
> 1. *(ingest of architecture-overview)* Earlier "direct operations execute SQL synchronously" framing was wrong; reads-through-builder is the actual rule.
> 2. *(ingest of repository-contract)* The split is sharpened from "writes vs. reads" to "**by-key vs. composed**." `findById` is by-key (no builder); other reads are composed (via builder). `create` / `update` / `delete` are by-key writes.
> 3. *(ingest of repository-contract)* `create()`'s signature is `T`, **not** `Partial<T>` (commit `92ce5fd`). The argument's *type itself* encodes the contract — the type system does the validation. See § "How decorators shape the contract" below.
> 4. *(drift D1, 2026-04-29)* The earlier framing of `find()` as "the composition entry point" with `findOne` / `findMany` as "sugar" was wrong — there is no `find()` method. `findOne` / `findMany` *are* the composition entry points. See [[sources/drift-d1-repository-find]].

> [!note] Refined 2026-04-30 — PK brand finishes the compile-time direction
> The four PK-using signatures (`findById`, `delete`, `update`, `create`) now consume the [[PrimaryKey Brand|`PrimaryKey<V>` brand]]: `findById` and `delete` take `PKInput<T>`; `update` takes `UnbrandedT<T> & PKInput<T>`; `create` takes `UnbrandedT<T>` and returns `PKOutput<T>` (branded — was `Partial<T>` before). The silent `update({ name: 'x' })` bug on autogen entities is closed at the type level. See [[sources/pk-aware-repository-methods]] and [[0008-pk-aware-compile-time]].

`Repository<T>` is currently constructed directly: `new Repository(User, db)`. The originally-planned `@InjectRepository(Entity)` decorator and accompanying [[Dependency Injection Container|DI container]] are **not yet implemented in `src/`** — see the implementation-status callout on [[0003-singleton-di-container]].

## How decorators shape the contract

The mechanism that makes `create(entity: T)` enforceable at compile time:

- `@NotNullable` and `@Nullable` (from `nullable.ts`) don't just write `NULLABLE_KEY` for runtime metadata — they constrain the field's *declared TypeScript type* via `NullableField<Value>` and `NotNullableField<Value>` mapped types.
- `@PrimaryColumn({ autogeneration })` produces a `NullableField<Value>` (optional on `T`); `@PrimaryColumn` without `autogeneration` produces a `NotNullableField<Value>` (required).
- Both `@PrimaryColumn` overloads further require the [[PrimaryKey Brand|`PrimaryKey<V>` brand]] on the field type — `id!: PrimaryKey<number>` (no autogen) or `id?: PrimaryKey<number>` (with autogen). This is what `findById`/`delete`/`update` consume to derive `PKInput<T>` structurally.

The result: the entity's class declaration *is* the source of truth. `create(entity: T)` doesn't need a separate "is this valid input?" check — TypeScript already knows which fields are required, and a missing field is a compile error. The runtime `requirePrimaryKey` gate exists for callers who bypass type-checking (untyped JS, dynamic dispatch).

This is what [[0005-no-any-type-driven-api]] is load-bearing for: `any` would erase the type-side enforcement and degrade the contract to runtime-only. The compile-time + runtime double-check works *because* the codebase forbids `any`.

## Why It Matters

- Establishes a **clean boundary**: persistence concerns are isolated in one class per entity. Domain services don't issue SQL; they call `repo.X()`.
- Enables **substitution** for tests: a test can inject a fake repository without touching SQL plumbing.
- The trivial/composed split prevents the API from becoming a bag of overloaded methods (the failure mode that haunts older ORMs).

## Examples

Current `examples/` style — direct construction, no DI yet:

```ts
const userRepository = new Repository(User, db);

// By-key write. Returns PKOutput<User> with ONLY the primary-key fields, branded
// (not a hydrated entity — follow with findById() if you need the full row).
const { id } = await userRepository.create({ email: 'a@b.com', name: 'Alice' });

// By-key read.
const user = await userRepository.findById(id);

// Composed read — `where` is a callback receiving a typed proxy.
// See [[Conditions Proxy]]. findMany / findOne build a QueryBuilder
// internally, apply options, and call the terminal in one shot.
const active = await userRepository.findMany({
  where: (u) => [u.active!.eq(true)],
});
```

See `examples/basic-crud/services/UserService.ts` for the canonical no-DI construction pattern.

Planned ergonomics, **once DI ships** (currently aspirational — see [[0003-singleton-di-container]]):

```ts
@Service
class UserService {
  @InjectRepository(User)
  private repo!: Repository<User>;
  // ...
}
```

## Connections

- [[0002-repository-with-lazy-query-builder]] — the ADR fixing the boundary.
- [[Repository]] — the concrete class with all seven operations and the `requirePrimaryKey` gate.
- [[Lazy Query Builder]] / [[QueryBuilder]] — the type composed reads delegate to.
- [[Autogeneration]] — the strategy concept that decides which PK columns are omittable at `create()`.
- [[lifecycle-of-a-create]] — the end-to-end write flow.
- [[query-lifecycle]] — the end-to-end six-step walkthrough of a read.
- [[Conditions Proxy]] — the typed object the `where` callback receives.
- [[PrimaryKey Brand]] — the type-level marker the four PK-using methods consume.
- [[Dependency Injection Container]] — the planned wiring mechanism for `@InjectRepository` (not yet implemented).
- [[0005-no-any-type-driven-api]] — the strict-typing ADR that makes the type-shapes-the-contract mechanism enforceable.
- [[0008-pk-aware-compile-time]] — the ADR that finishes the compile-time direction for `findById` / `delete` / `update`.

## Sources

- `.raw/welcome.md` § "Repository pattern with a lazy query builder"
- `.raw/pk-aware-repository-methods.md` (the 2026-04-30 PK brand work)
