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

**Next:** ingest the five files in `.raw/` to produce real `wiki/sources/` pages, then populate `modules/`, `components/`, `decisions/`, `concepts/`, and `entities/` from those.
