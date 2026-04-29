---
type: domain
title: "Modules"
subdomain_of: ""
page_count: 1
created: 2026-04-29
updated: 2026-04-29
tags:
  - meta
  - modules
status: developing
related:
  - "[[index]]"
  - "[[Layered Architecture]]"
sources:
  - "../.raw/architecture-overview.md"
  - "../.raw/drift-m3-module-pages.md"
---

# Modules

One note per major module / package / service in the OOR codebase. Each entry maps to a real directory under [[order-of-relations]]'s `src/`.

> The source-tree map below is the authoritative catalog of which directory holds what (ingested from `.raw/architecture-overview.md` § "Source Tree, by Concern"). The table groups by concern, not leaf — the actual `src/` tree has **11 leaf directories** (6 under `core/`, 4 under `decorators/`, plus `query-builder/`). See [[sources/drift-m3-module-pages]] for the per-leaf inventory and the partial-Path-A decision (file pages where there's substantive uncovered surface; defer the rest).

## Source-tree map

| Path | Concern | Direction |
|---|---|---|
| `src/decorators/` | Every decorator (`@Entity`, `@Column`, `@PrimaryColumn`, `@ToOne`, `@Nullable`, `@NotNullable`). Writes only to `context.metadata` and (for `@Entity`) the database's metadata storage. | Layer 1 (writers) |
| `src/core/metadata/` | [[MetadataStorage]], resolved `EntityMetadata` / `ColumnMetadata` / `RelationMetadata` types, `resolveInheritance` and `resolveRelations` passes. | Layer 2 |
| `src/core/database/` | [[Database]]: Bun `SQL` connection wrapper, schema `create()` / `drop()`, FK-aware drop ordering. | Layer 5 (wire edge) |
| `src/core/repository/` | `Repository<T>`: entity-shaped CRUD surface, primary-key validation, write-path SQL. | Layer 3 |
| `src/core/sql-types/` | `COLUMN_TYPE` enum and the SQL fragment each type maps to. | Shared |
| `src/core/utils/` | `sqlJoin` (the only sanctioned way to join SQL fragments) and shared types. | Shared |
| `src/query-builder/` | `QueryBuilder<T>`: [[Conditions Proxy]], `FindOptions`, inheritance search modes. | Layer 4 |
| `src/errors.ts` + per-module `*.errors.ts` | Every error type extends a single `OrmError` base, exported from the package root. | Cross-cutting |

## The layering rule (operational form)

Per [[Layered Architecture]]:

- **Writers of metadata** go in `decorators/`.
- **Readers of metadata** go in `core/` or `query-builder/`.
- **Nothing under `core/metadata/` is allowed to import from above.**

When in doubt about where new code belongs, the import direction makes the answer mechanical.

## Module pages

- [[modules/sql-types]] — `COLUMN_TYPE` enum, `getColumnTypeDefinition` (the only DDL fragment producer), `toForeignKeyType` (SERIAL→INTEGER FK demotion).

### Deferred (handled elsewhere or low-leverage)

| Module | Coverage | Status |
|---|---|---|
| `src/core/database/` | [[Database]] component page | Component page covers it; module stub deferred. |
| `src/core/repository/` | [[Repository]] component page | Component page covers it; module stub deferred. |
| `src/core/metadata/` | [[MetadataStorage]] component page | Component page covers it; module stub deferred. |
| `src/query-builder/` | [[QueryBuilder]] component page | Component page covers it; module stub deferred. |
| `src/core/utils/` | [[sqlJoin]] concept covers half | `Constructor` type helpers uncovered — file later if useful. |
| `src/core/orm-error/` | None | High-leverage; file in a future drift pass. |
| `src/decorators/entity/` | [[entity-registration]] flow | Flow covers it; module stub deferred. |
| `src/decorators/column/` | [[Autogeneration]] concept | Concept covers `@PrimaryColumn`/`@Column` interactions. |
| `src/decorators/nullable/` | Mentioned in [[Repository]], [[Repository Pattern]] | `NullableField`/`NotNullableField` mechanic is load-bearing — file later. |
| `src/decorators/relation/` | [[Relation Target Thunk]] covers thunk only | `@ToOne(options)` and `OneToOneOptions.foreignKeys` undocumented — file later. |

## Frontmatter for module pages

```yaml
type: module
path: "src/<dir>/"
status: active
language: typescript
purpose: ""
maintainer: ""
last_updated: YYYY-MM-DD
linked_issues: []
depends_on: []
used_by: []
```
