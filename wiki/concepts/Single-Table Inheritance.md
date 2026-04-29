---
type: concept
title: "Single-Table Inheritance"
complexity: intermediate
domain: "ORM design"
aliases:
  - "STI"
  - "Class-table inheritance (single-table flavor)"
  - "Discriminator-based inheritance"
created: 2026-04-29
updated: 2026-04-29
tags:
  - concept
  - inheritance
  - metadata
status: stable
related:
  - "[[MetadataStorage]]"
  - "[[entity-registration]]"
  - "[[sources/decorator-metadata-storage]]"
sources:
  - "../.raw/decorator-metadata-storage.md"
---

# Single-Table Inheritance

## Definition

**Single-table inheritance (STI)** maps a class hierarchy to a *single* SQL table, distinguishing rows by a **discriminator** column. In OOR, STI emerges automatically from prototype chains: subclasses inherit the parent's table name, and discriminators are assigned (or wiped) based on whether sibling classes exist.

Resolution happens lazily inside [[MetadataStorage]] on the first read after a `set()`.

## How It Works

When [[MetadataStorage]] resolves inheritance:

1. **Walk the prototype chain.** For each registered constructor `C`, follow `Object.getPrototypeOf(C.prototype)?.constructor` upward until reaching a class that is **not** decorated with `@Entity`. The deepest decorated ancestor is the **root** of the hierarchy.
2. **Adopt the root's table name.** Every class in the chain — `Base ← User ← AdminUser` — gets the table name of the topmost decorated class. Multi-level chains collapse to a single table.
3. **Assign a discriminator.** By default each class's discriminator is *its own declared table name* (the value `mapTableName` resolved to in `@Entity`).
4. **Wipe singletons.** If only one class maps to a given table, its discriminator is cleared — no discriminator column is needed when there's nothing to discriminate. As soon as a sibling class registers and resolution re-runs, both classes get their discriminators back.

Resolution is gated by `isMetadataResolved`. The flag is reset on every `set()`, so adding a new entity invalidates the cache and the next read recomputes — idempotently. The "re-resolving after a new entity is added does not corrupt discriminator" test in `metadata.test.ts` pins this contract.

## Why It Matters

- **Shared schema, distinct types.** `User` and `AdminUser` live in the same `users` table; queries against either type project to the same rows, distinguished by the discriminator.
- **No hierarchy walking at query time.** Resolution happens once (per registration delta); the query builder reads a fully-resolved `EntityMetadata` and doesn't repeat the prototype walk.
- **Column inheritance is free.** Stage-3 doesn't merge column arrays across the hierarchy — but the parent's metadata is already in storage under the parent constructor, so the inheritance resolver just reads it from there. No duplication, no synthesis.
- **Idempotent under late additions.** Registering a sibling later in the load sequence doesn't corrupt previously-resolved entries — the flag-and-reset pattern handles it.

## Examples

```ts
@Entity(db) // decorator => table 'User' (or whatever class name)
class User { /* @PrimaryColumn id, @Column email, ... */ }

@Entity(db)
class AdminUser extends User { /* @Column adminLevel */ }
```

After resolution:

| Class | Table | Discriminator |
|---|---|---|
| `User` | `User` | `'User'` |
| `AdminUser` | `User` | `'AdminUser'` |

If only `User` were registered, its discriminator would be **empty** (no sibling to disambiguate). Once `AdminUser` registers and the next read triggers re-resolution, both pick up their discriminators.

For a multi-level chain `Base ← User ← AdminUser`, all three collapse to `Base`'s table; the discriminator on each is its own class name.

## Pitfalls

- **The root must be decorated.** The walk stops at the first non-`@Entity` ancestor. An undecorated `BaseEntity` is treated as "not part of the hierarchy" — `User extends BaseEntity` would map to the `User` table, not `BaseEntity`.
- **Discriminator can flip from absent to present.** A class might have no discriminator in early load order, then gain one once a sibling registers. Code that caches `EntityMetadata` outside `MetadataStorage`'s resolution loop will see stale data.
- **Column collisions across siblings are not detected here.** `User.email` and `AdminUser.email` would both write to the `users.email` column with no merge logic — the metadata layer doesn't validate column uniqueness across siblings.

## Connections

- [[MetadataStorage]] — owns the resolution flag and the storage `Map`.
- [[entity-registration]] — the flow whose Step 4 (first read) triggers this resolution.
- [[ECMAScript Stage-3 Decorators]] — `context.metadata` is *not* inherited across subclass declarations; this is what makes the prototype walk necessary at resolution time rather than at decoration time.

## Sources

- `.raw/decorator-metadata-storage.md` § "Inheritance"
