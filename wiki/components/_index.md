---
type: domain
title: "Components"
subdomain_of: ""
page_count: 2
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

> [!gap] Still to come
> Decorator components (`@Entity`, `@Column`, `@PrimaryColumn`, `@ToOne`, `@Nullable`), `Repository<T>`, `QueryBuilder<T>`, `Container` (when DI ships) — populated as the deeper `.raw/` notes are ingested.
