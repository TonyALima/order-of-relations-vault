---
type: meta
title: "Getting Started"
created: 2026-04-29
updated: 2026-04-29
tags:
  - meta
  - guide
status: developing
related:
  - "[[index]]"
  - "[[overview]]"
sources: []
---

# Getting Started

How to use this vault as a knowledge companion for the **Order of Relations (OOR)** project.

---

## The Three-Layer Mental Model

```
.raw/   ← immutable source documents (never edited)
wiki/   ← LLM-generated knowledge base (synthesis, cross-refs)
CLAUDE.md, _templates/, .obsidian/ ← schema and tooling
```

When in doubt: **add to `.raw/`, synthesize into `wiki/`**.

---

## Day-to-Day Operations

### Ingest a source

1. Drop a file into `.raw/` (or paste a URL into chat).
2. Tell Claude: `ingest <filename>` or `process this`.
3. Claude reads the source, extracts entities and concepts, files them into the right `wiki/` folders, cross-references them, and updates [[index]], [[log]], and [[hot]].

### Ask a question

- `what do you know about <topic>` or `query: <question>`
- Claude reads [[hot]] first, then [[index]], then drills into specific pages. Good answers get filed under `wiki/questions/`.

### Lint the wiki

- `lint the wiki` or `health check`
- Claude scans for orphan pages, dead wikilinks, stale claims, missing cross-refs.

### Save a conversation

- `save this` or `/save`
- Files the current chat into the right `wiki/` folder as a structured note.

---

## What's Already Here (as of 2026-04-29)

- Five hand-written design notes are sitting in `.raw/`, untouched. They are the seed corpus:
  - `welcome.md` — vault charter, high-level decisions
  - `architecture-overview.md` — layered view, query lifecycle
  - `decorator-metadata-storage.md` — how decorators write metadata
  - `query-builder-design.md` — clause accumulation, lazy execution
  - `repository-contract.md` — what `create`/`findOne`/`find` guarantee
- Nothing in `wiki/` is real synthesis yet — only the scaffold.

**Next step:** ingest those five files. Each one will produce a `wiki/sources/` page plus seeded entries in `decisions/`, `concepts/`, `flows/`, and `components/`.

---

## When Things Get Ambiguous

- A note that explains *why we picked X over Y* belongs in `decisions/`.
- A note that walks through *what happens when foo runs* belongs in `flows/`.
- A note that defines *what foo is and how it works* belongs in `concepts/`.
- A note about *a specific subsystem of the code* belongs in `modules/`.
- A note about *a reusable building block* (a decorator, a class, a function) belongs in `components/`.

When a single existing note touches several of these, *split* it on ingest rather than forcing it into one folder.
