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

2026-04-29. Vault was scaffolded in GitHub mode for the **Order of Relations (OOR)** project — a TypeScript ORM for PostgreSQL that doubles as a TCC and an npm package. Five hand-written design notes were migrated to `.raw/` as immutable source documents and are awaiting ingestion.

## Key Recent Facts

- OOR is a TypeScript ORM for PostgreSQL, built on Bun's native `SQL` driver.
- It uses ECMAScript Stage-3 decorators (no `reflect-metadata`).
- Architecture: decorators → MetadataStorage → Repository → QueryBuilder → Bun SQL.
- Hard rules: no `sql.unsafe`, no `any`, TDD with `bun test`, parameterized queries only.
- Project is simultaneously an undergraduate thesis (TCC) and a publishable npm package.

## Recent Changes

- Created: [[index]], [[overview]], [[log]], [[hot]], [[getting-started]], all `_index.md` files.
- Migrated to `.raw/`: `welcome.md`, `architecture-overview.md`, `decorator-metadata-storage.md`, `query-builder-design.md`, `repository-contract.md`.
- Flagged: nothing yet — first ingest hasn't happened.

## Active Threads

- Owner is about to run the first ingest pass over the five files in `.raw/` to produce `wiki/sources/` summaries and seed `modules/`, `components/`, `decisions/`, `concepts/`, and `entities/`.
- Open question: which existing notes are *decisions* (should become ADRs) vs *flows* (e.g., the query lifecycle in `architecture-overview.md`) vs *concepts* (e.g., metadata storage).
