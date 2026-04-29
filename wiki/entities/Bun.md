---
type: entity
title: "Bun"
entity_type: tool
role: "Runtime, package manager, test runner, bundler — the single OOR toolchain"
first_mentioned: "[[sources/welcome]]"
created: 2026-04-29
updated: 2026-04-29
tags:
  - entity
  - tool
  - runtime
  - toolchain
status: seed
related:
  - "[[0007-bun-toolchain]]"
  - "[[0006-tdd-rhythm]]"
  - "[[TypeScript]]"
  - "[[PostgreSQL]]"
sources:
  - "../.raw/welcome.md"
---

# Bun

## Overview

Bun is a JavaScript / TypeScript runtime, package manager, test runner, and bundler distributed as a single binary. For OOR it is the **only** toolchain — replacing `node`, `ts-node`, `npm`, `jest`, and `webpack`. Bun's native `SQL` driver is what OOR uses to talk to PostgreSQL.

## Key Facts

- **Native TypeScript.** No `tsconfig` execution flag, no transpile step for running scripts.
- **Native test runner.** `bun test` is what the OOR TDD loop runs against; tests are colocated as `*.test.ts` next to source files.
- **Native package manager.** `bun install` replaces `npm install` / `pnpm install`.
- **Native `.env` loading.** No `dotenv` package needed.
- **Native `SQL` driver.** OOR's `Repository<T>` and `QueryBuilder<T>` ultimately produce parameterized queries that go through Bun's `sql\`...\`` tagged-template API. The `sql.unsafe` escape hatch is **banned** in the OOR codebase — see [[0004-parameterized-sql-only]].
- **Single binary.** `bun --version` is the only runtime version that matters for contributors.

## Connections

- [[0007-bun-toolchain]] — the ADR adopting Bun as the single toolchain.
- [[0006-tdd-rhythm]] — `bun test` is the test runner that makes the TDD loop fast enough to be habitual.
- [[Parameterized SQL]] — Bun's `SQL` driver is the layer at which parameterization happens.
- [[PostgreSQL]] — the database Bun's `SQL` driver targets in OOR.

## Sources

- `.raw/welcome.md`
