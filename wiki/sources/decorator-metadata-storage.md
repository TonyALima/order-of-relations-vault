---
type: source
title: "Decorator Metadata Storage"
source_type: design-note
author: "Tony Albert"
date_published: 2026-04-29
url: ""
confidence: high
key_claims:
  - "MetadataStorage stores exactly three things: entity, column, and relation metadata."
  - "Storage is a single Map<Constructor, EntityMetadata>, intentionally not a WeakMap."
  - "Field decorators write to context.metadata via TWO symbol keys: COLUMNS_KEY and RELATIONS_KEY."
  - "Relations are declared by closure (getTarget) to defer lookup past the temporal dead zone."
  - "@Entity validates at decoration time (must have a primary column) before committing to storage."
  - "Single-table inheritance is resolved lazily on first read; isMetadataResolved is reset on every set()."
  - "Discriminator is wiped when only one class maps to a given table; reappears once a sibling registers."
  - "Multi-level chains (Base ← User ← AdminUser) collapse to the topmost ancestor's table."
  - "Only one explicit error type lives in metadata.errors.ts: RelationTargetNotFoundError."
  - "storage.get(UnregisteredClass) returns undefined; translation to a user error is the repository's job."
created: 2026-04-29
updated: 2026-04-29
tags:
  - source
  - decorators
  - metadata
status: stable
related:
  - "[[sources/welcome]]"
  - "[[sources/architecture-overview]]"
  - "[[MetadataStorage]]"
  - "[[entity-registration]]"
  - "[[Single-Table Inheritance]]"
  - "[[Relation Target Thunk]]"
sources:
  - "../.raw/decorator-metadata-storage.md"
---

# Decorator Metadata Storage

## Summary

The pin-down note on what `MetadataStorage` stores, how decorators write into it, when it gets resolved, and the rationale for the shape. Refines the symbol-key scheme that `[[sources/architecture-overview]]` first sketched, and surfaces a concrete inter-source disagreement on whether nullability lives behind a third symbol key.

## Key Claims

- **Three records, nothing more.** `EntityMetadata`, `ColumnMetadata`, `RelationMetadata`. Schema names, indexes, cascading rules: deliberately absent. Adding later is cheaper than removing.
- **`Map<Constructor, EntityMetadata>`, not a WeakMap.** OOR explicitly wants entity classes alive for the connection's lifetime — the registry *is* the schema.
- **Two symbol keys** appear in this source (`COLUMNS_KEY`, `RELATIONS_KEY`) — but a code spot-check after this ingest revealed **a third (`NULLABLE_KEY`)** that this source elided. See § "Resolved disagreement" below.
- **`@Entity(db, mapTableName?)`** — `mapTableName` defaults to the class name. Validates at decoration time: if no column has `primary: true`, throws `MissingPrimaryColumnError` before touching storage.
- **`context.metadata` is per-class**, not inherited. Stage-3 gives each subclass a fresh bag — by design.
- **Inheritance resolution walks `Object.getPrototypeOf(target.prototype)?.constructor`** up to the topmost decorated ancestor and adopts its table name.
- **Discriminator is dynamic.** Wiped when only one class maps to a given table; reappears once a sibling registers. Multi-level chains collapse to the topmost ancestor's table.
- **`isMetadataResolved` is the laziness flag.** Reset on every `set()`; resolution runs at the next `get()` or iteration. Idempotent across additions (pinned by `metadata.test.ts`'s "re-resolving after a new entity is added does not corrupt discriminator" test).
- **Relations use a `getTarget()` thunk** (closure returning the constructor). Avoids temporal-dead-zone failures in circular entity graphs.
- **Single error in `metadata.errors.ts`:** `RelationTargetNotFoundError`. Orthogonal failures (`MissingPrimaryColumnError` from `@Entity`, `undefined` from `storage.get`) live elsewhere on purpose.

## Entities Mentioned

- [[order-of-relations]] — concrete file references: `metadata.errors.ts`, `metadata.test.ts`.

## Concepts Introduced

- [[Single-Table Inheritance]] — the resolution mechanism with parent-prototype walk, discriminator wiping rule, and multi-level collapse.
- [[Relation Target Thunk]] — the closure pattern that breaks circular-reference temporal-dead-zones.

## Flows Introduced

- [[entity-registration]] — the decorator-evaluation sequence ending in `db.getMetadata().set(ctor, { ... })`.

## Corrections to Existing Pages

- [[ECMAScript Stage-3 Decorators]] — the symbol-key list is **two** (`COLUMNS_KEY`, `RELATIONS_KEY`), not three. The previous "(`COLUMNS_KEY`, `RELATIONS_KEY`, `NULLABLE_KEY`)" framing came from `[[sources/architecture-overview]]` and is contradicted by this source's explicit symbol declarations and `@Entity` body.
- [[MetadataStorage]] — same correction; deepened with the `isMetadataResolved` flag, idempotent re-resolution, WeakMap rationale, the `getTarget` thunk note, and the failure-mode catalog.

## Resolved disagreement

> [!note] Resolved 2026-04-29 — three symbols, not two
> This source said "OOR uses two symbols" and showed an `@Entity` body reading only `COLUMNS_KEY` and `RELATIONS_KEY`. `[[sources/architecture-overview]]` claimed three (including `NULLABLE_KEY`). Owner provided pointers to `src/decorators/nullable/nullable.ts` and `src/decorators/column/column.ts` in [[order-of-relations]]; the code resolves it:
>
> - `nullable.ts` line 1 declares `export const NULLABLE_KEY = Symbol('nullable')`.
> - `@Nullable` / `@NotNullable` write `Map<string, boolean>` (property name → `true` / `false`) under `NULLABLE_KEY`.
> - `column.ts` lines 31–37: `@Column` reads `NULLABLE_KEY`, looks up the property, and throws `MissingNullabilityDecoratorError` (from `nullable.errors.ts`) if missing. `@PrimaryColumn` is exempt — it forces `nullable: false`.
> - **Decorator order matters**: `@Nullable` must be the *inner* decorator (closer to the property) so Stage-3's bottom-up application order runs it before `@Column`.
>
> Why this source elided `NULLABLE_KEY`: its scope was "what flows into `MetadataStorage`." `NULLABLE_KEY` is a **peer-to-peer channel between sibling field decorators**; it's consumed by `@Column` and never reaches storage. From `@Entity`'s viewpoint, two buckets matter. From `context.metadata`'s viewpoint, three exist.
>
> Wiki pages corrected in this resolution: [[ECMAScript Stage-3 Decorators]] (back to three symbols, plus an order-constraint warning), [[MetadataStorage]] (three-key write-path table; `MissingNullabilityDecoratorError` added to failure modes), [[entity-registration]] (third bucket and order-matters note).

## Notes

The source explicitly says "for now" about `RelationType.TO_ONE` — `@ToMany`/`@ManyToMany` are anticipated but not yet present.
