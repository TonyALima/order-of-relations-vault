---
type: source
title: "Drift D3 — FindOptions.inheritance is undocumented"
source_type: drift-correction
author: "Tony Albert"
date_published: 2026-04-29
url: ""
confidence: high
key_claims:
  - "FindOptions<T> carries a second option besides `where` — `inheritance: InheritanceSearchType`."
  - "InheritanceSearchType has three values: ALL (default, no filter), ONLY (= self discriminator), SUBCLASSES (IN list of self + descendants)."
  - "Subclass discovery uses Object.prototype.isPrototypeOf walk against the live metadata Map."
  - "The discriminator filter is only added when meta.discriminator is truthy; resolveInheritance wipes it for singletons."
  - "applyOptions runs the where callback first (replaces conditions), then inheritance pushes onto the same array — discriminator AND user conditions, but a second applyOptions clobbers it."
created: 2026-04-29
updated: 2026-04-29
tags:
  - source
  - drift
status: stable
related:
  - "[[QueryBuilder]]"
  - "[[Single-Table Inheritance]]"
  - "[[Lazy Query Builder]]"
  - "[[query-lifecycle]]"
sources:
  - "../.raw/drift-d3-find-options-inheritance.md"
---

# Drift D3 — FindOptions.inheritance is undocumented

## Summary

The largest gap noticed in the wiki: `FindOptions<T>` carries a second option besides `where` — `inheritance: InheritanceSearchType` — which controls how reads scope across a single-table-inheritance hierarchy. The option is shipped, exported, used by the inheritance example, and previously unmentioned anywhere in `wiki/`.

## What the Code Actually Exposes

`src/query-builder/types.ts`:

```ts
export enum InheritanceSearchType {
  ALL = 'ALL',
  ONLY = 'ONLY',
  SUBCLASSES = 'SUBCLASSES',
}

export interface FindOptions<T> {
  where?: (conditions: Conditions<T>) => (Condition | undefined)[];
  inheritance?: InheritanceSearchType;
}
```

`src/query-builder/query-builder.ts` consumes it inside `applyOptions`:

```ts
if (options?.inheritance === InheritanceSearchType.SUBCLASSES) {
  this.setSubClassesDiscriminator();
} else if (options?.inheritance === InheritanceSearchType.ONLY) {
  this.setConcreteClassDiscriminator();
}
```

## Semantics of Each Value

- **`ALL`** (default) — no discriminator filter; every row in the inherited table is returned, regardless of subclass. The "give me everything" escape hatch.
- **`ONLY`** — adds `discriminator = <self>` predicate. Returns only rows that map exactly to `T`.
- **`SUBCLASSES`** — adds `discriminator IN (...)` whose value list is `T` and every descendant in the prototype chain. Discovery via `Object.prototype.isPrototypeOf.call(this.entity.prototype, t.prototype)`.

## Discriminator-Only-When-Needed

The discriminator filter is only added when `meta.discriminator` is truthy. `MetadataStorage.resolveInheritance` (see [[Single-Table Inheritance]]) wipes the discriminator on entities alone in their table. So calling `userRepo.findMany({ inheritance: ONLY })` against a `User` with no subclasses is a no-op at SQL level — until a sibling registers, at which point the same call starts emitting the predicate. The option's effect depends on whether siblings exist, not just on what the caller passes.

## Interaction with `where`

`applyOptions` runs the `where` callback first (which **replaces** `this.conditions` wholesale — see [[apply-options-accumulation]]), then the inheritance branch **pushes** onto the same array. Net effect: when both are passed, the discriminator condition gets ANDed onto user conditions. But if `applyOptions` is called twice — once with `inheritance`, once with `where` — the second `where` clobbers the discriminator. Real footgun.

## Why the Drift Mattered

- The inheritance example is the *single most prominent* use of the API surface that the wiki didn't cover.
- [[Single-Table Inheritance]] documented the metadata side but stopped there. The user-visible "how do I read across a hierarchy?" question had no answer.
- The closed `InheritanceSearchType` enum is exactly the kind of "type-driven public API" that [[0005-no-any-type-driven-api]] enforces.

## Resolution

- [[QueryBuilder]] — `applyOptions()` section now documents the `inheritance` option, the three values, and the two private-but-actually-public helpers.
- [[Single-Table Inheritance]] — added a "Reading across the hierarchy" section.
- [[Lazy Query Builder]] — forward-link to the new sections.
- [[query-lifecycle]] — step describing inheritance-condition push.
- [[brief]] — bullet documenting `InheritanceSearchType`.

## Source Citations

- `src/query-builder/types.ts` lines 23–32 — enum + FindOptions shape.
- `src/query-builder/query-builder.ts` lines 15–62 — setSubClassesDiscriminator / setConcreteClassDiscriminator + applyOptions branching.
- `src/core/metadata/metadata.ts` lines 58–82 — resolveInheritance wipe rule.
- `examples/inheritance/services/UserHierarchyService.ts` — canonical user-facing call site.
