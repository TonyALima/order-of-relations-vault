---
type: source
title: "Drift M3 — module pages backlog"
source_type: drift-correction
author: "Tony Albert"
date_published: 2026-04-29
url: ""
confidence: medium
key_claims:
  - "wiki/modules/ is empty; the index acknowledges this as a backlog item."
  - "src/ has 11 leaf directories, not 8 as the modules/_index claims."
  - "Three modules with no existing wiki coverage are high-leverage: src/core/sql-types/, src/core/orm-error/, src/decorators/relation/, src/decorators/nullable/."
  - "Path A: file per-module pages (some stubs redirecting to existing component pages)."
  - "Path B: retire wiki/modules/ entirely and lean on Layered Architecture + component pages."
created: 2026-04-29
updated: 2026-04-29
tags:
  - source
  - drift
  - backlog
status: partial
related:
  - "[[modules/_index]]"
  - "[[Layered Architecture]]"
sources:
  - "../.raw/drift-m3-module-pages.md"
---

# Drift M3 — module pages backlog

## Summary

[[index]] currently says *"Module pages not yet filed; the [[modules/_index]] holds the source-tree map (8 directories cataloged with their concerns and layer assignments)."* This drift note revisits that backlog: the source tree has **11** leaf directories (not 8), and four of them have no other wiki coverage at all.

## Source Tree Today

```
src/
├── core/
│   ├── database/        (component: [[Database]])
│   ├── repository/      (component: [[Repository]])
│   ├── metadata/        (component: [[MetadataStorage]])
│   ├── sql-types/       — uncovered until [[modules/sql-types]]
│   ├── utils/           — sqlJoin covered; Constructor type helpers uncovered
│   └── orm-error/       — uncovered (abstract OrmError base)
├── decorators/
│   ├── entity/          (flow: [[entity-registration]])
│   ├── column/          (concept: [[Autogeneration]])
│   ├── nullable/        — partly mentioned, no dedicated page
│   └── relation/        (concept: [[Relation Target Thunk]] covers thunk only)
└── query-builder/       (component: [[QueryBuilder]])
```

## Two Paths

### Path A — close the backlog by filing module pages

Each leaf gets a uniform-shape page (Module name, Layer, Exports, Owns, Reads/Writes, See also). Stubs are fine where a component page already covers substance.

### Path B — close the bucket and delete `wiki/modules/`

If per-module pages don't carry weight that [[Layered Architecture]] + component pages don't already carry, replace the index entry with: *"There are no per-module pages; module-level concerns are covered by [[Layered Architecture]] (boundaries) and the per-component / per-concept pages (substance)."*

## Decision (2026-04-29)

**Mixed: Path A for high-leverage modules, defer the rest.**

- [[modules/sql-types]] — filed (see [[drift-m5-sql-types-module]]).
- The remaining stubs (`database`, `repository`, `metadata`, `query-builder`, `decorators-entity`, `decorators-column`, `utils`) — deferred. Their component / concept pages already carry the weight; cheap to add later if the navigation pattern proves valuable.
- `orm-error`, `decorators-nullable`, `decorators-relation` — flagged for a future drift pass; these have substantive uncovered surface.

[[index]]'s "8 directories" → "11 directories" miscount fixed in [[modules/_index]].

## Source Citations

- `src/` directory layout: enumerated above.
- `wiki/index.md` — the "Module pages not yet filed" sentence.
- `wiki/modules/_index.md` — the source-tree map with the off-by-three count.
