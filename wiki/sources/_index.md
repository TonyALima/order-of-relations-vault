---
type: domain
title: "Sources"
subdomain_of: ""
page_count: 5
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

> [!check] All five sources ingested
> - ✅ `welcome.md` → [[sources/welcome]]
> - ✅ `architecture-overview.md` → [[sources/architecture-overview]]
> - ✅ `decorator-metadata-storage.md` → [[sources/decorator-metadata-storage]]
> - ✅ `query-builder-design.md` → [[sources/query-builder-design]]
> - ✅ `repository-contract.md` → [[sources/repository-contract]]

## Source pages

- [[sources/welcome|Welcome to the OOR Vault]] — manifesto: pillars + 7 ADR seeds. (2026-04-29)
- [[sources/architecture-overview|Architecture Overview]] — five-layer stack, query lifecycle, schema create/drop; corrects three claims from welcome.md. (2026-04-29)
- [[sources/decorator-metadata-storage|Decorator Metadata Storage]] — pin-down on storage shape, three-symbol scheme (resolved against code), lazy resolution, failure modes. (2026-04-29)
- [[sources/query-builder-design|Query Builder Design]] — mutable single-owner builder, `where`-callback signature, SQL-composition safety, two terminal methods today. Corrects two claims from earlier wiki pages. (2026-04-29)
- [[sources/repository-contract|Repository Contract]] — by-key operations, `requirePrimaryKey` gate, `create(entity: T)` type-shaped contract, autogeneration strategies, single error type. Sharpens the read/write split. (2026-04-29)
