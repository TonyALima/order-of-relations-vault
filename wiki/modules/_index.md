---
type: domain
title: "Modules"
subdomain_of: ""
page_count: 0
created: 2026-04-29
updated: 2026-04-29
tags:
  - meta
  - modules
status: seed
related:
  - "[[index]]"
sources: []
---

# Modules

One note per major module / package / service in the OOR codebase. Each entry should map to a real directory or top-level concern under `src/`.

> [!gap] No modules ingested yet
> Run an ingest pass over `.raw/architecture-overview.md` to seed initial module pages: `decorators`, `repository`, `query-builder`, `container`, `migrations`, `database`.

## Module pages

- _none yet_

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
