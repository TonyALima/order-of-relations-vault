---
type: entity
title: "order-of-relations"
entity_type: repository
role: "The OOR codebase — TypeScript ORM library this vault documents"
first_mentioned: "[[overview]]"
created: 2026-04-29
updated: 2026-04-29
tags:
  - entity
  - repository
  - codebase
status: stable
related:
  - "[[overview]]"
  - "[[Bun]]"
  - "[[PostgreSQL]]"
  - "[[TypeScript]]"
sources: []
---

# order-of-relations

## Overview

`order-of-relations` is the source repository this vault documents. It hosts the TypeScript ORM library — the **engineering artifact** of the OOR project, paired with the TCC text as the **academic artifact**. The vault and the codebase are siblings: neither contains the other.

## Locations

- **Local checkout:** `../order-of-relations` — sibling directory to the vault repo. Use this when you need to read actual source, run `bun test`, or verify a current claim against the implementation.
- **GitHub:** <https://github.com/TonyALima/order-of-relations> — canonical remote. Reach for it when the local checkout is unavailable, or when citing a specific commit / line.
- **npm:** *not yet published.* The TCC angle gates the public release.

## Key Facts

- **Owner:** Tony Albert ([github.com/TonyALima](https://github.com/TonyALima)).
- **Stack:** [[TypeScript]] source, [[Bun]] toolchain (runtime + test runner + bundler + package manager), [[PostgreSQL]] as the only target database.
- **License & version:** see the repo `README.md` and `package.json` directly. Not duplicated here so it cannot drift.
- **Test layout:** colocated `*.test.ts` next to source files; runner is `bun test`. See [[0006-tdd-rhythm]].
- **Hard rules embedded in the codebase:**
  - No `sql.unsafe` anywhere — see [[0004-parameterized-sql-only]].
  - No `any` — `@typescript-eslint/no-explicit-any` strict — see [[0005-no-any-type-driven-api]].
  - Stage-3 decorators only — see [[0001-stage-3-decorators]].

## Vault ↔ Code Boundary

| Authoritative for | Lives in |
|---|---|
| What the code *does* (current behavior) | `../order-of-relations` |
| Why the code is shaped this way | This vault (`wiki/decisions/`, `wiki/concepts/`, `wiki/flows/`) |
| Source documents and prior design notes | `.raw/` |

When a wiki page and the code disagree, the code wins on behavior — but the disagreement gets logged as a `> [!contradiction]` rather than silently overwritten. The wiki is allowed to lag, but it is not allowed to lie.

## Connections

- [[overview]] — vault overview, where this repo is also linked.
- [[Bun]] / [[PostgreSQL]] / [[TypeScript]] — the runtime, database, and language the codebase commits to.
- All seven ADRs ([[0001-stage-3-decorators]] through [[0007-bun-toolchain]]) — the load-bearing decisions encoded in the code's structure.

## Sources

- `CLAUDE.md` § "Where the code lives"
