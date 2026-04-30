---
type: domain
title: "Comparisons"
subdomain_of: ""
page_count: 5
created: 2026-04-29
updated: 2026-04-30
tags:
  - meta
  - comparisons
status: developing
related:
  - "[[index]]"
sources: []
---

# Comparisons

Side-by-side analyses framed as TCC defense material. Each page answers: *what does OOR contribute that the field didn't already have, and why does it matter?*

> [!convention] These are not consumer-decision aids
> Comparisons in this folder are written from OOR's point of view, defending the contribution. They are not "should I pick OOR or X?" decision trees, and they do not include "when to reach for which" or "where OOR is behind" sections. Project-maturity comparisons are also out of scope — a TCC vs an established multi-year project is structurally meaningless on that axis.

## Comparison pages

### Summary matrix

- [[orms-summary]] — single feature matrix of OOR's contributions across all three competitors. The at-a-glance scoreboard; cross-links to the long-form pages for the thesis behind each row.

### ORM vs ORM

- [[oor-vs-typeorm]] — TypeORM-shaped ergonomics with modern guarantees. The contribution is lifting the constraints (legacy decorators, `reflect-metadata`, raw SQL fragments, `any` at the parameter seam) that TypeORM accumulated as the early default.
- [[oor-vs-drizzle]] — the OO ergonomic profile, modernized. Drizzle bends the type system to fit SQL; OOR keeps class-as-schema, single-entry-point-per-entity, native polymorphism — the path Drizzle deliberately walked away from.
- [[oor-vs-prisma]] — TypeScript itself as the schema. The contribution is exploring the path Prisma's design closed off: schema-as-code (no DSL, no codegen, no generated artifacts) with a fluent composable builder over class-declared entities.

### Decorator dialect

- [[stage-3-vs-legacy-decorators]] — the foundational dialect choice OOR rests on. Stage-3 is the standardization-track, polyfill-free, toolchain-aligned dialect; legacy is what the major decorator-using libraries are stuck on by architectural lock-in. OOR's bet is being early on the right side of that transition.

## Convention for new comparison pages

Every page in this folder follows the same skeleton:

1. **Frontmatter** — `subjects:` (two), `dimensions:` (~10), `verdict:` (one-line conclusion that reads as a contribution claim, not a hedged decision aid).
2. **`## What OOR brings that's new`** — bullet list at the top, 3–5 contribution claims. Reader should know OOR's case before the comparison table.
3. **`## Overview`** — what question this comparison answers about OOR's place in the field. Optional `> [!key-insight]` callout naming the thesis.
4. **`## Comparison`** — dimensions table, ~10 rows. Treat planned-and-well-scoped features as equal-value (cross-link the open-question page).
5. **`## OOR's contribution, dimension by dimension`** — long-form prose, one subsection per major axis. Frame as positive contributions, not neutral observations.
6. **`## Why OOR matters in a crowded market`** — closing argument; the TCC defense in one section.
7. **`## Sources`** — upstream doc URLs, ADR cross-links.

