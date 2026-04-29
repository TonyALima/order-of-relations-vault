# Order of Relations (OOR) — LLM Wiki

Mode: B (GitHub / Repository)
Purpose: Persistent knowledge base mapping the OOR codebase — a TypeScript ORM for PostgreSQL that doubles as a TCC (UNIFEI undergraduate thesis) and a publishable npm package.
Owner: Tony Albert
Created: 2026-04-29

## Where the code lives

This vault documents a real codebase. The code itself is **not** in this vault.

- **Local checkout:** `../order-of-relations` (sibling directory to this vault's repo).
- **GitHub:** https://github.com/TonyALima/order-of-relations

When you need to ground a claim in actual source — verify a method signature, check whether a decorator is implemented, confirm a SQL string — read from `../order-of-relations` (or fetch from GitHub if the local checkout is missing). Treat the code as authoritative and the wiki as interpretation; if they disagree, flag a contradiction rather than silently reconciling.

## Structure

```
vault/
├── .raw/                       # immutable source documents — never edit
├── wiki/
│   ├── index.md                # master catalog of every page
│   ├── log.md                  # append-only operation history (newest on top)
│   ├── hot.md                  # ~500-word recent-context cache
│   ├── overview.md             # executive summary of the wiki
│   ├── getting-started.md      # how to use this vault
│   ├── modules/                # one note per major module / package
│   ├── components/             # reusable building blocks (decorators, classes)
│   ├── decisions/              # ADRs — load-bearing design calls
│   ├── dependencies/           # external packages and runtimes
│   ├── flows/                  # end-to-end behaviors (lifecycle of X)
│   ├── sources/                # one synthesis per .raw/ source
│   ├── entities/               # people, orgs, products, repos, libs
│   ├── concepts/               # ideas, patterns, frameworks
│   ├── domains/                # top-level topic areas
│   ├── comparisons/            # side-by-side analyses
│   ├── questions/              # filed answers to queries
│   └── meta/                   # dashboards, lint reports, conventions
├── _templates/                 # YAML+markdown templates for each note type
├── .obsidian/                  # Obsidian config + CSS snippets
└── CLAUDE.md                   # this file
```

## Conventions

- All notes use YAML frontmatter: `type`, `title`, `created`, `updated`, `tags`, `status` (minimum). See `_templates/` for the canonical shape.
- Wikilinks use `[[Note Name]]` — filenames are unique across the vault, so no paths needed.
- `.raw/` contains source documents and is **never modified**. To correct or extend a source, file a new note in `wiki/` that links back to it.
- `wiki/index.md` is the master catalog: update on every ingest.
- `wiki/log.md` is **append-only**: never edit past entries; new entries go at the **top**.
- `wiki/hot.md` is overwritten end-to-end after every significant operation. Keep under 500 words.
- ADRs live in `wiki/decisions/` with the naming convention `NNNN-short-title.md` (zero-padded sequence).
- Obsidian's `workspace.json` is gitignored — every other `.obsidian/` file is committed.

## Operations

- **Ingest:** drop a source into `.raw/`, say `ingest <filename>`. Claude routes through the `wiki-ingest` skill.
- **Query:** ask any question; Claude reads `hot.md`, then `index.md`, then drills into specific pages.
- **Lint:** say `lint the wiki` to run `wiki-lint` (orphans, dead wikilinks, stale claims).
- **Save:** say `save this` to file the current chat as a structured note.
- **Archive:** move cold sources to `.archive/` (create on demand) to keep `.raw/` lean.

## Getting Started

See [[wiki/getting-started]] inside Obsidian for the day-to-day operating guide.

## What is OOR (one paragraph)

Order of Relations is a small, opinionated ORM that maps decorated TypeScript classes to PostgreSQL tables and exposes a fluent, type-safe query builder over a generic `Repository<T>`. Pillars: ECMAScript Stage-3 decorators (no `reflect-metadata`), parameterized SQL only (no `sql.unsafe`, ever), strict no-`any`, TDD with `bun test`, and Bun as the single toolchain. The full design rationale lives in `.raw/welcome.md` and will be ingested into `wiki/decisions/`, `wiki/concepts/`, and `wiki/flows/` on the first ingest pass.
