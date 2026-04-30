# PK-aware compile-time enforcement for `findById`, `delete`, and `update`

This note is the narrative behind the type-level primary-key work that finishes the create-required-fields direction. It captures the problem, the options, the chosen design, and the sub-decisions that landed in the implementation. It is self-contained on purpose — every code snippet here matches what shipped, and no external file is required to read it.

For the surrounding system, see [[Repository Pattern]] and [[ECMAScript Stage-3 Decorators]]. For the upstream half of the same direction, see [[Autogeneration]].

---

## The problem we landed on

After the create-required-fields work shipped, `Repository<T>.create(entity: T)` enforced required-field correctness at compile time. The other four repository methods kept their pre-existing input contracts — and three of them had problems:

- `findById(key: Partial<T>)` and `delete(key: Partial<T>)` accepted *any* subset of `T`. A caller could pass `{}`, or `{ name: 'x' }`, and only fail at runtime via `IncompletePrimaryKeyError`. Type-loose, but at least loud.
- `update(entity: T)` was worse. With autogeneration, autogen PKs are declared `id?: number` — optional in `T`. So `T` itself allowed `update({ name: 'x' })`. The body would build `WHERE id = NULL`, match no rows, and return silently. **A silent runtime bug.**

`findMany(options?: FindOptions<T>)` and `findOne(options?: FindOptions<T>)` were unaffected — they take a query-builder, not a raw entity shape.

---

## Why this is structurally hard

TypeScript has no way to know which fields of `T` are primary keys. The decorator stores PK-ness in **runtime metadata** (the `@PrimaryColumn` decorator pushes a flag into `context.metadata`), not in the type system. Stage-3 decorators can READ the field's declared type and reject incompatible declarations (the constraint-flip pattern from `@PrimaryColumn` and `@Nullable`/`@NotNullable`), but they **cannot inject type information into the field**.

So compile-time PK awareness has to come from one of two places:

1. The user explicitly writing a marker on PK fields' declared types.
2. The user explicitly threading PK keys as a generic at `Repository` construction (`new Repository<User, 'id'>(...)`).

There is no third option. The decorator can't introduce information out of thin air.

---

## What we considered

### A. Branded `PrimaryKey<V>` marker on the field type

User declares `id!: PrimaryKey<number>` instead of `id!: number`. The brand is purely structural — at runtime, `id` is just a number. The type system uses the brand to derive PK-key information via a conditional mapped type.

- **Pros**: pure type-level dispatch; all three methods get tight types automatically; no per-instantiation ceremony.
- **Cons**: every PK declaration changes (~15-25 sites in this codebase); the brand is visible at the declaration site and can be confused with a runtime wrapper, even though it isn't.

### B. Generic `Repository<T, PK extends keyof T>`

User threads PK keys explicitly: `new Repository<User, 'id'>(User, db)`.

- **Pros**: no entity-declaration changes.
- **Cons**: verbose at construction; the user must remember to declare PK; **TypeScript won't catch a mismatch between the `PK` generic and the `@PrimaryColumn` decorator metadata** — declaring `Repository<User, 'name'>` would typecheck silently and only break at runtime. For a publishable library, that's a bad footgun.

### C. Runtime-only tightening

Don't change types. Add a stricter runtime check at the start of `update()` that rejects missing-or-`undefined` PK values.

- **Pros**: smallest change, no API impact, no migration.
- **Cons**: the type system gains nothing — `findById({})` still compiles; `update({ name: 'x' })` still compiles. Closes the silent-bug case but is a step backward from the type-precision direction the previous work set.

---

## Why we chose A

Two arguments converged on the brand:

1. **Consistency with the existing direction.** The prior work went out of its way to lift validation from runtime to compile time. Falling back to runtime-only here would be inconsistent — the user already paid the migration cost (every SERIAL entity got `autogeneration: { dbSide: () => undefined }`) for compile-time correctness; finishing the job is a small marginal step.
2. **The footgun in Option B is unacceptable for a publishable library.** A user writing `Repository<User, 'name'>` would silently get a `Repository` whose runtime PK check disagrees with its compile-time PK type.

The cost of A — `id!: PrimaryKey<number>` at every PK declaration — is bounded and mechanical. ~15-25 sites, all in entities or test fixtures. Search-and-replace migration with no logic changes.

---

## The key insight that made A ergonomic: brand asymmetry

`PrimaryKey<V>` is `V & { readonly [__pkBrand]: true }` — an intersection type. That makes `PrimaryKey<V>` a **subtype** of `V`. Assignment in the subtype-to-supertype direction works; the reverse requires a cast.

Concretely:

- `PrimaryKey<number>` → `number` ✓ (intersection narrows to its left operand).
- `number` → `PrimaryKey<number>` ✗ (plain number lacks the brand).

We exploited this:

