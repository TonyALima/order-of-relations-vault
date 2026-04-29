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

2026-04-29. First ingest pass complete: `.raw/welcome.md` is now fully resolved into the wiki (16 new pages). Four `.raw/` files remain pending — the deeper architecture, decorator-storage, query-builder, and repository sources.

## Key Recent Facts

- OOR is a TypeScript ORM for PostgreSQL, accessed via [[Bun]]'s native `SQL` driver. Bun is the single toolchain.
- The seven foundational ADRs are filed: [[0001-stage-3-decorators]], [[0002-repository-with-lazy-query-builder]], [[0003-singleton-di-container]], [[0004-parameterized-sql-only]], [[0005-no-any-type-driven-api]], [[0006-tdd-rhythm]], [[0007-bun-toolchain]].
- Five core concept pages exist as **seeds**: [[ECMAScript Stage-3 Decorators]], [[Repository Pattern]], [[Lazy Query Builder]], [[Parameterized SQL]], [[Dependency Injection Container]]. Their depth is intentionally limited — the matching `.raw/` files will fill in mechanism details on later ingests.
- Three entity pages exist: [[Bun]], [[PostgreSQL]], [[TypeScript]].
- Hard rules from welcome.md, now codified as ADRs: no `sql.unsafe`, strict `no-explicit-any`, TDD with `bun test`, parameterized queries only.
- Project remains simultaneously a TCC (UNIFEI undergraduate thesis) and a publishable npm package.

## Recent Changes

- **Created (16 pages):** [[sources/welcome]]; ADRs 0001–0007; concept pages for Stage-3 decorators / repository pattern / lazy query builder / parameterized SQL / DI container; entity pages for Bun / PostgreSQL / TypeScript.
- **Updated:** [[index]] (catalog now lists 7 ADRs, 5 concepts, 3 entities, 1 source), [[sources/_index]], [[concepts/_index]], [[entities/_index]], [[decisions/_index]].
- **Manifest seeded:** `.raw/.manifest.json` now tracks the welcome.md ingest with hash + page lists.
- **Flagged:** nothing yet — no contradictions surfaced.

## Active Threads

- **Next ingest candidates** (in `.raw/`): `architecture-overview.md` (likely produces a `flows/` page on the lifecycle of `findMany`, plus deepens `Repository Pattern` + `Lazy Query Builder`), `decorator-metadata-storage.md` (deepens `ECMAScript Stage-3 Decorators` + creates a `components/` page for `MetadataStorage`), `query-builder-design.md` (deepens `Lazy Query Builder` + creates `flows/query-compilation.md`), `repository-contract.md` (creates `components/Repository.md` and a `flows/` page per CRUD method).
- **Open question (still open):** which contents in those four sources are *decisions* worth promoting to additional ADRs versus *flows* versus *concept-deep-dives*. Apply the rule: if a passage describes "we will X because Y," it's an ADR; if it describes "step 1, step 2, step 3," it's a flow; if it explains "X means Y," it's a concept.
- **Watchpoint:** the `0001` ADR claims metadata is stored in a `Map` owned by the library, but the actual layout (per-class? per-property? what keys?) lives in `decorator-metadata-storage.md`. When that ingest happens, cross-check for any contradiction with the [[ECMAScript Stage-3 Decorators]] concept page.
