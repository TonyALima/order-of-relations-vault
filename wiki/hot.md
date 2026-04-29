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

2026-04-29. **Drift correction batch ingested (D1, D3, D4, D5, M3, M5, M6).** Seven `.raw/drift-*` notes audit the wiki against the live `[[order-of-relations]]` codebase and call out specific gaps. All seven now have synthesis pages and the corrections they call for have been applied. Page count: 52.

## Drift outcomes

- **D1 — `Repository.find()` does not exist.** Removed `find()` from [[brief]], [[Repository]], [[Repository Pattern]], [[Lazy Query Builder]]. New framing: `findOne` / `findMany` *are* the composition entry points (no more "sugar over `find()`").
- **D3 — `FindOptions.inheritance` is real.** Closed the largest documentation gap. `InheritanceSearchType` (`ALL` / `ONLY` / `SUBCLASSES`) is documented in [[QueryBuilder]], [[Single-Table Inheritance]], [[query-lifecycle]], [[brief]]. The discriminator-only-when-needed and double-applyOptions footguns are pinned.
- **D4 — `FieldConditionBuilder` operator inventory.** Fixed [[Conditions Proxy]]: nine methods (`eq`, `ne`, `gt`, `gte`, `lt`, `lte`, `isNull`, `isNotNull`, `in`), uniform across types. Removed non-existent `before` / `after` / `matches` and the per-type narrowing claim.
- **D5 — implicit `idx_discriminator` index.** STI roots emit `ALTER ADD COLUMN discriminator` + `CREATE INDEX idx_discriminator`. Documented in [[MetadataStorage]], [[schema-create-drop]], [[Single-Table Inheritance]], [[Database]]. Latent name-collision risk flagged.
- **M3 — module pages backlog.** Filed [[modules/sql-types]] (highest leverage); deferred the rest with a Deferred table in [[modules/_index]]. Fixed the 8 → 11 leaf-directory miscount.
- **M5 — `src/core/sql-types/` undocumented.** [[modules/sql-types]] now covers `COLUMN_TYPE` (closed enum), `getColumnTypeDefinition`, and the SERIAL→INTEGER `toForeignKeyType` demotion. Cross-linked from [[Autogeneration]], [[Relation Target Thunk]], [[MetadataStorage]], [[brief]].
- **M6 — `examples/` invisible.** Filed [[examples/_index]] + per-scenario front-doors ([[examples/basic-crud]], [[examples/inheritance]], [[examples/relations]] (stub)). [[examples/inheritance]] is load-bearing — it's the canonical use site for `InheritanceSearchType`.

## Key facts (current state)

- **Repository surface = exactly 6 methods.** `findMany`, `findOne`, `findById`, `create`, `delete`, `update`. There is no `find()`. Reads compose through `findOne` / `findMany`'s internal one-shot `QueryBuilder`.
- **`FindOptions<T>` carries TWO fields**: `where` (replaces conditions) and `inheritance` (pushes a discriminator predicate). `applyOptions` runs `where` first, then inheritance — so a single call cleanly ANDs them; a second call's `where` clobbers the discriminator.
- **`FieldConditionBuilder<V>` exposes 9 methods, uniform across `V`**. Per-type operator narrowing is "future ergonomics," not shipped.
- **STI schema-create emits THREE statements per root**: `CREATE TABLE`, `ALTER ADD COLUMN discriminator`, `CREATE INDEX idx_discriminator`. Subclass entities are skipped (their `tableName` matches the root).
- **`COLUMN_TYPE` is a closed enum (~50 PG types)**; `toForeignKeyType` demotes SERIAL/SMALLSERIAL/BIGSERIAL → INTEGER/SMALLINT/BIGINT for FK columns referencing those PKs.
- **`examples/inheritance/services/UserHierarchyService.ts`** is the only place `InheritanceSearchType` is shown in use — link to it whenever the option comes up.

## Open Questions (still deferred)

- 🔓 [[decorator-order-independence]] — could `@Column` / `@Nullable` work regardless of decoration order?
- 🔓 [[get-one-limit-1]] — `getOne()` slicing vs `LIMIT 1`?
- 🔓 [[apply-options-accumulation]] — `applyOptions()` replace vs accumulate?
- 🔓 (potential, from D5) `idx_discriminator` collision — namespace per-table (`idx_<root>_discriminator`)?

## Active Threads

- **Drift backlog still has uncovered ground:** `src/core/orm-error/`, `src/decorators/nullable/`, `src/decorators/relation/`. Drift-M3's "deferred" list captures them; file when leverage warrants.
- **`examples/relations/` page is a stub** — files weren't opened in the drift pass; expand once read.
- **Suggested next action:** `lint the wiki` — page counts may now drift again after this large batch (12 new pages, 13 updated). A lint pass surfaces orphans, dead wikilinks, frontmatter gaps. Drift-D5 pointed at one optional new question page ([[idx-discriminator-collision]]) that has not been filed.
