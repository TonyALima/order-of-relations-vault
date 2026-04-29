---
type: meta
title: "Hot Cache"
updated: 2026-04-29T00:00:00
created: 2026-04-29
tags:
  - meta
  - cache
status: evergreen
related: []
sources: []
---

# Recent Context

## Last Updated

2026-04-29. Two of five `.raw/` sources ingested. Architecture is now mapped end to end (decorators → metadata → repository → query-builder → Bun SQL); the **read path** has a step-by-step flow page; the **schema lifecycle** has its own flow. Three `.raw/` files remain: `decorator-metadata-storage.md`, `query-builder-design.md`, `repository-contract.md`.

## Key Recent Facts

- **Layering** ([[Layered Architecture]]): five layers, dependencies always point downward. Decorators are the only writers of metadata; nothing under `core/metadata/` imports from above.
- **MetadataStorage is per-`Database`**, not library-global. `@Entity(db)` registers entities into that database's storage. Two `Database` instances = two metadata maps. (Corrects the welcome-doc framing.)
- **Repository reads delegate to QueryBuilder.** `findOne` / `findMany` / `findById` all build a `QueryBuilder<T>` internally and call its terminal methods. Only writes (`create`/`update`/`delete`) build SQL directly. (Corrects the welcome-doc framing.)
- **`where` is a callback** receiving a typed [[Conditions Proxy]] (one `FieldConditionBuilder` per column). Not a plain object. Returns `Condition[]`. The query builder validates the array (rejects `undefined` entries) before composing SQL.
- **Bun's `SQL` is the only wire-protocol speaker.** Nothing in OOR talks PostgreSQL directly.
- **Schema lifecycle:** `Database.create()` is two passes (tables, then `ALTER TABLE ADD FOREIGN KEY`); `Database.drop()` topologically reverses the relation graph so FK targets drop after referrers.
- **DI is not yet implemented.** No `Container`, `@Service`, `@Inject`, or `@InjectRepository` in `src/` as of this ingest. Examples use direct `new Repository(User, db)`. ADR 0003 stands as the decided design; only implementation is pending.

## Recent Changes

- **Created (7):** [[sources/architecture-overview]], [[query-lifecycle]], [[schema-create-drop]], [[MetadataStorage]], [[Database]], [[Layered Architecture]], [[Conditions Proxy]].
- **Refined:** [[ECMAScript Stage-3 Decorators]] (per-`Database` storage), [[Repository Pattern]] (read delegation), [[Lazy Query Builder]] (callback-style `where`), [[0003-singleton-di-container]] (implementation-status warning).
- **Updated indexes:** [[index]], [[concepts/_index]], [[components/_index]], [[flows/_index]], [[modules/_index]] (now hosts the full source-tree catalog with layer assignments).
- **Manifest:** `.raw/.manifest.json` now tracks both ingested sources.

## Recently Resolved

- **Test layout** (resolved 2026-04-29): unit tests colocated under `src/`; integration tests under top-level `tests/`. See [[0006-tdd-rhythm]] § Clarification (2026-04-29).

## Active Threads

- **Next ingest candidate:** `.raw/decorator-metadata-storage.md` — expected to deepen [[ECMAScript Stage-3 Decorators]] and [[MetadataStorage]] (the symbol-key layout, the resolution passes' implementation, the `@Nullable` / `@NotNullable` enforcement). May produce a `concepts/Class-Table Inheritance.md` page.
- **Pattern to watch in remaining ingests:** the welcome-doc has now been wrong twice (test layout, DI presence). Read the next three sources with the assumption that they may further refine `[[Repository Pattern]]`, `[[Lazy Query Builder]]`, and any concept page.
- **Watchpoints carried forward:** when `.raw/decorator-metadata-storage.md` is ingested, cross-check the `COLUMNS_KEY` / `RELATIONS_KEY` / `NULLABLE_KEY` symbol scheme against [[ECMAScript Stage-3 Decorators]] and [[MetadataStorage]]; refine if it disagrees.
