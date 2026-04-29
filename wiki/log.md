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

## 2026-04-29 — resolved | Test-layout contradiction

- Owner confirmed: unit tests are colocated as `*.test.ts` next to source under `src/`; integration tests live under the top-level `tests/` directory. Both run under `bun test`.
- Pages updated:
  - [[0006-tdd-rhythm]] — Decision section rewritten to split unit vs. integration; new `## Clarification (2026-04-29)` block added; Positive consequences and Alternatives Considered expanded to reflect the dual layout. Original text replaced (this is the wiki side, not the immutable source).
  - [[sources/welcome]] — `> [!contradiction]` callout downgraded to a `> [!note]` Resolved note pointing at the clarified ADR. The `.raw/welcome.md` file itself is untouched (immutable per vault policy).
  - [[hot]] — "Open Contradictions" section retired; replaced with a "Recently Resolved" note.
- Resolution shape used: **clarification, not supersession.** The ADR's load-bearing claim (TDD rhythm + `bun test`) was correct; only the location sub-claim was under-specified.

---

## 2026-04-29 — recorded | Codebase location

- Owner provided the location of the source repo this vault documents: local at `../order-of-relations`, remote at <https://github.com/TonyALima/order-of-relations>.
- Pages created: [[order-of-relations]] (entity page).
- Pages updated: [[CLAUDE]] (added "Where the code lives" section), [[overview]] (added repo-location table), [[entities/_index]], [[index]].
- Verified the local checkout exists. Discovered a contradiction with [[0006-tdd-rhythm]] / [[sources/welcome]]: the manifesto says tests are colocated rather than in a separate `tests/` tree, but the repo has both — 11 colocated `*.test.ts` files plus 3 in `tests/`. Filed `> [!contradiction]` callouts on both pages. Resolution pending owner.

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
