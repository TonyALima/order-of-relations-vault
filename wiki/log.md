---
type: meta
title: "Operation Log"
created: 2026-04-29
updated: 2026-04-29
tags:
  - meta
  - log
status: evergreen
related: []
sources: []
---

# Operation Log

Append-only record of every wiki operation. Newest entries on top. Never edit past entries — file new corrections instead.

---

## 2026-04-29 — ingest | Welcome to the OOR Vault

- Source: `.raw/welcome.md` (md5 `6c43e80854db106e0c8ac31648c25c77`)
- Summary: [[sources/welcome|Welcome to the OOR Vault]]
- Pages created (16):
  - Source: [[sources/welcome]]
  - ADRs: [[0001-stage-3-decorators]], [[0002-repository-with-lazy-query-builder]], [[0003-singleton-di-container]], [[0004-parameterized-sql-only]], [[0005-no-any-type-driven-api]], [[0006-tdd-rhythm]], [[0007-bun-toolchain]]
  - Concepts: [[ECMAScript Stage-3 Decorators]], [[Repository Pattern]], [[Lazy Query Builder]], [[Parameterized SQL]], [[Dependency Injection Container]]
  - Entities: [[Bun]], [[PostgreSQL]], [[TypeScript]]
- Pages updated: [[index]], [[hot]], [[sources/_index]], [[concepts/_index]], [[entities/_index]], [[decisions/_index]]
- Manifest: created `.raw/.manifest.json` with delta entry for `welcome.md`.
- Key insight: welcome.md was authored *as* a decision register draft — the seven "High-Level Decisions" map cleanly onto seven ADRs. Deeper concept pages (metadata storage, query-builder mechanics, repository contract) intentionally left as seeds; they will be filled by the four sibling `.raw/` ingests.

---

## 2026-04-29 — Vault scaffolded (GitHub mode)

- Migrated five pre-existing notes (`Welcome.md`, `Architecture Overview.md`, `Decorator Metadata Storage.md`, `Query Builder Design.md`, `Repository Contract.md`) to `.raw/` as kebab-case source documents. They are immutable from this point on.
- Created full wiki scaffold: `index.md`, `log.md`, `hot.md`, `overview.md`, `getting-started.md`.
- Created subfolders for GitHub mode: `modules/`, `components/`, `decisions/`, `dependencies/`, `flows/`.
- Created standard subfolders: `sources/`, `entities/`, `concepts/`, `domains/`, `comparisons/`, `questions/`, `meta/`.
- Each subfolder seeded with `_index.md`.
- Added `_templates/` (source, entity, concept, comparison, question).
- Applied vault visual customization at `.obsidian/snippets/vault-colors.css`.
- Updated `.gitignore` to cover the wiki conventions.
- Created vault `CLAUDE.md` declaring the wiki schema and conventions.

**Next:** ingest the four remaining `.raw/` files (`architecture-overview.md`, `decorator-metadata-storage.md`, `query-builder-design.md`, `repository-contract.md`) to populate `flows/`, `components/`, and the deeper concept pages currently seeded as stubs.
