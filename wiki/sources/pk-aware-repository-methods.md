---
type: source
title: "PK-aware Repository Methods"
source_type: design-note
author: "Tony Albert"
date_published: 2026-04-30
url: ""
confidence: high
key_claims:
  - "After create-required-fields shipped, three of the remaining five Repository methods had loose contracts: findById/delete accepted any Partial<T>; update silently accepted entities missing autogen PKs and built WHERE id = NULL."
  - "Compile-time PK awareness is structurally impossible without explicit user input — Stage-3 decorators can read a field's declared type but cannot inject type information into it. Only two source-of-truth options exist: (A) a brand on the field type, or (B) a generic PK key on Repository<T, PK>."
  - "Option A wins on two grounds: consistency with the existing compile-time direction, and the unacceptable footgun in B (Repository<User, 'name'> would typecheck silently and disagree with @PrimaryColumn metadata at runtime)."
  - "PrimaryKey<V> = V & { readonly [__pkBrand]: true } — an intersection type. The brand is purely structural; runtime is unchanged."
  - "Brand asymmetry is the load-bearing ergonomic trick: branded → unbranded works freely (intersection narrows to its left operand); unbranded → branded requires a cast. Outputs stay branded; inputs accept unbranded values via Unbrand<V>."
  - "@PrimaryColumn's two overloads gain a brand term in their ClassFieldDecoratorContext constraint, enforcing at the decorator call site that every PK field is declared with PrimaryKey<V>."
  - "The six declaration shapes (with/without autogen × PrimaryKey<V> | V | undefined) all resolve correctly: two accept, four reject."
  - "PKKeys<T> = { [K in keyof T]-?: NonNullable<T[K]> extends PrimaryKey<unknown> ? K : never }[keyof T] — derives PK property names structurally from T."
  - "Method signatures: findById(key: PKInput<T>); delete(key: PKInput<T>); update(entity: UnbrandedT<T> & PKInput<T>); create(entity: UnbrandedT<T>): Promise<PKOutput<T>>. findOne / findMany are untouched (they take FindOptions<T>, not raw entity shapes)."
  - "create() now returns PKOutput<T> (branded), not Partial<T>. The brand is preserved across the read-side outputs (findById/findOne/findMany also return branded T)."
  - "update() keeps a runtime requirePrimaryKey defense as a safety net for cast-bypassed callers; the type system is the headline, the runtime check is the floor."
  - "requirePrimaryKey's parameter type was widened from Partial<T> to PKInput<T> | UnbrandedT<T> because the new strict PK shapes are not assignable to Partial<T>."
  - "Conditions<T> changed to use Unbrand<T[K]> per field, so users write c.id?.eq(1) without ceremony — the brand is erased at runtime, so SQL comparison doesn't care."
  - "PrimaryKey<V> is exported from the public package barrel; PKInput<T>, UnbrandedT<T>, Unbrand<V>, PKOutput<T> are NOT — they are internal wiring."
  - "Out of scope deliberately: pk<V>(v) casting helper, branding non-PK fields (Email<string>), a decorator that auto-injects the brand (structurally impossible)."
created: 2026-04-30
updated: 2026-04-30
tags:
  - source
  - repository
  - typescript
  - primary-key
status: stable
related:
  - "[[sources/repository-contract]]"
  - "[[sources/welcome]]"
  - "[[Repository]]"
  - "[[Repository Pattern]]"
  - "[[PrimaryKey Brand]]"
  - "[[Autogeneration]]"
  - "[[Conditions Proxy]]"
  - "[[ECMAScript Stage-3 Decorators]]"
  - "[[0008-pk-aware-compile-time]]"
sources:
  - "../.raw/pk-aware-repository-methods.md"
---

# PK-aware Repository Methods

## Summary

The pin-down note for the type-level primary-key work that finishes the create-required-fields direction. After `create(entity: T)` started enforcing required-field correctness at compile time, the remaining four Repository methods kept loose contracts — three were broken: `findById` / `delete` accepted any `Partial<T>` (loud at runtime via `IncompletePrimaryKeyError`); `update` was worse — autogen PKs declared `id?: number` made `update({ name: 'x' })` typecheck and silently produce `WHERE id = NULL`. This source is the narrative behind the chosen fix: a structural brand on PK field types, plus updated method signatures that consume the brand to derive PK shapes.

The source is post-implementation — every code snippet matches what shipped.

## Key Claims

### The problem

- `findById(key: Partial<T>)` and `delete(key: Partial<T>)` accepted *any* subset of `T`. Loose at the type level; loud at runtime.
- `update(entity: T)` was worse: with autogeneration, autogen PKs are declared `id?: number`, so `T` itself allowed `update({ name: 'x' })`. The body built `WHERE id = NULL`, matched no rows, returned silently. **A silent runtime bug.**
- `findMany` / `findOne` were untouched — they take a `FindOptions<T>` query-builder, not a raw entity shape.

