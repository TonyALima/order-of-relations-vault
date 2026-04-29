---
type: component
title: "Database"
component_kind: class
module: "src/core/database/"
exports_as: "Database"
created: 2026-04-29
updated: 2026-04-29
tags:
  - component
  - database
  - core
status: stable
related:
  - "[[MetadataStorage]]"
  - "[[Bun]]"
  - "[[schema-create-drop]]"
  - "[[sources/architecture-overview]]"
sources:
  - "../.raw/architecture-overview.md"
---

# Database

## Purpose

`Database` is the connection-and-metadata host. Every entity is registered against an instance of it (`@Entity(db)`), every repository is constructed against one (`new Repository(User, db)`), and the schema lifecycle is one of its three jobs.

## The Three Jobs

`Database` does exactly three things:

### 1. Connection

Wraps a Bun `SQL` instance. The constructor accepts an optional URL; `connect(url?)` either uses the supplied URL or falls back to Bun's default (which reads `DATABASE_URL` from the environment).

The wrapper is thin: callers don't talk to Bun's `SQL` directly through `Database` — the [[Lazy Query Builder]] and the [[Repository Pattern|Repository]]'s write paths get the `SQL` handle and use it.

### 2. Metadata host

Owns the entity [[MetadataStorage]] for entities registered against this instance. Exposed via `db.getMetadata()`. Two `Database` instances in the same process means two independent metadata maps.

This is the canonical site for the per-`Database`-not-per-library ownership rule.

### 3. Schema lifecycle

`create()` materializes the schema in two passes (see [[schema-create-drop]]):

1. Emit `CREATE TABLE` for each entity (columns + primary keys, no FKs).
2. Emit `ALTER TABLE … ADD FOREIGN KEY` for each relation.

`drop()` walks the relation graph in topological reverse so foreign-key targets are dropped after their referrers.

> Migrations beyond create/drop are out of scope for this class — see `[[Schema Migrations]]`.

## Constructor & Lifecycle

Typical usage:

```ts
export const db = new Database();
await db.connect(); // uses DATABASE_URL by default

@Entity(db)
class User { /* ... */ }

await db.create(); // emits DDL for User and any other entity registered against db
```

The class is instantiated **before** any `@Entity` declaration, since each `@Entity(db)` references it. Decorator evaluation populates `db.getMetadata()`; the actual database connection can be opened later via `connect()`.

## Why one class, three jobs?

Splitting connection / metadata / schema across three classes would require passing all three handles around together — every site that uses one tends to use the others. The grouping is anchored by what they share: schema lifecycle reads metadata and writes to the connection; query execution reads metadata and writes to the connection. Keeping them on one object keeps the API surface tight.

## Connections

- [[MetadataStorage]] — the per-`Database` map this class hosts.
- [[Bun]] — the runtime providing the `SQL` driver this class wraps.
- [[schema-create-drop]] — the flow `create()` and `drop()` follow.
- [[Layered Architecture]] — `Database` sits at the edge between the wire (Bun `SQL`) and the rest of the stack.

## Sources

- `.raw/architecture-overview.md` § "Database, Schema, and Lifecycle"
