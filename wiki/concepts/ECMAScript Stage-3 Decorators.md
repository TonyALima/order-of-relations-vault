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
updated: 2026-04-29
tags:
  - concept
  - decorators
  - typescript
status: seed
related:
  - "[[0001-stage-3-decorators]]"
  - "[[sources/welcome]]"
sources:
  - "../.raw/welcome.md"
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

1. **`context.metadata`** — Stage-3's per-class metadata bag, used as a *write-time buffer*. Field decorators (`@Column`, `@PrimaryColumn`, `@ToOne`, `@Nullable`, `@NotNullable`) push into `context.metadata` under private symbol keys (`COLUMNS_KEY`, `RELATIONS_KEY`, `NULLABLE_KEY`).
2. **A `Map<Constructor, EntityMetadata>` owned by a [[Database]] instance** — the durable store. The `@Entity(db)` class decorator runs *after* all field decorators (per the language spec), pulls the buffered arrays out of `context.metadata`, and flushes a finalized `EntityMetadata` into `db.getMetadata()` (see [[MetadataStorage]]).

> [!note] Refined 2026-04-29
> An earlier version of this page said the `Map` was "owned by the library." Per `.raw/architecture-overview.md`, the storage is **per-`Database`**, not per-library — `@Entity(db)` takes the database as an argument. Two databases in the same process means two metadata maps. The library does not own a global singleton.

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
- [[Decorator Metadata Storage]] — deeper symbol-key layout (page populated by a future ingest of `.raw/decorator-metadata-storage.md`).
- [[Database]] — the owner of `MetadataStorage`; takes `@Entity(db)` as its registration argument.
- [[TypeScript]] — host language; Stage-3 decorators require TS 5.0+.

## Sources

- `.raw/welcome.md` § "Stage-3 decorators, no `reflect-metadata`"
