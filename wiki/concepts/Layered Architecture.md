---
type: concept
title: "Layered Architecture"
complexity: foundational
domain: "OOR architecture"
aliases:
  - "Layered architecture (OOR)"
  - "Five-layer stack"
  - "Layering rule"
created: 2026-04-29
updated: 2026-04-29
tags:
  - concept
  - architecture
status: stable
related:
  - "[[MetadataStorage]]"
  - "[[Database]]"
  - "[[Repository Pattern]]"
  - "[[Lazy Query Builder]]"
  - "[[sources/architecture-overview]]"
sources:
  - "../.raw/architecture-overview.md"
---

# Layered Architecture

## Definition

OOR is organized as **five strict layers**. Dependencies between layers always point downward — a layer never imports from a layer above it. This is the load-bearing organizational rule of the codebase, not a stylistic preference.

```
@Entity / @Column / @PrimaryColumn / @ToOne / @Nullable    ← decorators
                       │  write to
                       ▼
              MetadataStorage (Map<Constructor, EntityMetadata>)
                       │  read by
                       ▼
                 Repository<T>
                       │  delegates composed reads to
                       ▼
                 QueryBuilder<T>
                       │  executes via
                       ▼
              Database (Bun SQL connection)
```

## How It Works

Each layer has a single, narrow responsibility:

| Layer | Responsibility | Module path |
|---|---|---|
| Decorators | Describe entities, columns, relations, nullability | `src/decorators/` |
| MetadataStorage | Hold resolved entity descriptions, per `Database` | `src/core/metadata/` |
| Repository | Entity-shaped CRUD (`create`, `update`, `delete`, `findById`); read delegation | `src/core/repository/` |
| QueryBuilder | Composed reads: `where` callback, conditions proxy, SQL composition | `src/query-builder/` |
| Database | Bun `SQL` connection, metadata host, schema `create()`/`drop()` | `src/core/database/` |

Four cross-cutting properties hold:

1. **Decorators are the only writers of metadata.** Nothing outside `src/decorators/` calls `MetadataStorage.set`.
2. **Metadata freezes after class evaluation.** Once the entity classes have been loaded, the metadata is immutable for the rest of the process.
3. **The repository never builds SQL on its own for read paths.** `findOne`, `findMany`, `findById` all construct a `QueryBuilder` and delegate. Writes are the exception.
4. **Bun's `SQL` is the only thing that touches the wire.** Nothing in OOR speaks the PostgreSQL protocol directly.

## Why It Matters

- **Reasoning is local.** Pick a layer; you only need the layers below to understand it. A `QueryBuilder` change cannot be invalidated by a `Repository` refactor.
- **Test isolation is cheap.** Layers below the one under test are easy to fake (the metadata is just a `Map`; `Database` is just a connection wrapper).
- **Refactor blast radius is bounded.** A breaking change to `MetadataStorage` ripples up through `Repository` and `QueryBuilder` but never sideways into another `core/` module — there is no "sideways" to ripple to.
- **The dependency direction enforces the design.** "Decorators write metadata, others read" is a property of the codebase, not just a convention — it's mechanically true because the import direction makes the alternative impossible to compile.

## The Layering Rule (operational form)

When asking "where does this code belong?":

- **Writes metadata?** It must live under `src/decorators/`.
- **Reads metadata to do something?** It belongs in `core/` (anything entity-shaped) or `query-builder/` (anything composed).
- **Talks to the wire?** It goes through `Database`'s Bun `SQL` handle. Nothing else.

Nothing under `core/metadata/` is allowed to import from above. The compile-time errors enforce the discipline.

## Examples

A real instance of the rule keeping the codebase honest: when adding a new column type, the `COLUMN_TYPE` enum extension lives in `src/core/sql-types/`, the SQL fragment mapping for it lives there too — but the *decorators* that surface it (`@Column({ type: COLUMN_TYPE.JSONB })`) live in `src/decorators/` and import the enum from below. The reverse import (decorators → sql-types) is allowed; sql-types → decorators would not be.

## Connections

- [[MetadataStorage]] — layer 2.
- [[Database]] — layer 5 (the wire edge).
- [[Repository Pattern]] — layer 3.
- [[Lazy Query Builder]] — layer 4.
- [[ECMAScript Stage-3 Decorators]] — layer 1's underlying language feature.

## Sources

- `.raw/architecture-overview.md` §§ "The Layered View", "Source Tree, by Concern"
