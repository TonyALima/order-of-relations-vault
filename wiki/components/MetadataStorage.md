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

```ts
class MetadataStorage {
  private storage = new Map<Constructor, EntityMetadata>();
  private isMetadataResolved = false;
  // ...
}
```

A single `Map<Constructor, EntityMetadata>`, keyed by the class constructor itself, plus a resolution flag. Keyed by **reference identity** on the constructor, not by name string.

`EntityMetadata` aggregates exactly three things:

- `tableName: string` — defaults to the class name; resolved against the prototype chain on first read (see [[Single-Table Inheritance]]).
- `columns: ColumnMetadata[]` — one entry per `@Column` / `@PrimaryColumn`. Each carries property name, mapped column name, SQL type, primary flag, nullability, and an optional `autogeneration` strategy.
- `relations: RelationMetadata[]` — one entry per `@ToOne`. Each carries property name, relation type (`TO_ONE`), nullability, FK columns (resolved lazily), and a `getTarget()` thunk (see [[Relation Target Thunk]]).

Schema names, indexes, cascading rules: deliberately absent. They'd belong on the same record but aren't needed yet — adding fields later is cheaper than removing.

### Why `Map`, not per-class statics or `WeakMap`

- **Per-class statics** would scatter metadata across user code. The `Map` keeps ownership inside the library.
- **`Symbol.iterator`** is part of the public surface — schema generators and the inheritance resolver both walk all registered entities. Per-class statics make iteration an extra primitive.
- **`WeakMap` would be a bug.** OOR explicitly wants entity classes alive for the connection's lifetime — the registry *is* the schema. A weak reference would let entities collect under GC pressure.

## Ownership

> [!key-insight] One per `Database`, not one per process
> `MetadataStorage` is owned by a `Database` instance. `@Entity(db)` takes the database as its argument and registers the entity into **that database's** storage. Two `Database` instances in the same process means two independent metadata maps.
>
> This refines the welcome-doc claim that "metadata is stored in a custom `metadataStorage` `Map` owned by the library" — the library doesn't own a singleton; each `Database` does.

## Write Path (decorators only)

Nothing outside `src/decorators/` calls `MetadataStorage.set`. The write path uses **three** symbol keys on Stage-3's `context.metadata` bag, each with a different shape:

| Key | Shape | Written by | Read by |
|---|---|---|---|
| `COLUMNS_KEY` | `ColumnMetadata[]` | `@Column`, `@PrimaryColumn` | `@Entity` |
| `RELATIONS_KEY` | `RelationMetadata[]` | `@ToOne` | `@Entity` |
| `NULLABLE_KEY` | `Map<string, boolean>` | `@Nullable`, `@NotNullable` | `@Column` (peer-to-peer; never reaches storage) |

The flow:

1. Field decorators run (in source-bottom-up order; the inner decorator runs first). `@Nullable` / `@NotNullable` set `NULLABLE_KEY[propertyName] = true | false`. `@Column` then reads that entry — and throws `MissingNullabilityDecoratorError` if it's missing — before pushing the `ColumnMetadata` (with the resolved `nullable` field baked in) onto `COLUMNS_KEY`. `@PrimaryColumn` skips the `NULLABLE_KEY` check and forces `nullable: false`.
2. `@Entity(db, mapTableName?)` runs last (class decorators run after field decorators per the language spec). It pulls `COLUMNS_KEY` and `RELATIONS_KEY` out of `context.metadata` (it ignores `NULLABLE_KEY` — already consumed), asserts at least one column is `primary` (`MissingPrimaryColumnError` if not), builds an `EntityMetadata`, and writes it into `db.getMetadata()`.

See the full [[entity-registration]] flow for the per-step sequence and the order-matters constraint.

After all class declarations have been evaluated, the storage is **frozen for the process lifetime** — no other code path mutates it.

