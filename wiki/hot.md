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

2026-04-29. Three of five `.raw/` sources ingested. Decorator metadata layer now pinned down: storage shape, write path, lazy resolution, failure modes, single-table inheritance, and the `getTarget()` thunk pattern for circular relations. Two `.raw/` files remain: `query-builder-design.md`, `repository-contract.md`.

## Key Recent Facts

- **MetadataStorage is `Map<Constructor, EntityMetadata>` per `Database`.** Not a `WeakMap` (entities must outlive the connection — the registry is the schema). Iteration is part of the public surface.
- **Three symbol keys** on `context.metadata`: `COLUMNS_KEY` (`ColumnMetadata[]`), `RELATIONS_KEY` (`RelationMetadata[]`), `NULLABLE_KEY` (`Map<string, boolean>`). `@Entity` reads only the first two; `NULLABLE_KEY` is a peer-to-peer channel — `@Nullable`/`@NotNullable` write it, `@Column` reads it and bakes `nullable` into the `ColumnMetadata`.
- **Decorator order matters on columns:** `@Nullable` must be the *inner* decorator (closer to the property) so Stage-3's bottom-up application order runs it before `@Column`. Reversing them throws `MissingNullabilityDecoratorError`. `@PrimaryColumn` is exempt.
- **Lazy resolution flag.** `isMetadataResolved` reset on every `set()`; first subsequent `get()` runs `resolveInheritance` + `resolveRelations`. Idempotent across late additions.
- **Single-table inheritance** ([[Single-Table Inheritance]]): walk `Object.getPrototypeOf` to topmost decorated ancestor, adopt its table name; discriminator = own table name, **wiped** when only one class maps to a table; reappears with siblings.
- **Relations use closures** ([[Relation Target Thunk]]): `@ToOne(() => User)` defers the lookup past the TDZ; `RelationTargetNotFoundError` if the closure returns an unregistered constructor.
- **Failure modes:** `MissingPrimaryColumnError` (at decoration time, from `@Entity`); `RelationTargetNotFoundError` (at first-read resolution, from `metadata.errors.ts`); `storage.get(unregistered)` returns `undefined` deliberately — repository translates.
- **Layering** ([[Layered Architecture]]): five layers, downward dependencies only. Decorators are the only writers of metadata.
- **Bun's `SQL` is the only wire-protocol speaker.** Repository writes build SQL directly; Repository reads delegate to QueryBuilder.
- **DI is not yet implemented.** Direct `new Repository(User, db)` in examples; ADR 0003 stands as decided design.

## Recent Changes

- **Created (4):** [[sources/decorator-metadata-storage]], [[entity-registration]], [[Single-Table Inheritance]], [[Relation Target Thunk]].
- **Refined:** [[ECMAScript Stage-3 Decorators]] (two-symbol correction; second dated note callout), [[MetadataStorage]] (deepened with resolution flag, WeakMap rationale, failure-mode table, contradiction callout).
- **Updated indexes:** [[index]], [[concepts/_index]], [[flows/_index]], [[sources/_index]].
- **Manifest:** `.raw/.manifest.json` now tracks all three ingested sources.

## Recently Resolved

- **`@Nullable` write path** (resolved 2026-04-29): three symbols, not two. `architecture-overview.md` was right; `decorator-metadata-storage.md` had elided `NULLABLE_KEY` because its scope was "what flows into storage" — and `NULLABLE_KEY` doesn't reach storage. Code spot-check against `src/decorators/nullable/nullable.ts` and `src/decorators/column/column.ts` confirmed the third bucket exists, holds a `Map<string, boolean>`, and is enforced by `@Column` via `MissingNullabilityDecoratorError`. Decorator-order constraint surfaced as a new fact: `@Nullable` must be inner. Wiki pages corrected: [[ECMAScript Stage-3 Decorators]], [[MetadataStorage]], [[entity-registration]], [[sources/decorator-metadata-storage]].
- **Test layout** (resolved 2026-04-29): unit tests colocated under `src/`; integration tests under top-level `tests/`. See [[0006-tdd-rhythm]] § Clarification.

## Open Questions

- 🔓 [[decorator-order-independence]] — could `@Column` / `@Nullable` be made order-independent? Filed 2026-04-29 with three sketched approaches (defer-to-`@Entity` is cleanest). No decision yet.

## Active Threads

- **Next ingest candidate:** `.raw/query-builder-design.md` — expected to deepen [[Lazy Query Builder]] and [[Conditions Proxy]], possibly produce `concepts/Type Narrowing.md` and refine [[query-lifecycle]] step 3 / 5 details.
- **Pattern that's now confirmed (3-for-3):** sources disagree with each other or with the manifesto. Read the next two with the assumption that further refinements to existing concept and ADR pages will land.
- **Watchpoint carried forward:** when `.raw/repository-contract.md` is ingested, cross-check the `findById` / `findOne` / `create` per-method contracts against the current [[Repository Pattern]] page; the read-write split may need further refinement.
