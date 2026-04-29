---
type: overview
title: "OOR Wiki — Overview"
created: 2026-04-29
updated: 2026-04-29
tags:
  - meta
  - overview
status: developing
related:
  - "[[index]]"
  - "[[getting-started]]"
sources: []
---

# OOR Wiki — Overview

This vault is the persistent knowledge base for **Order of Relations (OOR)**, a TypeScript ORM for PostgreSQL. OOR is two products at once:

- A **TCC** (undergraduate thesis at UNIFEI) — the academic artifact.
- A **publishable npm package** — the engineering artifact.

The vault is in **GitHub mode**: it maps a code repository as the central object. Modules, components, decisions, dependencies, and flows are first-class citizens.

---

## Where the Code Lives

This vault is *about* a codebase; it does not contain the code itself. See [[order-of-relations]] for the repository entity page.

| Where | Path / URL |
|---|---|
| Local checkout | `../order-of-relations` (sibling directory) |
| GitHub | <https://github.com/TonyALima/order-of-relations> |

The code is the primary source of truth for behavior. The vault is the primary source of truth for *why*. When the two disagree, file a `> [!contradiction]` callout rather than silently reconciling.

---

## What Lives Where

| Folder | Holds | Examples |
|---|---|---|
| `.raw/` | Immutable source documents | Hand-written design notes, README dumps, git log exports, transcripts |
| `wiki/modules/` | Major packages or subsystems | `decorators`, `repository`, `query-builder`, `container`, `migrations` |
| `wiki/components/` | Reusable building blocks | `@Entity`, `@Column`, `Repository<T>`, `QueryBuilder<T>` |
| `wiki/decisions/` | ADRs (Architecture Decision Records) | "Use Stage-3 decorators", "No `reflect-metadata`", "Bun-only toolchain" |
| `wiki/dependencies/` | External packages / runtimes | `bun`, `pg`, `typescript`, ESLint plugin set |
| `wiki/flows/` | Data flows and request paths | Lifecycle of a `findMany`, lifecycle of a `create`, schema-migration apply path |
| `wiki/sources/` | Synthesis pages, one per item in `.raw/` | One page per legacy design note |
| `wiki/entities/` | People, orgs, libs, repos | Bun, PostgreSQL, TypeORM (for comparison), the user as maintainer |
| `wiki/concepts/` | Ideas, patterns, frameworks | "Repository pattern", "Lazy query builder", "Stage-3 decorators" |
| `wiki/domains/` | Top-level topic areas | "ORM design", "TypeScript type system", "PostgreSQL" |
| `wiki/comparisons/` | Side-by-side analyses | OOR vs TypeORM, OOR vs Drizzle, OOR vs Prisma |
| `wiki/questions/` | Filed answers to queries | "Why no `reflect-metadata`?", "When does the query builder execute?" |
| `wiki/meta/` | Dashboards, lint reports, conventions | Lint output, contributor conventions |

---

## Wiki Conventions

- Every page has YAML frontmatter (`type`, `title`, `created`, `updated`, `tags`, `status`).
- Wikilinks use `[[Note Name]]` — filenames are unique across the vault.
- `.raw/` is **never modified**. Treat it as append-only.
- `wiki/index.md` is the master catalog and must be updated on every ingest.
- `wiki/log.md` is **append-only**; new entries go at the top.
- `wiki/hot.md` is overwritten end-to-end after each significant operation.

For day-to-day operation (ingest, query, lint, save), see [[getting-started]].
