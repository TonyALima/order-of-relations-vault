---
type: component
title: "Repository"
component_kind: class
module: "src/core/repository/"
exports_as: "Repository"
created: 2026-04-29
updated: 2026-04-30
tags:
  - component
  - repository
  - core
status: stable
related:
  - "[[Repository Pattern]]"
  - "[[QueryBuilder]]"
  - "[[Autogeneration]]"
  - "[[PrimaryKey Brand]]"
  - "[[lifecycle-of-a-create]]"
  - "[[query-lifecycle]]"
  - "[[sources/repository-contract]]"
  - "[[sources/pk-aware-repository-methods]]"
sources:
  - "../.raw/repository-contract.md"
  - "../.raw/architecture-overview.md"
  - "../.raw/pk-aware-repository-methods.md"
---

# Repository

## Purpose

`Repository<T>` is the persistence boundary for entity `T`. It owns single-row, by-key operations against one entity. It does **not** compose queries (that's [[QueryBuilder]]), it does **not** wire services (that's the [[Dependency Injection Container|DI container]] when it ships), and it does **not** evolve schema (see [[Database]]'s `create()`/`drop()` and the future `Schema Migrations`).

Headline rule: **narrow surface, deep contract.** Every method that operates on a primary key funnels through one private gate; every error story collapses to one error type; the type system carries the rest.

## Operations

`Repository<T>` exposes exactly **six** public methods. Four of them — `findById`, `delete`, `update`, `create` — derive their input/output PK shapes from the [[PrimaryKey Brand|`PrimaryKey<V>` brand]] on the entity's `@PrimaryColumn` fields.

| Method | Signature | Owns |
|---|---|---|
| `create` | `(entity: UnbrandedT<T>) => Promise<PKOutput<T>>` | Insert one row, return the primary key (branded). |
| `findById` | `(key: PKInput<T>) => Promise<T \| null>` | Locate a row by its full primary key. |
| `findOne` | `(options?: FindOptions<T>) => Promise<T \| null>` | Build a `QueryBuilder<T>`, apply options, run `getOne()`. |
| `findMany` | `(options?: FindOptions<T>) => Promise<T[]>` | Same shape as `findOne`, run `getMany()`. |
| `update` | `(entity: UnbrandedT<T> & PKInput<T>) => Promise<void>` | Overwrite a row identified by its primary key. |
| `delete` | `(key: PKInput<T>) => Promise<void>` | Remove a row by its primary key. |

`findOne` / `findMany` are the **composition entry points** — they accept `FindOptions<T>` and execute one SQL statement. `findById` is the by-key reader. `create`, `update`, `delete` are the write surface.

`PKInput<T>` is the strict input PK shape (every PK key, all required, all non-undefined, **unbranded** — callers pass `{ id: 1 }`, no cast needed). `PKOutput<T>` is the same shape **branded**. `UnbrandedT<T>` is `T` with every field unbranded. See [[PrimaryKey Brand]] § "Helper types" for the derivations.

> [!note] Drift correction 2026-04-29
> An earlier version of this table listed `find(): QueryBuilder<T>` as a "handoff" method. **There is no `find()`.** The builder is constructed inside `findMany` / `findOne`, used once, and discarded. Direct construction (`new QueryBuilder<T>(EntityClass, db)`) is possible but is a library-internal hatch, not a documented user API. See [[sources/drift-d1-repository-find]].

> [!note] Refinement 2026-04-30 — PK-aware compile-time enforcement
> Four signatures changed when the `PrimaryKey<V>` brand shipped:
> - `findById`/`delete` previously accepted `Partial<T>` (any subset). Now they require `PKInput<T>` — `findById({})` and `findById({ name: 'x' })` are compile errors.
> - `update(entity: T)` previously typechecked `update({ name: 'x' })` on autogen entities (because autogen PKs are declared `id?: PrimaryKey<number>` — optional in `T`) and built `WHERE id = NULL` silently. Now `update`'s input is `UnbrandedT<T> & PKInput<T>` — PK fields are required and non-undefined regardless of `T`'s optional modifier.
> - `create()`'s return type went from `Partial<T>` to `PKOutput<T>` (PK fields only, **branded**). Read-side outputs (`findById`/`findOne`/`findMany` returning `T`) are likewise branded; the round-trip `repo.update(await repo.findById({ id }))` works without casts.
>
> See [[sources/pk-aware-repository-methods]] and [[0008-pk-aware-compile-time]].

## The `requirePrimaryKey` gate

Every method that needs a primary key — `create`, `findById`, `update`, `delete` — calls one private method, `requirePrimaryKey`. It decides what counts as "complete" depending on each PK column's `autogeneration` metadata:

- A PK column **with** `autogeneration` is omittable; the gate doesn't require it.
- A PK column **without** `autogeneration` is required; the gate throws `IncompletePrimaryKeyError` if absent.

For composite primary keys, the rule applies **per column**.

This single gate is why the runtime story is symmetric across the four PK-using methods. There isn't a per-method "is this complete?" check; there is one rule, applied four times.

After the [[PrimaryKey Brand]] work, the gate is the **floor** for cast-bypassed callers (`update(data as User)`) — the type system catches the headline cases. The gate's parameter type widened from `Partial<T>` to `PKInput<T> | UnbrandedT<T>`, since the new strict shapes aren't assignable to `Partial<T>` (the brand asymmetry blocks the widening).

## `create()` — the contract in detail

### Compile-time (commit `92ce5fd`, with the nullable suite from `648d9fa`)

The signature is `create(entity: T)`, **not** `Partial<T>`. Compile-time enforcement is delegated to the entity's own type — which the decorators shape via `NullableField<Value>` and `NotNullableField<Value>` types from `nullable.ts`:

| Decorator on field | `T`'s view of field | `create()` requires it? |
|---|---|---|
| `@Column @NotNullable` | required | yes |
| `@Column @Nullable` | optional | no |
| `@PrimaryColumn({ autogeneration: ... })` | optional | no |
| `@PrimaryColumn` without `autogeneration` | required | yes |

The canonical phrasing: *required on the type means required at `create()`*. The only escape is `autogeneration`.

```ts
class CompileUser {
  @PrimaryColumn({ type: COLUMN_TYPE.INTEGER }) id!: number;
  @Column({ type: COLUMN_TYPE.TEXT }) @NotNullable name!: string;
  @Column({ type: COLUMN_TYPE.TEXT }) @NotNullable email!: string;
}

// @ts-expect-error — email required
repo.create({ id: 1, name: 'Alice' });
```

What `create()` does **not** check at compile time: relations and FK shape. Those still go through runtime metadata.

### Runtime

`create()` returns `Promise<PKOutput<T>>` — the returned object contains exactly the **primary-key fields** (branded), populated from the SQL `RETURNING` clause. It does **not** hand back a hydrated entity.

The destructured `id` is `PrimaryKey<number>`, which is assignable to `number` anywhere a plain number is expected, so `const { id } = await repo.create(...); await repo.findById({ id })` works without ceremony — and the `id!` non-null assertion seen in older code is now redundant (`PKOutput<T>` makes PK fields non-undefined).

> [!key-insight] Why `create()` doesn't return the full row
> *"`create()`'s job is to commit a row and tell you how to find it again, not to re-read it."* If you want the full row, follow with `findById()`. This is more honest than ORMs that auto-rehydrate — the caller decides whether the round-trip is worth it.

For autogenerated PK columns (see [[Autogeneration]]):

- `clientSide`: the library calls the function once before the INSERT, writes the value into both the SQL statement and the returned key.
- `dbSide`: the column is omitted from the INSERT entirely; the database fills it; the value comes back via `RETURNING`.
- **An explicit caller value always wins** (commit `cfe4cc5`). Passing `id: 42` to a SERIAL or UUID column is a supported override.

If a non-autogenerated PK column is absent, `create()` throws `IncompletePrimaryKeyError` — same path as the read methods.

See [[lifecycle-of-a-create]] for the end-to-end flow.

## How reads compose

There is no caller-facing builder handoff today. `findOne(options?)` and `findMany(options?)` are the composition entry points: they construct a `QueryBuilder<T>`, call `applyOptions(options)`, then call the terminal method (`getOne()` / `getMany()`). The builder is short-lived and discarded after one terminal call.

The `where` callback inside `FindOptions<T>` is where composition lives — see [[Conditions Proxy]] for the shape and [[QueryBuilder]] / [[Lazy Query Builder]] for the mechanics.

If you need direct access to a `QueryBuilder<T>` (e.g. to share state across helpers), construct one explicitly: `new QueryBuilder<T>(EntityClass, db)`. This is a library-internal hatch — not a documented user-facing API — and the repository does not facilitate it.

## Failure Modes

| Where | Error | Notes |
|---|---|---|
| `findById(key)`, `delete(key)` | `IncompletePrimaryKeyError` | Any PK property absent from `key`. (Type system catches this first via `PKInput<T>`; the runtime gate catches cast-bypassed callers.) |
| `create(entity)` | `IncompletePrimaryKeyError` | Non-autogenerated PK property absent. (Type system catches this first via `UnbrandedT<T>`'s required-modifier preservation; the runtime gate catches bypassed-type-check callers.) |
| `update(entity)` | `IncompletePrimaryKeyError` | Indirectly, via `requirePrimaryKey`. (Type system catches missing PK first via `UnbrandedT<T> & PKInput<T>`.) |
| Anything else | *(propagates from driver)* | Connection failures, constraint violations, type mismatches — these come from `pg` / Bun SQL unwrapped. The repository is intentionally thin around them; wrapping would obscure the driver's diagnostic. |

The error carries `entityName` and `missingProperties: string[]`, both populated and asserted by tests.

## What's deliberately out of scope

| Feature | Why it's out |
|---|---|
| **Partial updates** | `update(entity: T)` takes full `T`. Field-level merge semantics belong on a builder, not a repository. |
| **Identity map** | Two `findById({ id: 1 })` calls may return two distinct `User` objects. Equality is **by value**, not by reference. |
| **Cascade** | Relations are written when present in the input object and ignored otherwise. The repository never reaches into another table on its own. |
| **Transaction management** | Transactions are the caller's job (or a future concern). The repository takes a `Database` and uses whatever connection it hands. |

The closing line from the source is worth keeping: *"These are not gaps; they are the boundary. Anything composed, reactive, or stateful goes elsewhere."*

## Connections

- [[Repository Pattern]] — the concept this class realizes.
- [[QueryBuilder]] — constructed inside `findOne` / `findMany`; used once, then discarded.
- [[Autogeneration]] — the strategy concept this class executes against.
- [[PrimaryKey Brand]] — the type-level marker the four PK-using signatures consume.
- [[lifecycle-of-a-create]] — the end-to-end write flow.
- [[query-lifecycle]] — the read flow `findOne`/`findMany` ride on.
- [[MetadataStorage]] — what the repository reads to compose SQL.
- [[Layered Architecture]] — `Repository` is layer 3 in the five-layer stack.
- [[0002-repository-with-lazy-query-builder]] — the ADR.
- [[0005-no-any-type-driven-api]] — the ADR that makes the type-level contract enforceable.
- [[0008-pk-aware-compile-time]] — the ADR that ties the four PK-using signatures to the brand.

## Sources

- `.raw/repository-contract.md` (the pin-down for the operations table and `requirePrimaryKey` gate)
- `.raw/architecture-overview.md` § "The Boundary Between Repository and Query Builder"
- `.raw/pk-aware-repository-methods.md` (the design memo for the 2026-04-30 signature changes)
