---
type: entity
title: "PostgreSQL"
entity_type: database
role: "The only database OOR targets"
first_mentioned: "[[sources/welcome]]"
created: 2026-04-29
updated: 2026-04-29
tags:
  - entity
  - database
status: seed
related:
  - "[[Bun]]"
  - "[[Parameterized SQL]]"
sources:
  - "../.raw/welcome.md"
---

# PostgreSQL

## Overview

PostgreSQL is the sole target database for OOR. The library makes no abstraction over multiple SQL dialects — design choices, type mappings, and the query builder's emitted SQL all assume PostgreSQL semantics. This is a deliberate scope decision: a TCC-scale ORM cannot credibly cover MySQL, SQLite, and Postgres simultaneously.

## Key Facts

- **Sole dialect.** Column type mappings, identifier quoting, and parameterization (`$1`, `$2` placeholders) follow PostgreSQL's wire protocol.
- **Driver.** Accessed via [[Bun]]'s native `SQL` driver, which speaks the PostgreSQL wire protocol directly. No `pg` package on the dependency list.
- **Schema migrations.** OOR's migration model is schema-based and assumes Postgres-style DDL. Detail page pending an ingest of the future `.raw/schema-migrations.md`.
- **Identifier handling.** Dynamic identifiers (table or column names) cannot be parameterized in PostgreSQL; they must come from an allowlist when needed. See [[Parameterized SQL]].

## Connections

- [[Bun]] — provides the `SQL` driver OOR uses to reach PostgreSQL.
- [[Parameterized SQL]] — the safety property at the wire protocol layer.
- [[0004-parameterized-sql-only]] — `sql.unsafe` is banned across the codebase; this matters precisely because PostgreSQL is the one wire we ever touch.

## Sources

- `.raw/welcome.md`