> [!note] Resolved 2026-04-29 — the third symbol exists
> An earlier version of this page claimed only **two** symbol keys, citing `.raw/decorator-metadata-storage.md`. A code spot-check against `[[order-of-relations]]` (`src/decorators/nullable/nullable.ts` line 1, `src/decorators/column/column.ts` lines 31–37) confirmed `NULLABLE_KEY` exists and is enforced by `@Column`. `decorator-metadata-storage.md` had elided it because the source's scope was "what `MetadataStorage` stores"; `NULLABLE_KEY` never reaches storage. `architecture-overview.md` was right. See [[sources/decorator-metadata-storage]] § "Resolved disagreement".

## Read Path & Lazy Resolution

`isMetadataResolved` is the laziness flag. Every `set()` resets it to `false`. The next `get()` (or iteration) triggers two resolution passes:

- **`resolveInheritance`** — for each registered class, walks `Object.getPrototypeOf(target.prototype)?.constructor` up to the topmost decorated ancestor and adopts its `tableName`. Multi-level chains collapse to one table. Discriminators are assigned per class, then **wiped** if only one class maps to a given table; a sibling registering later restores them on the next resolution pass. See [[Single-Table Inheritance]].
- **`resolveRelations`** — for each `RelationMetadata`, calls its `getTarget()` thunk (see [[Relation Target Thunk]]) and looks up the result in the storage. If the target isn't registered, throws `RelationTargetNotFoundError`. Foreign-key column names that decorators left as `null` are filled in here.

After the passes, `isMetadataResolved = true`. Subsequent reads hit the resolved cache.

The flag-and-reset design makes resolution **idempotent across late additions** — registering a sibling class doesn't corrupt previously-resolved entries; the next read recomputes everything that needs to change. The "re-resolving after a new entity is added does not corrupt discriminator" test in `metadata.test.ts` pins this contract.

## Why it matters

- **Decorators stay narrow.** They write per-class data into `context.metadata`, never directly into `MetadataStorage`. Only `@Entity` flushes the buffer.
- **Resolution is lazy and idempotent.** Decorators don't need to reason about declaration order; resolution happens at first read after all classes are loaded, and re-runs cleanly on late additions.
- **Per-`Database` ownership** makes test isolation trivial: spin up a fresh `Database`, register entities against it, and the rest of the process is untouched. Tests build fresh storages constantly — a process-global singleton would make that miserable.

## Failure Modes

| Where | Error | Notes |
|---|---|---|
| `@Entity` | `MissingPrimaryColumnError` | Thrown at decoration time if no column has `primary: true`. Class never reaches storage. |
| `@Column` (reading `NULLABLE_KEY`) | `MissingNullabilityDecoratorError` | Thrown at decoration time if `@Nullable` / `@NotNullable` wasn't applied first (i.e., wasn't the inner decorator). Lives in `src/decorators/nullable/nullable.errors.ts`. `@PrimaryColumn` is exempt. |
| `resolveRelations` | `RelationTargetNotFoundError` | Thrown when a `getTarget()` thunk returns a constructor that was never registered. The error carries the target class name and the relation path (e.g., `posts.author`). The only error type defined in `metadata.errors.ts`. |
| `storage.get(UnregisteredClass)` | *(none — returns `undefined`)* | The metadata layer **deliberately** does not throw here. "No entry" can mean "forgot `@Entity`" or "passed the wrong class"; the metadata layer can't disambiguate. Translation to a user-facing error is the repository's job. |

## Connections

- [[Database]] — owns the storage.
- [[ECMAScript Stage-3 Decorators]] — the language feature whose `context.metadata` bag serves as the write-time buffer.
- [[entity-registration]] — the flow whose final step is this storage's `set()`.
- [[Single-Table Inheritance]] — what `resolveInheritance` produces.
- [[Relation Target Thunk]] — the closure pattern `resolveRelations` invokes.
- [[Layered Architecture]] — the dependency-direction rule that says nothing under `core/metadata/` imports from above.
- [[query-lifecycle]] — Step 2 ("Metadata resolution") is exactly this read path.

## Sources

- `.raw/architecture-overview.md` §§ "The Layered View", "How Decorators Talk to the Rest of the System"
- `.raw/decorator-metadata-storage.md` (the pin-down: storage shape, two-symbol scheme, `isMetadataResolved` flag, failure modes)
