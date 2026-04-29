---
type: domain
title: "Sources"
subdomain_of: ""
page_count: 12
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

> [!check] Twelve sources ingested
> **Primary (5):**
> - ✅ `welcome.md` → [[sources/welcome]]
> - ✅ `architecture-overview.md` → [[sources/architecture-overview]]
> - ✅ `decorator-metadata-storage.md` → [[sources/decorator-metadata-storage]]
> - ✅ `query-builder-design.md` → [[sources/query-builder-design]]
> - ✅ `repository-contract.md` → [[sources/repository-contract]]
>
> **Drift corrections (7):**
> - ✅ `drift-d1-repository-find.md` → [[sources/drift-d1-repository-find]]
> - ✅ `drift-d3-find-options-inheritance.md` → [[sources/drift-d3-find-options-inheritance]]
> - ✅ `drift-d4-conditions-proxy-operators.md` → [[sources/drift-d4-conditions-proxy-operators]]
> - ✅ `drift-d5-discriminator-index.md` → [[sources/drift-d5-discriminator-index]]
> - ✅ `drift-m3-module-pages.md` → [[sources/drift-m3-module-pages]]
> - ✅ `drift-m5-sql-types-module.md` → [[sources/drift-m5-sql-types-module]]
> - ✅ `drift-m6-examples-pointer.md` → [[sources/drift-m6-examples-pointer]]

## Source pages

- [[sources/welcome|Welcome to the OOR Vault]] — manifesto: pillars + 7 ADR seeds. (2026-04-29)
- [[sources/architecture-overview|Architecture Overview]] — five-layer stack, query lifecycle, schema create/drop; corrects three claims from welcome.md. (2026-04-29)
- [[sources/decorator-metadata-storage|Decorator Metadata Storage]] — pin-down on storage shape, three-symbol scheme (resolved against code), lazy resolution, failure modes. (2026-04-29)
- [[sources/query-builder-design|Query Builder Design]] — mutable single-owner builder, `where`-callback signature, SQL-composition safety, two terminal methods today. Corrects two claims from earlier wiki pages. (2026-04-29)
- [[sources/repository-contract|Repository Contract]] — by-key operations, `requirePrimaryKey` gate, `create(entity: T)` type-shaped contract, autogeneration strategies, single error type. Sharpens the read/write split. (2026-04-29)

### Drift corrections (post-audit)

- [[sources/drift-d1-repository-find|D1: Repository.find() does not exist]] — drops the `find()` clause from agent-facing surface; reframes `findOne` / `findMany` as the composition entry points. (2026-04-29)
- [[sources/drift-d3-find-options-inheritance|D3: FindOptions.inheritance]] — closes the largest documentation gap; adds `InheritanceSearchType` semantics + interaction with `where`. (2026-04-29)
- [[sources/drift-d4-conditions-proxy-operators|D4: FieldConditionBuilder operators]] — fixes the operator inventory (nine methods, uniform across types; no `neq`, no `before`/`after`/`matches`). (2026-04-29)
- [[sources/drift-d5-discriminator-index|D5: idx_discriminator]] — documents the implicit STI index, the STI base-table-only skip rule, and the latent name-collision risk. (2026-04-29)
- [[sources/drift-m3-module-pages|M3: module pages backlog]] — fixes the 8→11 leaf miscount; partial Path-A decision (file high-leverage modules, defer the rest). (2026-04-29)
- [[sources/drift-m5-sql-types-module|M5: sql-types module]] — files `[[modules/sql-types]]`; documents `toForeignKeyType` SERIAL demotion. (2026-04-29)
- [[sources/drift-m6-examples-pointer|M6: examples/ invisible]] — files `[[examples/_index]]` + per-scenario front-doors. (2026-04-29)
