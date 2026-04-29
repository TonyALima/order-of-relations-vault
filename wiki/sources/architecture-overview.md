---
type: source
title: "Architecture Overview"
source_type: design-note
author: "Tony Albert"
date_published: 2026-04-29
url: ""
confidence: high
key_claims:
  - "OOR has five layers; dependencies always point downward."
  - "MetadataStorage is owned by a Database instance, not by the library globally."
  - "Decorators are the only writers of metadata; metadata freezes after class evaluation."
  - "Repository.findOne / findMany / findById all delegate to QueryBuilder; only writes (create/update/delete) build SQL directly."
  - "The where callback receives a typed conditions proxy (one FieldConditionBuilder per column), not a plain object."
  - "Bun's SQL is the only thing that touches the wire."
  - "DI (Container, @Service, @Inject, @InjectRepository) is not yet implemented in src/; current wiring is direct construction."
  - "Database.create() emits CREATE TABLE in two passes (tables, then ALTER TABLE ADD FOREIGN KEY); Database.drop() topologically reverses the relation graph."
created: 2026-04-29
updated: 2026-04-29
tags:
  - source
  - architecture
status: stable
related:
  - "[[sources/welcome]]"
  - "[[Layered Architecture]]"
  - "[[query-lifecycle]]"
sources:
  - "../.raw/architecture-overview.md"
---

# Architecture Overview

## Summary

The conceptual map of OOR. Defines the five-layer stack (decorators → MetadataStorage → Repository → QueryBuilder → Bun SQL), the **downward-only** dependency rule, and the actual lifecycle of a `findMany` call. Refines and *corrects* several claims from `.raw/welcome.md`: notably that `MetadataStorage` is per-`Database` rather than a library global, that repository reads delegate to the query builder, and that DI has not yet shipped in `src/`.

## Key Claims

- **Five layers, downward dependencies only.** Decorators write metadata; nothing under `core/metadata/` imports from above. See [[Layered Architecture]].
- **`MetadataStorage` is owned by a `Database` instance.** `@Entity(db)` takes the database as an argument. Two databases in the same process means two metadata maps. See [[MetadataStorage]].
- **Decorators are the only writers of metadata.** Nothing outside `src/decorators/` calls `MetadataStorage.set`. After class evaluation, metadata is frozen for the process lifetime.
- **Repository never builds read SQL on its own.** `findOne`, `findMany`, `findById` all construct a `QueryBuilder<T>` and delegate. Writes (`create`, `update`, `delete`) build SQL directly because they don't compose. See [[Repository Pattern]] (refined).
- **`where` is a callback receiving a typed proxy.** `where: (u) => [u.email!.eq('a@b.com')]`. The proxy has one `FieldConditionBuilder` per column. See [[Conditions Proxy]].
- **Validation step:** the query builder rejects `undefined` entries in the returned condition array (`UndefinedWhereConditionError`) — common bug from `u.foo?.eq(...)` against a non-existent column.
- **Bun's `SQL` is the only thing that touches the wire.** Nothing in OOR speaks the PostgreSQL protocol directly.
- **`Database` does three jobs:** connection (wraps Bun `SQL`, falls back to `DATABASE_URL`), metadata host, schema lifecycle (`create()` / `drop()`). See [[Database]].
- **Schema `create()` is two passes:** base tables first, then `ALTER TABLE … ADD FOREIGN KEY` for relations. `drop()` walks the relation graph in topological reverse so FK targets are dropped after referrers. See [[schema-create-drop]].
- **DI is not yet implemented.** No `Container`, `@Service`, `@Inject`, or `@InjectRepository` in `src/` as of this source's date. Current `examples/` use direct construction: `new Repository(User, db)`.
- **Decorator ordering matters.** Field decorators run before class decorators. By the time `@Entity(db)` fires, `COLUMNS_KEY` / `RELATIONS_KEY` arrays in `context.metadata` are populated. `@Nullable` / `@NotNullable` populate a per-property nullability map; `@Column` and `@ToOne` refuse to register a property whose nullability isn't declared.

## Entities Mentioned

- [[order-of-relations]] — the codebase. Source-tree map at the bottom of this source covers `src/decorators/`, `src/core/metadata/`, `src/core/database/`, `src/core/repository/`, `src/core/sql-types/`, `src/core/utils/`, `src/query-builder/`, `src/errors.ts`.
- [[Bun]] — its `SQL` is the only wire-protocol speaker.
- [[PostgreSQL]] — the wire target.

## Concepts Introduced

- [[Layered Architecture]] — five layers, downward dependencies only.
- [[Conditions Proxy]] — the typed proxy of `FieldConditionBuilder`s passed to the `where` callback.

## Components Introduced

- [[MetadataStorage]] — `Map<Constructor, EntityMetadata>`, owned by a `Database`.
- [[Database]] — connection + metadata host + schema lifecycle.

## Flows Introduced

- [[query-lifecycle]] — six-step walkthrough of `userRepo.findMany({ where: ... })`.
- [[schema-create-drop]] — two-pass `CREATE TABLE`, topologically reversed `DROP`.

## Corrections to Existing Pages

This source refined three existing pages:

- [[ECMAScript Stage-3 Decorators]] — the `Map` is **per-`Database`**, not a single library-owned global. `context.metadata` (Stage-3's per-class bag) is the *buffer*; `@Entity(db)` flushes from the buffer into `db.getMetadata()`.
- [[Repository Pattern]] — `findOne` / `findMany` / `findById` delegate to a `QueryBuilder`. Only writes (`create`, `update`, `delete`) build SQL directly.
- [[Lazy Query Builder]] — `where` is a **callback** receiving a typed proxy, not a plain object.
- [[0003-singleton-di-container]] — flagged as **not yet implemented**.

## Notes

The "Source Tree, by Concern" section at the bottom of the source is the basis for the seeded module catalog at [[modules/_index]].
