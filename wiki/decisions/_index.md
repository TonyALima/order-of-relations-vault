---
type: domain
title: "Decisions (ADRs)"
subdomain_of: ""
page_count: 7
created: 2026-04-29
updated: 2026-04-29
tags:
  - meta
  - decisions
status: developing
related:
  - "[[index]]"
sources: []
---

# Decisions (ADRs)

Architecture Decision Records. Each ADR captures a *load-bearing* design call and its rationale. New decisions are appended; superseded decisions are marked with `status: superseded` and a link to the replacement.

## Decision pages

All seven seeded ADRs from `.raw/welcome.md` have been filed:

- [[0001-stage-3-decorators]] — use ECMAScript Stage-3 decorators; reject `experimentalDecorators` + `reflect-metadata`.
- [[0002-repository-with-lazy-query-builder]] — `Repository<T>` as the entry point; `find()` returns a lazy `QueryBuilder<T>`.
- [[0003-singleton-di-container]] — minimal singleton DI container, scoped to repository wiring.
- [[0004-parameterized-sql-only]] — `sql.unsafe` banned outright across the codebase.
- [[0005-no-any-type-driven-api]] — strict `@typescript-eslint/no-explicit-any`; type-driven public API.
- [[0006-tdd-rhythm]] — red-green-refactor with `bun test`, colocated `*.test.ts`.
- [[0007-bun-toolchain]] — Bun is the only toolchain (runtime, package manager, test runner, bundler).

## ADR naming convention

`NNNN-short-title.md` where `NNNN` is a zero-padded sequence: `0001-stage-3-decorators.md`, `0002-no-reflect-metadata.md`, ...

## Frontmatter for ADR pages

```yaml
type: decision
status: accepted        # proposed | accepted | superseded | deprecated
date: YYYY-MM-DD
deciders: ["Tony Albert"]
supersedes: []
superseded_by: ""
context: ""
```
