---
type: concept
title: "PrimaryKey Brand"
complexity: advanced
domain: "TypeScript / type system"
aliases:
  - "PrimaryKey<V>"
  - "PK brand"
  - "Branded primary key"
created: 2026-04-30
updated: 2026-04-30
tags:
  - concept
  - typescript
  - primary-key
  - brand
status: stable
related:
  - "[[Repository]]"
  - "[[Repository Pattern]]"
  - "[[Autogeneration]]"
  - "[[Conditions Proxy]]"
  - "[[ECMAScript Stage-3 Decorators]]"
  - "[[0008-pk-aware-compile-time]]"
  - "[[sources/pk-aware-repository-methods]]"
sources:
  - "../.raw/pk-aware-repository-methods.md"
---

# PrimaryKey Brand

## Definition

`PrimaryKey<V>` is a **structural brand** that marks a primary-key field at the type level. It is purely a type-level construct — at runtime a `PrimaryKey<number>` is just a `number`.

```ts
declare const __pkBrand: unique symbol;

/** Marks a primary-key field. Purely a type-level brand — erased at runtime. */
export type PrimaryKey<V> = V & { readonly [__pkBrand]: true };

/** Strips the `PrimaryKey<>` brand; passes other types through. */
export type Unbrand<V> = V extends PrimaryKey<infer U> ? U : V;
```

`PrimaryKey<V>` lives in the column-decorator module and is exported from the public package barrel — users need it to declare entities. `Unbrand<V>` is internal companion machinery.

## How It Works

### The brand is an intersection

`PrimaryKey<V>` is `V & { readonly [__pkBrand]: true }`. Any type with a `[__pkBrand]: true` property in its intersection extends `PrimaryKey<unknown>`, regardless of how it was constructed. The `unique symbol` is module-scoped — importing `PrimaryKey<V>` does **not** surface the symbol to consumers.

### Brand asymmetry — the load-bearing ergonomic trick

Because `PrimaryKey<V>` is a **subtype** of `V`:

| Direction | Compiles? | Reason |
|---|---|---|
| `PrimaryKey<number>` → `number` | ✓ | Intersection narrows to its left operand. |
| `number` → `PrimaryKey<number>` | ✗ | Plain `number` lacks the brand. |

OOR exploits this asymmetry to keep call sites brand-free:

- **Outputs** keep `T` branded. The brand is information; preserving it lets callers pass returned entities back into other methods without ceremony.
- **Inputs** accept *unbranded* values via the `Unbrand<V>` mapped type. Callers write `repo.findById({ id: 1 })` with a plain literal.

The round-trip `repo.update(await repo.findById({ id: 1 }))` works without any cast. The brand is invisible to consumers in normal flows; it shows up only at the **declaration** site, which is exactly where the visibility serves its purpose.

> [!key-insight] Why a brand and not a generic on `Repository<T, PK>`
> A generic `Repository<T, PK extends keyof T>` would force users to write `new Repository<User, 'id'>(User, db)` — verbose at construction, and TypeScript can't catch a mismatch between the `PK` generic and the runtime `@PrimaryColumn` metadata. `Repository<User, 'name'>` would typecheck silently and disagree with the decorator at runtime. **A brand on the declaration site closes that gap** — the field type and the decorator constraint are checked against each other at the same call. See [[0008-pk-aware-compile-time]] for the full alternative analysis.

## Helper types

Internal — not exported from the public barrel. They live alongside `Repository` and consume the brand structurally.

```ts
type PKKeys<T> = {
  [K in keyof T]-?: NonNullable<T[K]> extends PrimaryKey<unknown> ? K : never;
}[keyof T];

type UnbrandedT<T>      = { [K in keyof T]: Unbrand<T[K]> };
export type PKInput<T>  = { [K in PKKeys<T>]-?: NonNullable<Unbrand<T[K]>> };
export type PKOutput<T> = { [K in PKKeys<T>]-?: NonNullable<T[K]> };
```

| Type | Shape | Used as |
|---|---|---|
| `PKKeys<T>` | union of property names whose value extends `PrimaryKey<unknown>` | the structural extractor; `NonNullable<>` lets autogen `id?: PrimaryKey<number>` still match |
| `UnbrandedT<T>` | `T` with every field unbranded | `create()` input, `update()` entity shape |
| `PKInput<T>` | every PK key, all required, all non-undefined, **unbranded** | `findById(key)`, `delete(key)` |
| `PKOutput<T>` | every PK key, all required, all non-undefined, **branded** | `create()` return shape |

`PKInput<T>` and `PKOutput<T>` are exported from the repository module (so example apps and tests can reference them) but are **not** in the public barrel — library users do not construct or reason about them.

## How decorators enforce the brand

`@PrimaryColumn` carries the brand into the field-type constraint. Both overloads now include a brand term:

```ts
type NullablePrimaryKey<V> = PrimaryKey<V> | undefined;

// With autogeneration → field must be optional AND branded
ClassFieldDecoratorContext<This, NullableField<Value> & NullablePrimaryKey<Value>>

// Without autogeneration → field must be required AND branded
ClassFieldDecoratorContext<This, NotNullableField<Value> & PrimaryKey<Value>>
```

Six declaration shapes resolve cleanly:

