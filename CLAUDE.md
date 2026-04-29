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
├── .inbox/                     # staging for unshaped notes from other contexts; triage moves items into wiki/
│   └── .processed/             # archive of inbox items already filed
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
- Wikilinks use `[[Note Name]]` — filenames are unique across the vault, so the bare basename usually works. For sources and other path-prefixed targets, the path-form `[[sources/welcome]]` (with optional alias `[[sources/welcome|Welcome]]`) is also valid and renders cleaner.
- `.raw/` contains source documents and is **never modified**. To correct or extend a source, file a new note in `wiki/` that links back to it.
- `.inbox/` is the staging area for **unshaped notes from other contexts** (code-agent sessions, vault-unaware chats, quick captures). No schema required; one file per idea. Triage in a vault-aware session moves each item into the right wiki location and archives the original to `.inbox/.processed/`. Never use `.inbox/` for shaped sources — those go in `.raw/`. See `.inbox/README.md`.
- `wiki/index.md` is the master catalog: update on every ingest.
- `wiki/log.md` is **append-only**: never edit past entries; new entries go at the **top**. *Dead wikilinks inside past log entries are preserved as audit trail and are not lint issues.*
- `wiki/hot.md` is overwritten end-to-end after every significant operation. Keep under 500 words.
- Obsidian's `workspace.json` is gitignored — every other `.obsidian/` file is committed.

### Filename naming (per category)

The vault uses **category-specific** filename conventions, consistent within each folder. Wikilinks must match filenames exactly, so the convention you pick for a new file determines how it gets linked.

| Folder | Convention | Examples | Why |
|---|---|---|---|
| `wiki/concepts/` | **Title Case with spaces** | `Lazy Query Builder.md`, `Repository Pattern.md` | Concepts read as natural-language ideas; the page title and the wikilink form match. Exception: code identifiers keep their literal form (`sqlJoin.md`). |
| `wiki/components/` | **PascalCase**, matching the exported class | `Repository.md`, `QueryBuilder.md`, `MetadataStorage.md`, `Database.md` | Component pages document a concrete TypeScript class; the filename matches the symbol name to make grep/lookup symmetric. |
| `wiki/decisions/` | `NNNN-short-title.md` (zero-padded sequence) | `0001-stage-3-decorators.md`, `0007-bun-toolchain.md` | Standard ADR convention. Sortable in any file listing; references like "ADR 0003" map directly to the filename. |
| `wiki/sources/` | **lowercase-kebab**, mirroring the `.raw/` filename | `welcome.md`, `architecture-overview.md` | The `.raw/.manifest.json` keys path-to-path; symmetric naming makes the manifest readable and the synthesis-to-source mapping obvious. |
| `wiki/flows/` | **lowercase-kebab** | `query-lifecycle.md`, `entity-registration.md`, `lifecycle-of-a-create.md` | Flows are named after the behavior they describe ("the lifecycle of X"); kebab is conventional for action-shaped slugs. |
| `wiki/questions/` | **lowercase-kebab** | `decorator-order-independence.md`, `get-one-limit-1.md` | Questions are slugs of the question itself; kebab keeps URLs / cross-tool references portable. |
| `wiki/entities/` | **Proper-noun shape** for orgs/products (`Bun.md`, `PostgreSQL.md`); **kebab** for code-shaped entities (`order-of-relations.md`) | — | Match how the entity is referred to in the world. Repos use their actual GitHub slug; products use their canonical capitalization. |
| `_index.md` (any folder) | Literal `_index.md` | — | Convention for folder hub pages; sorts to the top of directory listings. |
| `wiki/index.md`, `log.md`, `hot.md`, `overview.md`, `getting-started.md` | Literal lowercase | — | Top-level meta hubs. |

When in doubt: look at neighboring files in the same folder and match the local convention. Mixing within a folder is the actual style violation; using a different convention across folders is intentional.

## Operations

- **Ingest:** drop a *shaped source* into `.raw/`, say `ingest <filename>`. Claude routes through the `wiki-ingest` skill. Use this for design memos, architecture docs, drift audits — anything authoritative.
- **Triage inbox:** drop *unshaped notes* into `.inbox/`, say `triage my inbox` (or `process .inbox/<file>`). Vault-aware Claude reads each note, proposes a destination (open question, drift correction, ADR seed, page edit, or discard), files it on confirmation, and moves the original to `.inbox/.processed/`. Use this when an idea surfaces in a context that doesn't know vault conventions.
- **Query:** ask any question; Claude reads `hot.md`, then `index.md`, then drills into specific pages.
- **Lint:** say `lint the wiki` to run `wiki-lint` (orphans, dead wikilinks, stale claims).
- **Save:** say `save this` to file the current chat as a structured note.
- **Archive:** move cold sources to `.archive/` (create on demand) to keep `.raw/` lean.

### Ingest vs triage — which one?

| Source | Folder | Operation | Why |
|---|---|---|---|
| Shaped, authoritative document (design memo, architecture doc, drift audit) | `.raw/` | `ingest` | Ingest extracts entities/concepts/flows and produces a `wiki/sources/X.md` synthesis. Treats input as authoritative. |
| Unshaped scratch note (code-agent jotting, half-formed thought, idea from another repo) | `.inbox/` | `triage my inbox` | Triage decides per-note whether it becomes a question, drift correction, ADR, or page edit. Treats input as provisional. |
| Idea you can describe in chat right now | — | just say it | Vault-aware Claude writes the page directly. No file needed. |

## Getting Started

See [[wiki/getting-started]] inside Obsidian for the day-to-day operating guide.

## What is OOR (one paragraph)

Order of Relations is a small, opinionated ORM that maps decorated TypeScript classes to PostgreSQL tables and exposes a fluent, type-safe query builder over a generic `Repository<T>`. Pillars: ECMAScript Stage-3 decorators (no `reflect-metadata`), parameterized SQL only (no `sql.unsafe`, ever), strict no-`any`, TDD with `bun test`, and Bun as the single toolchain. The full design rationale lives in `.raw/welcome.md` and will be ingested into `wiki/decisions/`, `wiki/concepts/`, and `wiki/flows/` on the first ingest pass.
