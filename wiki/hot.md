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

2026-04-29. **All five `.raw/` sources ingested.** OOR's architecture is now fully mapped end-to-end: decorators → metadata → repository → query-builder → Bun SQL. 38 wiki pages across sources, components, concepts, flows, decisions, entities, and questions. Zero open contradictions; three open questions deliberately deferred.

## Key Recent Facts

- **Layering** ([[Layered Architecture]]): five layers, downward dependencies only. Decorators are the only writers of metadata.
- **Three symbol keys** on `context.metadata`: `COLUMNS_KEY`, `RELATIONS_KEY`, `NULLABLE_KEY`. Decorator order matters on columns: `@Nullable` must be the *inner* decorator. `@PrimaryColumn` is exempt.
- **MetadataStorage is per-`Database`**, not library-global. `Map<Constructor, EntityMetadata>`, lazy-resolved, idempotent on late additions.
- **Repository is narrow** ([[Repository]] / [[Repository Pattern]]): split is **by-key (single-row)** vs. **composed**. `create`/`findById`/`update`/`delete` funnel through one `requirePrimaryKey` gate; `find()` returns a `QueryBuilder`; `findOne`/`findMany` are sugar.
- **`create(entity: T)`** — signature is `T`, not `Partial<T>`. The decorators shape `T` so "required on the type means required at `create()`." Returns `Partial<T>` with **only PK fields** (not a hydrated entity).
- **Autogeneration is explicit-only** ([[Autogeneration]], commit `3aa354b`). Two strategies (`clientSide` / `dbSide`). Caller value always wins (commit `cfe4cc5`).
- **Single repository error:** `IncompletePrimaryKeyError`. Driver errors propagate unwrapped.
- **QueryBuilder is mutable, single-owner.** State is one field: `conditions: Condition[] = []`. `applyOptions()` replaces (not accumulates). Two terminal methods: `getMany()`, `getOne()` (slices client-side, no `LIMIT 1` yet).
- **`where` callback signature:** `(conditions: Conditions<T>) => (Condition | undefined)[]`. `Conditions<T>` is a `Partial` mapped type; runtime catches missing-column entries with `UndefinedWhereConditionError` carrying the offending **index**.
- **SQL composition mechanics:** `opFragments` static map keyed by closed enum; `sql(c.columnName)` for identifiers; values bind as parameters; empty-array `IN ()` is valid match-nothing.
- **`sqlJoin` is the only sanctioned fragment joiner.** Hand-rolled `reduce` is rejected as a footgun.
- **No `qb.raw()` escape hatch.** Reaffirms [[0004-parameterized-sql-only]].
- **DI is not yet implemented.** Direct `new Repository(User, db)` in examples; ADR 0003 stands as decided design.
- **Test layout** (resolved): unit tests colocated under `src/`; integration tests under top-level `tests/`.

## Recent Changes

- **Final ingest: Repository Contract.** Created [[sources/repository-contract]], [[Repository]] (component), [[Autogeneration]] (concept), [[lifecycle-of-a-create]] (flow). Refined [[Repository Pattern]] (third refinement pass: by-key vs composed split, `create(entity: T)` signature, type-shapes-the-contract subsection).
- **Indexes finalized:** all sub-`_index.md` files now reflect their stable state. [[sources/_index]] marks all five sources ✅. [[index]] page count: 38.
- **Manifest complete:** `.raw/.manifest.json` tracks all five sources with hashes, created pages, and updated pages.

## Recently Resolved

- **`@Nullable` write path** (resolved 2026-04-29): three symbols, not two; code spot-check confirmed.
- **Test layout** (resolved 2026-04-29): unit colocated, integration in `tests/`.

## Open Questions (deliberately deferred design choices)

- 🔓 [[decorator-order-independence]] — could `@Column` / `@Nullable` work regardless of decoration order?
- 🔓 [[get-one-limit-1]] — `getOne()` slicing vs `LIMIT 1`?
- 🔓 [[apply-options-accumulation]] — `applyOptions()` replace vs accumulate?

## Active Threads

- **All `.raw/` sources are ingested.** No further single-source ingests pending. Future inputs will likely be:
  - **Code spot-checks** (like the `@Nullable` resolution pass) when wiki claims need verification against the live `[[order-of-relations]]` repo.
  - **Future `.raw/` files** the owner adds — schema migrations, comparison notes, TCC chapters, performance studies.
  - **Open-question resolutions** — when one of the three open questions is decided, the resolution flow is "fill the Answer section, flip status to `answered`, optionally file a new ADR."
- **Two throughlines for any future API addition:**
  - QueryBuilder: *"keep the builder small, keep the SQL parameterized, keep the types honest."*
  - Repository: *"these are not gaps; they are the boundary. Anything composed, reactive, or stateful goes elsewhere."*
- **Suggested next action:** `lint the wiki` — run `wiki-lint` to surface orphans, dead wikilinks, frontmatter gaps, and stale claims now that the bulk ingest is complete. Some `_index.md` page-counts may be slightly off; a lint pass will catch them.
