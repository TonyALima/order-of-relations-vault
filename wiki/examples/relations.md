---
type: example
title: "examples/relations"
example_path: "examples/relations/"
created: 2026-04-29
updated: 2026-04-29
tags:
  - example
  - relations
  - foreign-keys
status: stub
related:
  - "[[Relation Target Thunk]]"
  - "[[modules/sql-types]]"
  - "[[MetadataStorage]]"
sources:
  - "../.raw/drift-m6-examples-pointer.md"
---

# examples/relations

The `@ToOne` / relations scenario. Demonstrates FK column-name auto-derivation, the relation-target thunk pattern, and the SERIAL→INTEGER FK type demotion.

> [!gap] Stub page
> The `.raw/drift-m6-examples-pointer.md` source did not open this scenario's files in the drift pass — the contents below are the expected shape based on the rest of the codebase. Update this page once the example files are read directly.

## What It Should Demonstrate (expected)

- `@ToOne(() => Target)` with the closure-style target reference (see [[Relation Target Thunk]]).
- Auto-derived FK column names (`relationProperty + '_' + targetPkProperty`).
- SERIAL→INTEGER FK type demotion when the target's PK is `SERIAL` (see [[modules/sql-types]] § Foreign-key type demotion).
- The two-pass schema emission: `CREATE TABLE` for both entities first, then `ALTER TABLE … ADD FOREIGN KEY` (see [[schema-create-drop]]).

## See Also

- [[Relation Target Thunk]] — the `() => Target` closure pattern and TDZ rationale.
- [[modules/sql-types]] § Foreign-key type demotion — what the FK column's type ends up as.
- [[MetadataStorage]] — `resolveRelations` is where FK column names and types are derived.
- [[schema-create-drop]] — the two-pass DDL emission this scenario triggers.
- [[sources/drift-m6-examples-pointer]] — why this page exists.
