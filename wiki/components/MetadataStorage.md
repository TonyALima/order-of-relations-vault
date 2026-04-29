---
type: component
title: "MetadataStorage"
component_kind: class
module: "src/core/metadata/"
exports_as: "MetadataStorage"
created: 2026-04-29
updated: 2026-04-29
tags:
  - component
  - metadata
  - core
status: stable
related:
  - "[[Database]]"
  - "[[ECMAScript Stage-3 Decorators]]"
  - "[[Layered Architecture]]"
  - "[[sources/architecture-overview]]"
sources:
  - "../.raw/architecture-overview.md"
---

# MetadataStorage

## Purpose

`MetadataStorage` holds the resolved entity descriptions for a single `Database` — the structured form of what the decorators write. It is the **only** layer that the [[Repository Pattern|Repository]] and [[Lazy Query Builder|QueryBuilder]] read from when planning queries or composing DDL.

## Shape

A typed `Map`:

```ts
type MetadataStorage = Map<EntityConstructor, EntityMetadata>;
```

Where `EntityMetadata` aggregates:

- `tableName: string` — resolved from class name + inheritance pass.
- `columns: ColumnMetadata[]` — one entry per `@Column` / `@PrimaryColumn`.
- `relations: RelationMetadata[]` — one entry per `@ToOne` / etc.
- `discriminator?: DiscriminatorMetadata` — populated only for class-table-inheritance entities.

## Ownership

> [!key-insight] One per `Database`, not one per process
> `MetadataStorage` is owned by a `Database` instance. `@Entity(db)` takes the database as its argument and registers the entity into **that database's** storage. Two `Database` instances in the same process means two independent metadata maps.
>
> This refines the welcome-doc claim that "metadata is stored in a custom `metadataStorage` `Map` owned by the library" — the library doesn't own a singleton; each `Database` does.

## Write Path (decorators only)

Nothing outside `src/decorators/` calls `MetadataStorage.set`. The write path:

1. Field decorators (`@Column`, `@PrimaryColumn`, `@ToOne`, `@Nullable`, `@NotNullable`) push into `context.metadata` — Stage-3's **per-class** metadata bag — under private symbol keys (`COLUMNS_KEY`, `RELATIONS_KEY`, `NULLABLE_KEY`).
2. `@Entity(db)` runs last (class decorators run after field decorators per the language spec). It pulls the accumulated arrays out of `context.metadata`, asserts at least one column is primary, builds an `EntityMetadata`, and writes it into `db.getMetadata()`.

After all class declarations have been evaluated, the storage is **frozen for the process lifetime** — no other code path mutates it.

## Read Path

Read on first access for an entity triggers two resolution passes:

- `resolveInheritance` — computes `tableName` and `discriminator` for each entity, taking inheritance chains into account.
- `resolveRelations` — fills in foreign-key column names that relation decorators left as `null` (because the referenced entity might not have been declared yet at decoration time).

Subsequent reads hit the resolved cache.

## Why it matters

- **Decorators stay narrow.** They write per-class data into `context.metadata`, never directly into `MetadataStorage`. Only `@Entity` flushes the buffer.
- **Resolution is lazy and once-only.** Decorators don't need to reason about declaration order; resolution happens at first read after all classes are loaded.
- **Per-`Database` ownership** makes test isolation trivial: spin up a fresh `Database`, register entities against it, and the rest of the process is untouched.

## Connections

- [[Database]] — owns the storage.
- [[ECMAScript Stage-3 Decorators]] — the language feature whose `context.metadata` bag serves as the write-time buffer.
- [[Layered Architecture]] — the dependency-direction rule that says nothing under `core/metadata/` imports from above.
- [[query-lifecycle]] — Step 2 ("Metadata resolution") is exactly this read path.

## Sources

- `.raw/architecture-overview.md` §§ "The Layered View", "How Decorators Talk to the Rest of the System"
