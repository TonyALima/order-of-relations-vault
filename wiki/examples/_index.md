---
type: domain
title: "Examples"
subdomain_of: ""
page_count: 3
created: 2026-04-29
updated: 2026-04-29
tags:
  - meta
  - examples
status: developing
related:
  - "[[index]]"
  - "[[brief]]"
sources:
  - "../.raw/drift-m6-examples-pointer.md"
---

# Examples

Wiki front-doors for the runnable scenarios in `examples/` — the canonical, in-repo demonstrations of how the API is actually called. Each scenario is self-contained: `db.ts` + `entities/` + `services/` + `index.ts`.

> [!key-insight] Why these pages exist
> Without a wiki front-door, `examples/` was invisible to agents reading the wiki — every "how do I actually call this API?" lookup forced a grep against the repo. These pages mirror what `[[sources/_index]]` does for `.raw/`: every external artifact gets a wiki node that names what it demonstrates and links into the relevant concepts.

## Pages

- [[examples/basic-crud]] — the minimum-viable CRUD scenario. `@Entity`, `@Column`, `@PrimaryColumn`, `@NotNullable`, `Repository<T>`, `create` / `findMany` / `findById`.
- [[examples/inheritance]] — single-table inheritance with `User`/`AdminUser`. **The only place `InheritanceSearchType` is shown in use** — `findMany({ inheritance: SUBCLASSES })`.
- [[examples/relations]] — `@ToOne` with FK column auto-derivation and the SERIAL→INTEGER FK type demotion (see [[modules/sql-types]]).

## Conventions

- One wiki page per `examples/<scenario>/` directory.
- Each page lists the decorators / concepts the scenario exercises and deep-links into the relevant files.
- When a wiki page elsewhere needs a "how is this actually called?" anchor, link `examples/<scenario>/services/<File>.ts` (or the corresponding `[[examples/<name>]]` page).

See [[sources/drift-m6-examples-pointer]] for the drift note that motivated this section.