### Why the fix is structurally constrained

TypeScript has no way to know which fields of `T` are PKs. The decorator stores PK-ness in **runtime** metadata. Stage-3 decorators can READ a field's declared type and reject it via the constraint-flip pattern (the existing pattern from `@PrimaryColumn` and `@Nullable` / `@NotNullable`), but they **cannot inject** type information into the field. So compile-time PK awareness can only come from:

1. The user explicitly writing a marker on PK fields' declared types.
2. The user explicitly threading PK keys as a generic at `Repository` construction (`new Repository<User, 'id'>(...)`).

There is no third option.

### Alternatives considered

- **A. Branded `PrimaryKey<V>` marker on the field type.** All three methods get tight types automatically; cost is ~15-25 PK declaration sites changing to `id!: PrimaryKey<number>`.
- **B. Generic `Repository<T, PK extends keyof T>`.** No entity-declaration changes; verbose at construction; **TypeScript won't catch a mismatch between the `PK` generic and the `@PrimaryColumn` metadata** — `Repository<User, 'name'>` typechecks silently, breaks at runtime. Footgun unacceptable for a publishable library.
- **C. Runtime-only tightening.** Smallest change; no API impact; but the type system gains nothing — `findById({})` still compiles; `update({ name: 'x' })` still compiles. Closes the silent-bug case but is a step backward from the existing direction.

### Why A was chosen

1. **Consistency with the existing compile-time direction.** The prior work paid a real migration cost (every SERIAL entity got `autogeneration: { dbSide: () => undefined }`) for compile-time correctness; finishing the job is a small marginal step.
2. **B's footgun is unacceptable for a published library.** A user writing `Repository<User, 'name'>` would silently get a `Repository` whose runtime PK check disagrees with its compile-time PK type.

### The brand and its asymmetry

```ts
declare const __pkBrand: unique symbol;
export type PrimaryKey<V> = V & { readonly [__pkBrand]: true };
export type Unbrand<V> = V extends PrimaryKey<infer U> ? U : V;
```

`PrimaryKey<V>` is `V & { brand }` — an intersection type, and a **subtype** of `V`. Assignment in the subtype-to-supertype direction works; the reverse requires a cast. This asymmetry is exploited:

- **Outputs** (returns from `findMany`/`findOne`/`findById`/`create`) keep `T` branded. Preserving the brand lets callers re-pass returned entities into `update` trivially.
- **Inputs** (key shapes for `findById`/`delete`, entity shapes for `create`/`update`) accept *unbranded* values via a mapped type. Callers write `repo.findById({ id: 1 })` with a plain literal, no cast.

The round-trip `repo.update(await repo.findById({ id: 1 }))` works without casts. The brand is invisible to consumers in normal flows; it shows up only at the *declaration* site.

### `@PrimaryColumn` overloads gain a brand term

```ts
type NullablePrimaryKey<V> = PrimaryKey<V> | undefined;

// Overload 1: with autogeneration → field must be optional AND branded
export function PrimaryColumn<OptValue>(
  options: ColumnOptions<OptValue> & { autogeneration: Autogeneration<OptValue> },
): <This, Value>(
  _value: undefined,
  context: ClassFieldDecoratorContext<This, NullableField<Value> & NullablePrimaryKey<Value>>,
) => void;

// Overload 2: without autogeneration → field must be required AND branded
export function PrimaryColumn<OptValue>(
  options: ColumnOptions<OptValue> & { autogeneration?: undefined },
): <This, Value>(
  _value: undefined,
  context: ClassFieldDecoratorContext<This, NotNullableField<Value> & PrimaryKey<Value>>,
) => void;
```

The implementation signature is unchanged; the type-system distinction lives in the overloads. All six declaration shapes (with/without autogen × `PrimaryKey<number>` | `number` | `?` modifier mismatches) resolve correctly: two accept, four reject.

### Type helpers

Internal-only, alongside `Repository`:

```ts
type PKKeys<T> = {
  [K in keyof T]-?: NonNullable<T[K]> extends PrimaryKey<unknown> ? K : never;
}[keyof T];

type UnbrandedT<T>     = { [K in keyof T]: Unbrand<T[K]> };
export type PKInput<T> = { [K in PKKeys<T>]-?: NonNullable<Unbrand<T[K]>> };
export type PKOutput<T>= { [K in PKKeys<T>]-?: NonNullable<T[K]> };
```

`PKInput<T>` is the strict input PK shape (every PK key, all required, all non-undefined, **unbranded**). `PKOutput<T>` is the same shape, **branded**. `UnbrandedT<T>` is `T` with every field unbranded (used for `create()` and `update()` entity shapes).

