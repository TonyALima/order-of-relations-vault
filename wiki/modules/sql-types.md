---
type: module
title: "modules/sql-types"
path: "src/core/sql-types/"
status: active
language: typescript
purpose: "Closed COLUMN_TYPE enum, DDL fragment generator, foreign-key type demotion."
maintainer: "Tony Albert"
last_updated: 2026-04-29
created: 2026-04-29
updated: 2026-04-29
tags:
  - module
  - sql-types
  - ddl
related:
  - "[[Autogeneration]]"
  - "[[Relation Target Thunk]]"
  - "[[MetadataStorage]]"
  - "[[Parameterized SQL]]"
  - "[[sources/drift-m5-sql-types-module]]"
linked_issues: []
depends_on: []
used_by:
  - "src/core/database/"
  - "src/core/metadata/"
sources:
  - "../.raw/drift-m5-sql-types-module.md"
---

# modules/sql-types

## Purpose

`src/core/sql-types/` is the **type-system surface for DDL identifiers** — the closed inventory of PostgreSQL types decorators are allowed to declare, plus the helpers that turn each type into either a SQL fragment or its foreign-key counterpart.

It parallels [[Parameterized SQL]] for DML: just as values pass through the driver as bound parameters (never string-concatenated), every DDL type token is enum-bound (never string-supplied).

## Exports

### `enum COLUMN_TYPE`

A closed enum of ~50 PostgreSQL types — the entire universe a `@Column` / `@PrimaryColumn` is allowed to declare.

| Group | Members |
|---|---|
| Numeric | `SMALLINT`, `INTEGER`, `BIGINT`, `SMALLSERIAL`, `SERIAL`, `BIGSERIAL`, `DECIMAL`, `NUMERIC`, `REAL`, `DOUBLE_PRECISION`, `MONEY` |
| Character / binary | `CHAR`, `VARCHAR`, `TEXT`, `BYTEA` |
| Date / time | `DATE`, `TIME`, `TIME_WITH_TIME_ZONE`, `TIMESTAMP`, `TIMESTAMP_WITH_TIME_ZONE`, `INTERVAL` |
| Boolean & UUID | `BOOLEAN`, `UUID` |
| Structured | `JSON`, `JSONB`, `XML` |
| Geometric | `POINT`, `LINE`, `LSEG`, `BOX`, `PATH`, `POLYGON`, `CIRCLE` |
| Network | `CIDR`, `INET`, `MACADDR`, `MACADDR8` |
| Bit-string | `BIT`, `BIT_VARYING` |
| Text-search | `TSVECTOR`, `TSQUERY` |
| Range | `INT4RANGE`, `INT8RANGE`, `NUMRANGE`, `TSRANGE`, `TSTZRANGE`, `DATERANGE` |

**No string escape hatch.** Passing a string that isn't a `COLUMN_TYPE` member is a compile error. This is what backs [[0004-parameterized-sql-only]] for DDL identifiers — the type system rules out user-supplied DDL fragments at their source.

### `getColumnTypeDefinition(sql, type)`

A `switch` from each `COLUMN_TYPE` to a parameterized SQL fragment using Bun's tagged-template form (e.g. `COLUMN_TYPE.TIMESTAMP_WITH_TIME_ZONE` → `` sql`TIMESTAMP WITH TIME ZONE` ``). Default branch throws `UnsupportedColumnTypeError`.

This is the **only** DDL-fragment producer in the codebase. Everything `Database.createBaseTables` emits at the column level flows through it.

### `toForeignKeyType(type)`

The non-obvious one. When a relation's foreign-key column inherits its type from the target's primary-key column, SERIAL types are **demoted** to their underlying integer type:

```ts
case COLUMN_TYPE.SERIAL:      return COLUMN_TYPE.INTEGER;
case COLUMN_TYPE.SMALLSERIAL: return COLUMN_TYPE.SMALLINT;
case COLUMN_TYPE.BIGSERIAL:   return COLUMN_TYPE.BIGINT;
default:                       return type;  // identity for everything else
```

> [!key-insight] Why FK columns can't be SERIAL
> `SERIAL` is PostgreSQL syntactic sugar for `INTEGER` plus a sequence and a `DEFAULT nextval(...)`. A foreign-key column that *references* a SERIAL primary key doesn't own the sequence — it just stores the integer. Emitting `SERIAL` on the FK would (a) create a bogus second sequence, and (b) attach a `DEFAULT` to a column that should always copy the parent row's id. Demoting to the underlying integer type is the correct DDL.

The user never sees this transform — they declare `@ToOne(() => User)` and the FK column type is derived. But anyone reasoning about the emitted DDL needs to know: *if `User.id` is SERIAL, the `Post.author_id` FK column will be emitted as INTEGER.*

## Call Sites

- `MetadataStorage.resolveRelations()` (`src/core/metadata/metadata.ts` lines 84–103) calls `toForeignKeyType(pk.type)` when the relation has no explicit FK columns:

  ```ts
  relation.columns = primaryColumns.map((pk) => ({
    name: `${relation.propertyName}_${pk.propertyName}`,
    type: toForeignKeyType(pk.type),
    referencedProperty: pk.propertyName,
  }));
  ```

- `Database.createBaseTables()` (`src/core/database/database.ts` lines 48–58) calls `getColumnTypeDefinition(sql, type)` per column when emitting `CREATE TABLE`.

## Errors

- `UnsupportedColumnTypeError` — thrown by `getColumnTypeDefinition`'s default branch. Defined in `src/core/sql-types/sql-types.errors.ts`.

## Why This Is a Module Page

Three modules in `src/` had no dedicated wiki coverage at the time of the [[sources/drift-m3-module-pages|drift-m3]] pass; `sql-types` was the most load-bearing of the three (the SERIAL→INTEGER demotion is invisible from any other angle and bites anyone hand-writing migrations against the emitted DDL). Filing this page closes that gap; see [[sources/drift-m5-sql-types-module]].

## Connections

- [[Autogeneration]] — explains why `SERIAL + dbSide: () => undefined` is the canonical PK choice; this page explains the FK side of the same decision.
- [[Relation Target Thunk]] — `resolveRelations` (which calls the thunk) is also where `toForeignKeyType` runs.
- [[MetadataStorage]] — owns the resolution pass that calls these helpers.
- [[Parameterized SQL]] / [[0004-parameterized-sql-only]] — same safety property, applied at the DDL layer.
- [[schema-create-drop]] — the flow that consumes these helpers.

## Sources

- `.raw/drift-m5-sql-types-module.md` — the drift note that motivated this page.
- `src/core/sql-types/sql-types.ts` — the implementation.
- `src/core/sql-types/sql-types.errors.ts` — the error type.
