---
type: concept
title: "ECMAScript Stage-3 Decorators"
complexity: intermediate
domain: "TypeScript / JavaScript language features"
aliases:
  - "Stage-3 decorators"
  - "Standard decorators"
  - "TC39 decorators"
created: 2026-04-29
updated: 2026-04-30
tags:
  - concept
  - decorators
  - typescript
status: seed
related:
  - "[[0001-stage-3-decorators]]"
  - "[[PrimaryKey Brand]]"
  - "[[sources/welcome]]"
  - "[[sources/pk-aware-repository-methods]]"
sources:
  - "../.raw/welcome.md"
  - "../.raw/pk-aware-repository-methods.md"
---

# ECMAScript Stage-3 Decorators

## Definition

The **TC39 Stage-3 decorators proposal** is the standardized successor to TypeScript's legacy `experimentalDecorators`. Stage-3 means the proposal is finalized in spec text and is implemented in shipping engines, ahead of merging into the language spec at Stage 4. A decorator is a function applied with `@decorator` syntax to classes, methods, accessors, fields, and class auto-accessors, receiving a target value and a `context` object describing the binding.

## How It Works

A Stage-3 decorator's signature is `(value, context) => replacement`. The `context` argument carries:

- `kind` — `"class" | "method" | "field" | ...`
- `name` — the property name (or `undefined` for the class itself)
- `static` — whether the binding is on the constructor
- `private` — whether the binding is `#private`
- `addInitializer(fn)` — schedule a callback when the class is initialized
- `access` — getter/setter helpers

This differs sharply from the legacy form, which received `(target, propertyKey, descriptor)` and relied on `Reflect.defineMetadata` to thread metadata between decorators.

In OOR, the absence of `Reflect.metadata` is filled by:

1. **`context.metadata`** — Stage-3's per-class metadata bag, used as a *write-time buffer*. Field decorators push into it under **three** private symbol keys, each with a different shape:
    ```ts
    const COLUMNS_KEY   = Symbol('columns');    // ColumnMetadata[]
    const RELATIONS_KEY = Symbol('relations');  // RelationMetadata[]
    const NULLABLE_KEY  = Symbol('nullable');   // Map<string, boolean>
    ```
   `@Column` / `@PrimaryColumn` push onto `COLUMNS_KEY`. `@ToOne` pushes onto `RELATIONS_KEY`. `@Nullable` and `@NotNullable` set entries on the `NULLABLE_KEY` map (property name → `true` / `false`). The `NULLABLE_KEY` bucket is a **peer-to-peer channel between sibling decorators**: `@Column` reads it at registration time, but `@Entity` does not.
2. **A `Map<Constructor, EntityMetadata>` owned by a [[Database]] instance** — the durable store. The `@Entity(db)` class decorator runs *after* all field decorators (per the language spec), pulls `COLUMNS_KEY` and `RELATIONS_KEY` out of `context.metadata`, validates that at least one column is `primary` (else throws `MissingPrimaryColumnError`), and flushes a finalized `EntityMetadata` into `db.getMetadata()` (see [[MetadataStorage]] and the [[entity-registration]] flow). `NULLABLE_KEY` has already been consumed by the field decorators by this point; it doesn't reach storage.

The `context.metadata` bag is **fresh per class** — Stage-3 does not propagate it across subclass declarations. Inheritance is reconstructed at storage-resolution time by walking the prototype chain (see [[Single-Table Inheritance]]).

> [!warning] Decorator order matters: `@Nullable` must be inner
> Stage-3 decorators are applied **bottom-up** on a field — the decorator closest to the property runs first. Because `@Column` reads `NULLABLE_KEY` and throws `MissingNullabilityDecoratorError` if the property's entry is missing, `@Nullable` (or `@NotNullable`) must populate the bucket *before* `@Column` runs.
>
> ```ts
> // ✅ Works — @Nullable is inner (runs first), then @Column reads its entry.
> @Column({ type: COLUMN_TYPE.TEXT })
> @Nullable
> nickname?: string;
>
> // ❌ Throws MissingNullabilityDecoratorError — @Column is inner (runs first),
> //    NULLABLE_KEY has no entry for `nickname` yet.
> @Nullable
> @Column({ type: COLUMN_TYPE.TEXT })
> nickname?: string;
> ```
>
> `@PrimaryColumn` is exempt: it forces `nullable: false` and skips the `NULLABLE_KEY` lookup entirely.
>
> **Open question:** [[decorator-order-independence]] — whether to redesign so both orderings work.