| Field declaration | autogen? | Outcome |
|---|---|---|
| `id!: PrimaryKey<number>` | no | ✓ accept |
| `id?: PrimaryKey<number>` | yes | ✓ accept |
| `id!: number` | no | ✗ reject (no brand) |
| `id?: number` | yes | ✗ reject (no brand) |
| `id?: PrimaryKey<number>` | no | ✗ reject (`?` modifier wrong) |
| `id!: PrimaryKey<number>` | yes | ✗ reject (`!` modifier wrong) |

The four "wrong" cases fail at the decorator call site with a `Type ... is not assignable to ...` error.

This is a **second use of the constraint-flip pattern** in the codebase — the first being the `@Nullable` / `@NotNullable` enforcement of nullability via `NullableField<V>` / `NotNullableField<V>`. See [[ECMAScript Stage-3 Decorators]] for the pattern.

## Why It Matters

- **Closes the silent `update` bug.** Before the brand, `update({ name: 'x' })` typechecked on entities with autogen PKs (declared `id?: number`) and silently produced `WHERE id = NULL`. With `update(entity: UnbrandedT<T> & PKInput<T>)`, `id` is required and non-undefined at compile time. See [[Autogeneration]] for the upstream half of the change.
- **Tightens `findById` / `delete`.** `findById({})` and `findById({ name: 'x' })` no longer compile.
- **Symmetric read-side.** `findById`, `findOne`, `findMany`, and `create` all return branded `T` (or `PKOutput<T>`). Caller code never has to "remember" which methods return PKs in branded form.
- **Brand-free call sites.** `c.id?.eq(1)` works because [[Conditions Proxy]] uses `Unbrand<T[K]>` per field; `repo.findById({ id: 1 })` works because `PKInput<T>` is the unbranded shape; the brand only shows up where the user **declares** an entity.

## Pitfalls

### Manual brand construction is rare; if you need it, cast

There is no `pk<V>(v: V): PrimaryKey<V>` helper. Most paths go through repo methods which strip brands on input. If a user needs to construct a branded value (e.g. building a `PKOutput<T>` literal in a test), `value as PrimaryKey<V>` is the idiom. Deferred until proven painful.

### `NonNullable<>` in `PKKeys<T>` is load-bearing

Autogen PKs are declared `id?: PrimaryKey<number>` — the type of `T['id']` is `PrimaryKey<number> | undefined`. Without `NonNullable<>`, the conditional `T[K] extends PrimaryKey<unknown>` would test the union directly and fail. Wrapping with `NonNullable<>` strips `undefined` first.

### The brand cannot be auto-injected by a decorator

Stage-3 decorators can READ a field's declared type but cannot inject type information into it. So the `PrimaryKey<V>` annotation has to live at every `@PrimaryColumn` field declaration — there is no "smart decorator" that adds it for you. This is a structural limitation of the dialect, not a missing feature. See [[ECMAScript Stage-3 Decorators]].

## Examples

### Declaration

```ts
class User {
  @PrimaryColumn({ type: COLUMN_TYPE.INTEGER, autogeneration: { dbSide: () => undefined } })
  id?: PrimaryKey<number>;

  @Column({ type: COLUMN_TYPE.TEXT }) @NotNullable
  email!: string;
}

class OrderItem {
  @PrimaryColumn({ type: COLUMN_TYPE.INTEGER })
  orderId!: PrimaryKey<number>;

  @PrimaryColumn({ type: COLUMN_TYPE.INTEGER })
  productId!: PrimaryKey<number>;

  @Column({ type: COLUMN_TYPE.INTEGER }) @NotNullable
  quantity!: number;
}
```

### Use

```ts
const userRepo = new Repository(User, db);

// Compile-time PK enforcement
await userRepo.findById({});                 // ✗ — PKInput<User> requires `id`
await userRepo.findById({ name: 'x' });      // ✗ — wrong field
await userRepo.findById({ id: 1 });          // ✓ — plain number; brand stripped on input
await userRepo.update({ name: 'x' });        // ✗ — `id` required even though T['id'] is optional

// Round-trip without casts
const u = await userRepo.findById({ id: 1 });
if (u) await userRepo.update(u);             // ✓ — branded T is a subtype of UnbrandedT<T>

// Composite PK
const itemRepo = new Repository(OrderItem, db);
await itemRepo.findById({ orderId: 1 });                       // ✗ — productId missing
await itemRepo.findById({ orderId: 1, productId: 2 });         // ✓
```

## Connections

- [[Repository]] — owns the four signatures that consume the brand.
- [[Autogeneration]] — declares autogen PKs as `id?: PrimaryKey<number>`; the optional modifier on `T` is what `create()` reads to decide "omittable" at compile time.
- [[Conditions Proxy]] — uses `Unbrand<T[K]>` per field so `c.id?.eq(1)` accepts a plain literal.
- [[ECMAScript Stage-3 Decorators]] — the constraint-flip pattern that enforces the brand at the `@PrimaryColumn` declaration site.
- [[0008-pk-aware-compile-time]] — the ADR choosing the brand over the generic and runtime-only alternatives.
- [[0005-no-any-type-driven-api]] — the strict-typing ADR that makes the brand pattern enforceable.

## Sources

- `.raw/pk-aware-repository-methods.md` (the design memo behind the work).
