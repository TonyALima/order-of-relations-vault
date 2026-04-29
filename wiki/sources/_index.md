---
type: domain
title: "Sources"
subdomain_of: ""
page_count: 3
created: 2026-04-29
updated: 2026-04-29
tags:
  - meta
  - sources
status: developing
related:
  - "[[index]]"
sources: []
---

# Sources

One synthesis page per item in `.raw/`. Each page summarizes the source, lists its key claims, and links to every wiki page derived from it.

> [!gap] Pending ingest
> `.raw/` currently holds five hand-written design notes. **Three ingested**, two pending:
> - ✅ `welcome.md` → [[sources/welcome]]
> - ✅ `architecture-overview.md` → [[sources/architecture-overview]]
> - ✅ `decorator-metadata-storage.md` → [[sources/decorator-metadata-storage]]
> - `query-builder-design.md`
> - `repository-contract.md`

## Source pages

- [[sources/welcome|Welcome to the OOR Vault]] — manifesto: pillars + 7 ADR seeds. (2026-04-29)
- [[sources/architecture-overview|Architecture Overview]] — five-layer stack, query lifecycle, schema create/drop; corrects three claims from welcome.md. (2026-04-29)
- [[sources/decorator-metadata-storage|Decorator Metadata Storage]] — pin-down on storage shape, two-symbol scheme, lazy resolution, failure modes; surfaces inter-source disagreement on `NULLABLE_KEY`. (2026-04-29)
