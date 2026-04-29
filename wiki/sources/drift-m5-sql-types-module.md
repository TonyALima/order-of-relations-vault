---
type: source
title: "Drift M5 — sql-types module is undocumented"
source_type: drift-correction
author: "Tony Albert"
date_published: 2026-04-29
url: ""
confidence: high
key_claims:
  - "src/core/sql-types/ exports COLUMN_TYPE (closed enum, ~50 PG types), getColumnTypeDefinition (DDL fragment generator), and toForeignKeyType (SERIAL→INTEGER demotion)."
  - "COLUMN_TYPE is the type-system surface for DDL identifiers — closed, no string escape hatch."
  - "toForeignKeyType demotes SERIAL→INTEGER, SMALLSERIAL→SMALLINT, BIGSERIAL→BIGINT; identity for everything else."
  - "FK columns referencing a SERIAL PK are emitted as INTEGER — they don't own the sequence."
  - "Called from MetadataStorage.resolveRelations when the relation has no explicit FK columns."
created: 2026-04-29
updated: 2026-04-29
tags:
  - source
  - drift
status: stable
related:
  - "[[modules/sql-types]]"
  - "[[Autogeneration]]"
  - "[[Relation Target Thunk]]"
  - "[[MetadataStorage]]"
sources:
  - "../.raw/drift-m5-sql-types-module.md"
---

# Drift M5 — sql-types module is undocumented

## Summary

`src/core/sql-types/sql-types.ts` is referenced obliquely by [[Autogeneration]] (mentions SERIAL) and [[brief]] (mentions COLUMN_TYPE), but the module — the full enum, the DDL fragment generator, and especially the `toForeignKeyType` SERIAL→INTEGER demotion rule — had no dedicated wiki coverage. Closing this gap is high-leverage: anyone wiring a relation against a SERIAL PK needs to know the FK column will be emitted as `INTEGER`, not `SERIAL`.

## What the Module Exports

### `enum COLUMN_TYPE`

A closed enum of ~50 PostgreSQL types — numeric (incl. SERIAL/SMALLSERIAL/BIGSERIAL), character/binary, date/time (with zone variants), boolean, UUID, JSON/JSONB, XML, geometric, network, bit-string, text-search, range. The entire universe of column types decorators can declare. No string escape hatch — passing a non-`COLUMN_TYPE` member is a compile error.

This is the type-system surface for DDL identifiers — it parallels [[Parameterized SQL]] for DML.

### `getColumnTypeDefinition(sql, type)`

A `switch` that maps each `COLUMN_TYPE` to a parameterized SQL fragment using Bun's tagged-template form. Default branch throws `UnsupportedColumnTypeError`. Called by `Database.createBaseTables()` for per-column DDL.

### `toForeignKeyType(type)` — the non-obvious one

```ts
case COLUMN_TYPE.SERIAL:      return COLUMN_TYPE.INTEGER;
case COLUMN_TYPE.SMALLSERIAL: return COLUMN_TYPE.SMALLINT;
case COLUMN_TYPE.BIGSERIAL:   return COLUMN_TYPE.BIGINT;
default:                       return type;
```

Why: `SERIAL` is PostgreSQL syntactic sugar for an `INTEGER` column with a sequence and `DEFAULT nextval(...)`. A FK column *referencing* a SERIAL PK doesn't own the sequence — it just stores the integer value. Emitting `SERIAL` on the FK would create a bogus second sequence and attach a `DEFAULT` to a column that should always copy the parent row's id.

Called from `MetadataStorage.resolveRelations()` when the relation hasn't been given explicit FK columns:

```ts
relation.columns = primaryColumns.map((pk) => ({
  name: `${relation.propertyName}_${pk.propertyName}`,
  type: toForeignKeyType(pk.type),
  referencedProperty: pk.propertyName,
}));
```

The user never sees the transform — they declare `@ToOne(() => User)` and the FK column type is derived. But anyone reasoning about emitted DDL needs to know.

## Why the Drift Mattered

- **STI + relations.** A reader of [[Single-Table Inheritance]] + [[Relation Target Thunk]] + [[Autogeneration]] still wouldn't know the SERIAL-demotion rule. They might write a manual migration with a `SERIAL` FK that fights the parent's sequence.
- **Why SERIAL for `dbSide` PKs?** [[Autogeneration]] gave `SERIAL + dbSide: () => undefined` as canonical — but the *reason* is in `getColumnTypeDefinition` (SERIAL emits the DDL that creates the sequence). Without a sql-types page, that connection was implicit.

## Resolution

- [[modules/sql-types]] — new page covering the closed enum, DDL fragment generation, and FK type demotion.
- [[Autogeneration]] — sentence explaining SERIAL choice and FK demotion, with cross-link.
- [[Relation Target Thunk]] — step 3 of "When the Thunk Is Called" expanded with `toForeignKeyType` mention.
- [[MetadataStorage]] — type-demotion clause inline in resolveRelations description.
- [[brief]] — bullet documenting the SERIAL→INTEGER FK rule.

## Source Citations

- `src/core/sql-types/sql-types.ts` lines 5–53 — COLUMN_TYPE enum.
- `src/core/sql-types/sql-types.ts` lines 55–66 — toForeignKeyType.
- `src/core/sql-types/sql-types.ts` lines 68–167 — getColumnTypeDefinition.
- `src/core/sql-types/sql-types.errors.ts` — UnsupportedColumnTypeError.
- `src/core/metadata/metadata.ts` lines 84–103 — resolveRelations, the call site for toForeignKeyType.
