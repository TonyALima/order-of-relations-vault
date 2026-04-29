---
type: source
title: "Drift M6 — examples/ is invisible to the wiki"
source_type: drift-correction
author: "Tony Albert"
date_published: 2026-04-29
url: ""
confidence: high
key_claims:
  - "examples/ ships three runnable scenarios (basic-crud, inheritance, relations); the wiki never references them."
  - "examples/inheritance/services/UserHierarchyService.ts is the only place InheritanceSearchType is shown in use."
  - "Adding a per-example wiki page treats examples/ the way wiki/sources/ treats .raw/ — every external artifact gets a wiki front-door."
  - "Cheapest improvement: a one-line pointer in brief and index. Larger improvement: per-scenario wiki pages with cross-links."
created: 2026-04-29
updated: 2026-04-29
tags:
  - source
  - drift
status: stable
related:
  - "[[examples/_index]]"
  - "[[brief]]"
sources:
  - "../.raw/drift-m6-examples-pointer.md"
---

# Drift M6 — examples/ is invisible to the wiki

## Summary

The repo ships an `examples/` directory with runnable canonical usage. The wiki — including the agent-facing [[brief]] — never references it. For an agent doing "how do I actually call this API?" lookups, that's a costly omission: the wiki describes the API in prose, but the only source-of-truth for a complete working call site is `examples/`.

## What examples/ Contains

Three subdirectories, each self-contained (`db.ts` + `entities/` + `services/` + `index.ts`):

### `examples/basic-crud/`

The minimum-viable CRUD scenario.

- `entities/User.ts` — `@Entity` + `@PrimaryColumn({ type: SERIAL, autogeneration: { dbSide: () => undefined } })` + two `@Column @NotNullable` text fields. The canonical "User" the wiki keeps mentioning.
- `services/UserService.ts` — `new Repository(User, db)` + `create`, `findMany`, `findById`. The no-DI direct-construction pattern that [[0003-singleton-di-container]] flags as "current state."
- `db.ts`, `index.ts` — wiring.

### `examples/inheritance/`

The single-table-inheritance scenario.

- `entities/User.ts` + `entities/AdminUser.ts` (`extends User`).
- `services/UserHierarchyService.ts` — **the canonical example of the `inheritance` `FindOptions` option** (see [[drift-d3-find-options-inheritance]]). Calls `findMany({ inheritance: InheritanceSearchType.SUBCLASSES })` for both base and subclass repository.

The most under-cited file in the codebase from the wiki's perspective.

### `examples/relations/`

The relations / `@ToOne` scenario. Demonstrates FK-column-name auto-derivation and the SERIAL→INTEGER FK type demotion documented in [[drift-m5-sql-types-module]].

## Why the Drift Mattered

- **For agents:** [[brief]] is loaded into every agent session. A one-line pointer to `examples/` shaves N grep operations off every "how do I use the API" question.
- **For the wiki:** several pages embed inline code samples that duplicate (or contradict) `examples/`. Linking lets samples shrink to "see `examples/<scenario>/services/<File>.ts`" while staying authoritative.
- **For drift detection:** if `examples/` is the stated authoritative source, drift detection becomes "do the wiki snippets match the example files?" — much cheaper than "do the snippets match the codebase as a whole."

## Resolution — chose the larger change

Filed a per-example wiki page set:

- [[examples/_index]]
- [[examples/basic-crud]]
- [[examples/inheritance]] — explicit about being the only place `InheritanceSearchType` is shown in use; cross-links [[Single-Table Inheritance]], [[QueryBuilder]], and the new inheritance section from [[drift-d3-find-options-inheritance]].
- [[examples/relations]]

Plus pointers in [[brief]] and [[index]]. The inheritance example is load-bearing enough that it deserves a dedicated wiki node.

## Source Citations

- `examples/basic-crud/{db.ts, index.ts, entities/User.ts, services/UserService.ts}`
- `examples/inheritance/{db.ts, index.ts, entities/{User,AdminUser}.ts, services/UserHierarchyService.ts}` — esp. `listSubClassUsers()` / `listSubClassAdmins()`.
- `examples/relations/` — to be expanded on a later read.
