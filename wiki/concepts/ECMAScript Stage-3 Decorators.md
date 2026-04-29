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

In OOR, the absence of `Reflect.metadata` is filled by an in-library `metadataStorage` `Map`. Each `@Entity` / `@Column` writes into that map under a key derived from the class constructor; the `Repository` reads back from it.

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
- [[Decorator Metadata Storage]] — the in-library `Map` that replaces `Reflect.metadata` (page populated by a future ingest).
- [[TypeScript]] — host language; Stage-3 decorators require TS 5.0+.

## Sources

- `.raw/welcome.md` § "Stage-3 decorators, no `reflect-metadata`"