> [!note] History of refinements (2026-04-29)
> This page absorbed three corrections in one day:
> 1. The storage is **per-`Database`**, not library-global. Per `.raw/architecture-overview.md`.
> 2. The symbol-key list was first claimed to be three, then "corrected" to two (per `.raw/decorator-metadata-storage.md`'s narrower framing), then **re-corrected back to three** after a code spot-check (`src/decorators/nullable/nullable.ts` and `src/decorators/column/column.ts` in [[order-of-relations]]) confirmed the third bucket exists. The decorator-metadata-storage source had elided `NULLABLE_KEY` because it focuses on what flows into the storage `Map`; `NULLABLE_KEY` is consumed by `@Column` and never reaches storage. See [[sources/decorator-metadata-storage]] § "Resolved disagreement" for the trail.
> 3. The order constraint above was added on the same code spot-check.

## The constraint-flip pattern (read-only)

Stage-3 decorators **cannot inject type information** into a field — they can only READ the field's declared type via the `ClassFieldDecoratorContext<This, Value>` constraint. OOR exploits this by *narrowing* `Value` in decorator overloads to reject incompatible declarations at the call site:

- `@Nullable` requires `NullableField<V>` → field must be declared with the `?` modifier.
- `@NotNullable` requires `NotNullableField<V>` → field must be declared with the `!` modifier.
- `@PrimaryColumn` (with autogeneration) requires `NullableField<V> & NullablePrimaryKey<V>` → field must be optional **and** carry the [[PrimaryKey Brand|`PrimaryKey<V>` brand]].
- `@PrimaryColumn` (without autogeneration) requires `NotNullableField<V> & PrimaryKey<V>` → field must be required **and** branded.

This is the only structural mechanism for compile-time enforcement available in the dialect. It is why the brand has to live at the declaration site (a decorator cannot add it for you), and it is why the modifier (`?` vs `!`) on the field has to match the autogeneration option on `@PrimaryColumn` — both checks happen at the same decorator call.

See [[PrimaryKey Brand]] § "How decorators enforce the brand" for the resolution table across all six declaration shapes.

## Why It Matters

For OOR specifically:

- It's the future-proof choice — no migration looming when TC39 advances the proposal.
- It eliminates the `reflect-metadata` polyfill dependency for consumers of the published npm package.
- The library owns its metadata layout; behavior doesn't depend on global `Reflect` state shared with whatever else is in the consumer's bundle.

For the broader ecosystem: it ends a multi-year split where TypeScript's decorators were not standard JavaScript. Once Stage-3 lands as Stage-4, `experimentalDecorators` becomes the legacy variant.

## Examples

```ts
// Stage-3 class decorator
function Entity(value: new (...args: unknown[]) => unknown, context: ClassDecoratorContext) {
  metadataStorage.tables.set(value, { name: context.name ?? "" });
  return value;
}

@Entity
class User { /* ... */ }
```

Compare with the legacy form:

```ts
// Legacy — NOT used in OOR
function Entity(target: Function) {
  Reflect.defineMetadata("entity", true, target);
}
```

## Connections

- [[0001-stage-3-decorators]] — the ADR fixing this choice for OOR.
- [[MetadataStorage]] — the per-`Database` `Map` that replaces `Reflect.metadata`.
- [[entity-registration]] — the end-to-end flow turning a decorated class into an `EntityMetadata` entry.
- [[Single-Table Inheritance]] — what reconstructs inheritance at resolution time, since `context.metadata` doesn't propagate.
- [[Relation Target Thunk]] — the closure pattern that breaks circular-reference TDZ at decoration time.
- [[Database]] — the owner of `MetadataStorage`; takes `@Entity(db)` as its registration argument.
- [[TypeScript]] — host language; Stage-3 decorators require TS 5.0+.
- [[PrimaryKey Brand]] — the second use of the constraint-flip pattern (after nullability); enforces the brand at the `@PrimaryColumn` declaration site.

## Sources

- `.raw/welcome.md` § "Stage-3 decorators, no `reflect-metadata`"
- `.raw/pk-aware-repository-methods.md` § "@PrimaryColumn decorator constraint update"