- **Outputs** (returns from `findMany`, `findOne`, `findById`) keep `T` branded. The brand is information; preserving it lets callers re-pass returned entities into `update` trivially.
- **Inputs** (key shapes for `findById`/`delete`, entity shapes for `create`/`update`) accept *unbranded* values via a mapped type. Callers write `repo.findById({ id: 1 })` with a plain literal, no cast.

Because branded → unbranded is the easy direction, the round-trip `repo.update(await repo.findById({ id: 1 }))` works without any casts. The brand is invisible to consumers in normal flows; it shows up only at the *declaration* site, which is exactly where we want it to show up.

---

## The brand: `PrimaryKey<V>`

Lives in the column-decorator module. Exported from the public package barrel because users need it to declare entities.

```ts
declare const __pkBrand: unique symbol;

/** Marks a primary-key field. Purely a type-level brand — erased at runtime. */
export type PrimaryKey<V> = V & { readonly [__pkBrand]: true };

/** Strips the `PrimaryKey<>` brand from a value type; passes other types through. */
export type Unbrand<V> = V extends PrimaryKey<infer U> ? U : V;
```

The `unique symbol` declaration is at module scope; importing `PrimaryKey<V>` does not surface the symbol to consumers (they don't need it). Any type that has a `[__pkBrand]: true` property in its intersection extends `PrimaryKey<unknown>`, regardless of how it was constructed.

`Unbrand<V>` is the companion that the input-side mapped types use to peel the brand off.

---

## `@PrimaryColumn` decorator constraint update

The two existing overloads gain an additional brand term in their `ClassFieldDecoratorContext` constraint. The decorator now enforces, at compile time, that every `@PrimaryColumn` field is declared with the brand.

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

The implementation signature is unchanged (it accepts the union of both overload shapes; the type-system distinction lives in the overloads).

### Case analysis

| Field declaration | Overload | Constraint resolves to | Field extends? | Outcome |
|---|---|---|---|---|
| `id!: PrimaryKey<number>` (no autogen) | 2 | `PrimaryKey<number>` | yes | ✓ accept |
| `id?: PrimaryKey<number>` (with autogen) | 1 | `PrimaryKey<number> \| undefined` | yes | ✓ accept |
| `id!: number` (no autogen) | 2 | `number & {brand}` | no (no brand) | ✗ reject |
| `id?: number` (with autogen) | 1 | `(number & {brand}) \| undefined` | no | ✗ reject |
| `id?: PrimaryKey<number>` (no autogen) | 2 | `never` | no | ✗ reject |
| `id!: PrimaryKey<number>` (with autogen) | 1 | `never` | no | ✗ reject |

All six declaration shapes resolve correctly; the four "wrong" cases fail at the decorator call site.

---

## Type helpers

Internal helpers live alongside the `Repository` class. They derive PK information structurally from `T`:

```ts
import type { PrimaryKey, Unbrand } from '../../decorators/column/column';

type PKKeys<T> = {
  [K in keyof T]-?: NonNullable<T[K]> extends PrimaryKey<unknown> ? K : never;
}[keyof T];

type UnbrandedT<T> = { [K in keyof T]: Unbrand<T[K]> };

export type PKInput<T> = { [K in PKKeys<T>]-?: NonNullable<Unbrand<T[K]>> };

export type PKOutput<T> = { [K in PKKeys<T>]-?: NonNullable<T[K]> };
```

- `Unbrand<V>` (defined with the brand): strips the brand when present; passes other types through.
- `PKKeys<T>`: union of property names whose value type extends `PrimaryKey<unknown>`. Uses `NonNullable<>` so optional-declared PKs (autogen, `id?: PrimaryKey<number>`) still match.
- `UnbrandedT<T>`: `T` with every field unbranded. Used as the input shape for `create()` and the entity shape for `update()`.
- `PKInput<T>`: the strict PK shape — every PK key, all required, all non-undefined, **unbranded**. Used as the key shape for `findById`/`delete`.
- `PKOutput<T>`: the same shape, **branded**. Used as the return shape of `create`. Symmetrical with `findById`/`findOne`/`findMany`, which keep branded entities — the brand is preserved across all read-side outputs.

`PKInput<T>` and `PKOutput<T>` are exported from the repository module (so example apps and tests can reference them), but **not** from the public package barrel — they are consumer-side wiring, not public API. Library users do not construct or reason about `PKInput<User>`; they pass `{ id: 1 }`.

---

## Method signatures

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

### Behavior table

| Call | Compiles? | Reason |
|---|---|---|
| `findById({})` | ✗ | `PKInput<T>` requires PK fields. |
| `findById({ name: 'x' })` | ✗ | wrong field. |
| `findById({ id: 1 })` | ✓ | plain number accepted; brand stripped on input. |
| `delete({ id: 1 })` | ✓ | mirrors `findById`. |
| `update({ name: 'x' })` (autogen `id`) | ✗ | `PKInput<T>` requires `id` even if `T['id']` is optional. |
| `update({ id: 1, name: 'x', email: 'y' })` | ✓ | full entity, PK present. |
| `create({ name: 'x', email: 'y' })` (autogen `id`) | ✓ | `UnbrandedT<T>['id']` keeps optional modifier from `T`. |
| `create({ id: 1, name: 'x', email: 'y' })` | ✓ | override-wins for autogen PK; plain number accepted. |
| `const u = await repo.findById({ id: 1 }); await repo.update(u!)` | ✓ | branded `T` is a subtype of `UnbrandedT<T>`. |
| `const { id } = await repo.create({...}); await repo.findById({ id })` | ✓ | `id` from `PKOutput<T>` is `PrimaryKey<number>`; `findById` accepts plain `number`, and brand is a subtype. |

For `OrderItem` with composite PK `{ orderId, productId }`, `findById({ orderId: 1 })` is a compile error — `PKInput<T>` requires *all* PK fields.

---

## Sub-decisions worth recording

### `update()` keeps a runtime PK defense even with the new types

The new signature `update(entity: UnbrandedT<T> & PKInput<T>)` makes PK fields required and non-undefined at compile time. But callers can bypass types with casts (`update(data as User)`). The body adds `this.requirePrimaryKey(entity)` at the top so cast-bypassed callers get a loud `IncompletePrimaryKeyError` instead of the silent `WHERE id = NULL` no-op.

The compile-time enforcement is the headline; the runtime defense is the safety net for the boundary cases.

### `create()` returns `PKOutput<T>` (branded), not `PKInput<T>` (unbranded)

`create()` previously returned `Partial<T>` populated with PK fields only. The new return is `PKOutput<T>` — the same PK shape as `findById`/`findOne`, with the brand preserved. This keeps the read-side symmetric: every method that hands an entity (or PK-shaped subset of one) back to the caller hands it back branded. The destructured `id` is `PrimaryKey<number>`, which is assignable to `number` anywhere a plain number is expected, so all consumer call sites work identically — and the `id!` non-null assertion in patterns like `repo.findById({ id: id! })` becomes redundant (since `PKOutput<T>` makes `id` non-undefined).

### `requirePrimaryKey`'s parameter must widen, not stay at `Partial<T>`

The internal helper used to be typed `(key: Partial<T>, ...)`. After the changes, callers pass `PKInput<T>` (from `findById`/`delete`) or `UnbrandedT<T> & PKInput<T>` (from `update`). Neither is assignable to `Partial<T>`: `PKInput<T>['id']` is unbranded `number`, `Partial<T>['id']` is `PrimaryKey<number> | undefined`, and the brand asymmetry (plain `number` is not assignable to `PrimaryKey<number>`) blocks the widening. The helper's parameter type was widened to `PKInput<T> | UnbrandedT<T>` to make the call sites typecheck.

### `Conditions<T>` consumes `Unbrand<T[K]>`, not `T[K]`

The query-builder's per-field condition builder used to be typed `FieldConditionBuilder<T[K]>`. With branded PK fields, that meant `c.id?.eq(1)` would reject the plain literal `1` and demand a `PrimaryKey<number>`. The fix is straightforward: `Conditions<T>` is now `{ [K in keyof T]?: FieldConditionBuilder<Unbrand<T[K]>> }`. At runtime the brand is erased, so SQL comparison doesn't care; at the type level, users write `c.id?.eq(1)` without ceremony.

This also lets the `findById` body avoid an internal cast — it passes the unbranded `key[prop]` straight into `eq` without an `as T[keyof T]` workaround.

### `findMany` and `findOne` are not touched

Their inputs are `FindOptions<T>` (a fluent query builder), not raw entity shapes. PK awareness doesn't reach them. The `Conditions<T>` change above is the only ripple; the method signatures stay as they were.

---

## Out of scope (deliberately)

- **`pk<V>(v: V): PrimaryKey<V>` casting helper.** Most paths go through repo methods which strip brands on input. Manual brand construction is rare; users can `as PrimaryKey<V>` if they need to. Defer until proven painful.
- **Branding non-PK fields** (e.g. `Email<string>`, `UserId<string>`). Different design space; this work is about PK awareness only.
- **Decorator that injects the brand automatically.** Structurally impossible — Stage-3 decorators can't modify a field's declared type.
- **Promoting `PKInput<T>`, `UnbrandedT<T>`, or `Unbrand<V>` to the public barrel.** Internal until a user asks. The public surface stays at `PrimaryKey<V>` plus the four method signatures.

---

## Connections

- [[Repository Pattern]] — the surface this work tightens.
- [[Autogeneration]] — the upstream half of the same compile-time direction; the autogeneration mechanism is what made the silent-`update` bug structurally possible (autogen PKs declared `id?: number`).
- [[ECMAScript Stage-3 Decorators]] — the decorator constraint-flip pattern is what enforces the brand at the declaration site.
- [[Conditions Proxy]] — the `Conditions<T>` change (`Unbrand<T[K]>` in the field-condition builder) lives here.
