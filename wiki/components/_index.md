---
type: domain
title: "Components"
subdomain_of: ""
page_count: 4
created: 2026-04-29
updated: 2026-04-29
tags:
  - meta
  - components
status: developing
related:
  - "[[index]]"
  - "[[modules/_index]]"
sources: []
---

# Components

Reusable building blocks: decorators, classes, functions, generic types. Distinct from `modules/` — a module is a *bag of code*, a component is a *named, reusable thing*.

## Component pages

- [[MetadataStorage]] — `Map<Constructor, EntityMetadata>` owned by a `Database`; written only by decorators, frozen after class evaluation.
- [[Database]] — connection wrapper + metadata host + schema lifecycle (`create()`/`drop()`).
- [[QueryBuilder]] — mutable single-owner builder; `applyOptions()` (replace) → `getMany()` / `getOne()`. Internal state is a single `conditions: Condition[]` array.
- [[Repository]] — narrow by-key surface (`create`, `findById`, `update`, `delete`), single `requirePrimaryKey` gate, single error type (`IncompletePrimaryKeyError`). `find()` hands off a builder.

> [!gap] Still to come
> Decorator components (`@Entity`, `@Column`, `@PrimaryColumn`, `@ToOne`, `@Nullable`), `Container` (when DI ships) — pending future sources or code spot-checks.