### Method signatures

```ts
class Repository<T extends object> {
  findMany(options?: FindOptions<T>): Promise<T[]>;
  findOne(options?: FindOptions<T>): Promise<T | null>;
  findById(key: PKInput<T>): Promise<T | null>;
  create(entity: UnbrandedT<T>): Promise<PKOutput<T>>;
  delete(key: PKInput<T>): Promise<void>;
  update(entity: UnbrandedT<T> & PKInput<T>): Promise<void>;
}
```

`findById({})`, `findById({ name: 'x' })`, `update({ name: 'x' })` — all compile errors. `update({ id: 1, ... })` compiles. `findById({ orderId: 1 })` on a composite-PK entity is a compile error — `PKInput<T>` requires *all* PK fields.

### Sub-decisions

- **`update()` keeps a runtime PK defense.** Callers can bypass types with casts (`update(data as User)`). The body still runs `requirePrimaryKey(entity)` so cast-bypassed callers get a loud `IncompletePrimaryKeyError` instead of `WHERE id = NULL`.
- **`create()` returns `PKOutput<T>` (branded), not `PKInput<T>` (unbranded).** Keeps the read-side symmetric: every method that hands an entity (or PK-shaped subset) back to the caller hands it back branded. The destructured `id` is `PrimaryKey<number>`, assignable to `number` anywhere a plain number is expected; the `id!` non-null assertion in patterns like `repo.findById({ id: id! })` becomes redundant.
- **`requirePrimaryKey`'s parameter widened from `Partial<T>` to `PKInput<T> | UnbrandedT<T>`.** The new strict shapes aren't assignable to `Partial<T>` — the brand asymmetry blocks it.
- **`Conditions<T>` consumes `Unbrand<T[K]>`, not `T[K]`.** Otherwise `c.id?.eq(1)` would reject the plain literal `1`. The fix: `Conditions<T> = { [K in keyof T]?: FieldConditionBuilder<Unbrand<T[K]>> }`. Runtime-erased; no SQL impact.
- **`findMany` and `findOne` are not touched.** Their inputs are `FindOptions<T>`, not raw entity shapes.

### Out of scope (deliberately)

- `pk<V>(v: V): PrimaryKey<V>` casting helper. Defer until proven painful.
- Branding non-PK fields (`Email<string>`, `UserId<string>`). Different design space.
- Decorator that injects the brand automatically. Structurally impossible.
- Promoting `PKInput<T>` / `UnbrandedT<T>` / `Unbrand<V>` / `PKOutput<T>` to the public barrel. Internal until a user asks.

## Entities Mentioned

- [[order-of-relations]] — the codebase this work shipped in.
- [[TypeScript]] — host of the brand pattern; Stage-3 decorators + intersection types are the underlying mechanism.
- [[PostgreSQL]] — uninvolved at runtime; the brand is purely a type-level construct.

## Concepts Introduced

- [[PrimaryKey Brand]] — the `PrimaryKey<V>` brand, the asymmetric-intersection ergonomics, and the helper types.

## Concepts Refined

- [[Autogeneration]] — closes the silent-`update` bug by name. The autogeneration mechanism made the silent bug structurally possible (autogen PKs declared `id?: number`); the brand work is the type-level half that the runtime gate alone could not provide.
- [[ECMAScript Stage-3 Decorators]] — the constraint-flip pattern is now used to enforce the brand at the `@PrimaryColumn` declaration site, in addition to nullability.
- [[Conditions Proxy]] — `Conditions<T>` now uses `Unbrand<T[K]>` per field; the user-facing surface is unchanged in source-line shape (`c.id?.eq(1)`), but the brand is stripped from the value parameter type.

## Components Refined

- [[Repository]] — four method signatures change (`findById`, `delete`, `update`, `create`), the `requirePrimaryKey` helper widens its parameter type, and the **`create()` return type changes from `Partial<T>` to `PKOutput<T>`**.

## Decisions Introduced

- [[0008-pk-aware-compile-time]] — chooses Option A (the brand on the field type) over Option B (generic PK keys) and Option C (runtime-only tightening).

## Connections to Existing Work

- [[sources/repository-contract]] — defined `create(entity: T)` as the first compile-time leg. This source completes the direction for the other three PK-using methods.
- [[Autogeneration]] — declares autogen PKs as `id?: PrimaryKey<number>` after this change; the optional modifier on `T` is still load-bearing for `create()`.

## Notes

The closing line of the source pins the design throughline: every method that hands an entity back hands it back branded; every input accepts unbranded values; the brand is invisible at the call site because the asymmetry of `PrimaryKey<V> & V` (subtype on the way out, supertype on the way in) was the load-bearing ergonomic trick.
