---
type: decision
title: "ADR 0008 — PK-aware compile-time enforcement via PrimaryKey<V> brand"
status: accepted
date: 2026-04-30
deciders:
  - "Tony Albert"
supersedes: []
superseded_by: ""
context: "After create-required-fields shipped, the other PK-using Repository methods kept loose contracts; one (update) had a silent runtime bug. Compile-time PK awareness needed a structural mechanism."
created: 2026-04-30
updated: 2026-04-30
tags:
  - decision
  - adr
  - typescript
  - primary-key
  - repository
related:
  - "[[Repository]]"
  - "[[Repository Pattern]]"
  - "[[PrimaryKey Brand]]"
  - "[[Autogeneration]]"
  - "[[ECMAScript Stage-3 Decorators]]"
  - "[[0005-no-any-type-driven-api]]"
  - "[[sources/pk-aware-repository-methods]]"
sources:
  - "../.raw/pk-aware-repository-methods.md"
---

# ADR 0008 — PK-aware compile-time enforcement via `PrimaryKey<V>` brand

## Context

The earlier work that tightened `Repository.create(entity: T)` to enforce required-field correctness at compile time left three of the remaining four PK-using methods in a worse state than they should have been:

- `findById(key: Partial<T>)` and `delete(key: Partial<T>)` accepted **any** subset of `T`. Loose at the type level; loud at runtime via `IncompletePrimaryKeyError`.
- `update(entity: T)` was a **silent runtime bug**: with autogeneration, autogen PKs are declared `id?: number` — optional in `T`. So `T` itself allowed `update({ name: 'x' })`. The body built `WHERE id = NULL`, matched no rows, returned silently.

`findMany(options?)` and `findOne(options?)` were unaffected — they take a query-builder, not a raw entity shape.

Compile-time PK awareness is structurally constrained: TypeScript has no way to know which fields of `T` are PKs. The decorator stores PK-ness in **runtime** metadata. Stage-3 decorators can READ a field's declared type but **cannot inject** type information into it. So the source-of-truth for PK identity has to come from one of two places: (1) a marker the user writes on PK fields' declared types, or (2) a generic the user threads at `Repository` construction. There is no third option.

## Decision

**Adopt Option A: a structural brand on the field type, `PrimaryKey<V>`.**

```ts
declare const __pkBrand: unique symbol;
export type PrimaryKey<V> = V & { readonly [__pkBrand]: true };
export type Unbrand<V> = V extends PrimaryKey<infer U> ? U : V;
```

Every primary-key field is declared with the brand: `id!: PrimaryKey<number>` (or `id?: PrimaryKey<number>` for autogen). `@PrimaryColumn`'s two overloads gain a brand term in their `ClassFieldDecoratorContext` constraint — declaring a `@PrimaryColumn` on an unbranded field is now a decorator-call-site type error.

Repository signatures consume the brand to derive PK shapes:

```ts
findById(key: PKInput<T>):                Promise<T | null>;
delete(key: PKInput<T>):                  Promise<void>;
update(entity: UnbrandedT<T> & PKInput<T>): Promise<void>;
create(entity: UnbrandedT<T>):            Promise<PKOutput<T>>;
```

See [[PrimaryKey Brand]] for the full type machinery and [[Repository]] for the per-method behavior tables.

## Consequences

### Positive

- **Compile-time PK enforcement on every PK-using method.** `findById({})`, `findById({ name: 'x' })`, and `update({ name: 'x' })` (on autogen entities) all become compile errors. The silent `update` bug is closed at the type level.
- **No runtime / compile-time disagreement.** The brand is checked at the `@PrimaryColumn` declaration site, so the field's type and the decorator's metadata cannot drift apart.
- **Brand-free call sites.** Brand asymmetry (subtype on the way out, supertype on the way in via `Unbrand<V>`) keeps user code looking exactly like before — `repo.findById({ id: 1 })`, `c.id?.eq(1)`. The brand only shows up at the **declaration** site.
- **Symmetric read-side.** `findById`, `findOne`, `findMany`, and `create` all return branded entities (or `PKOutput<T>`). Round-trips like `repo.update(await repo.findById({ id }))` work without casts.
- **Consistent with the existing direction.** Finishes the compile-time enforcement that `create(entity: T)` started.

### Negative / trade-offs

- **Migration cost: every PK declaration changes.** ~15-25 sites in this codebase, all in entities or test fixtures. Search-and-replace migration with no logic changes.
- **`PrimaryKey<V>` is visible at the declaration site** and can be confused with a runtime wrapper. It isn't — the brand is purely a type-level construct. Documentation has to compensate.
- **Internal type plumbing is heavier.** `PKKeys<T>`, `UnbrandedT<T>`, `PKInput<T>`, `PKOutput<T>` are non-trivial mapped types; their interaction with `NonNullable<>` and optional-modifier preservation has to be reasoned about carefully. (`requirePrimaryKey`'s parameter type had to widen from `Partial<T>` to `PKInput<T> | UnbrandedT<T>` because the new shapes weren't assignable to `Partial<T>`.)

### Neutral

- **Runtime `requirePrimaryKey` is still in place.** The headline is the compile-time enforcement; the runtime gate is the floor for cast-bypassed callers (`update(data as User)`).
- **Public surface stays minimal.** `PrimaryKey<V>` ships in the public barrel; `PKInput<T>`, `PKOutput<T>`, `UnbrandedT<T>`, `Unbrand<V>` are internal until proven needed.

## Alternatives Considered

### B. Generic `Repository<T, PK extends keyof T>`

User threads PK keys explicitly: `new Repository<User, 'id'>(User, db)`.

- **Pros:** no entity-declaration changes.
- **Cons:** verbose at construction; **TypeScript won't catch a mismatch between the `PK` generic and the `@PrimaryColumn` decorator metadata**. Declaring `Repository<User, 'name'>` would typecheck silently and disagree with the runtime PK check. **For a publishable library, that's an unacceptable footgun.**

This was the disqualifying argument: a library author cannot rely on users to keep two source-of-truth declarations in sync when the type system can't help.

### C. Runtime-only tightening

Don't change types. Add a stricter runtime check at the start of `update()` that rejects missing-or-`undefined` PK values.

- **Pros:** smallest change, no API impact, no migration.
- **Cons:** the type system gains nothing — `findById({})` still compiles; `update({ name: 'x' })` still compiles. Closes the silent-bug case but is **a step backward** from the type-precision direction the previous work set. Inconsistent with the existing investment in compile-time guarantees.

### Why A wins

Two arguments converged on the brand:

1. **Consistency with the existing direction.** The prior work paid a real migration cost for compile-time correctness; finishing the job is a small marginal step.
2. **B's footgun is unacceptable.** The runtime/compile-time disagreement in B is exactly the failure mode A's declaration-site enforcement prevents.

The cost of A — `id!: PrimaryKey<number>` at every PK declaration — is bounded and mechanical.

## References

- `.raw/pk-aware-repository-methods.md` — the design memo behind this decision.
- [[sources/pk-aware-repository-methods]] — the synthesis.
- [[PrimaryKey Brand]] — the concept page.
- [[Repository]] — the four updated method signatures.
- [[0005-no-any-type-driven-api]] — the strict-typing ADR this builds on.
