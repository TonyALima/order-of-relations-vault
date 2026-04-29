# Drift Correction — `src/core/sql-types/` is undocumented (and `toForeignKeyType` is non-obvious)

`src/core/sql-types/sql-types.ts` is referenced obliquely by `[[Autogeneration]]` (which mentions SERIAL) and the brief (which mentions `COLUMN_TYPE`). But the module itself — the full enum, the DDL fragment generator, and especially the `toForeignKeyType` SERIAL→INTEGER demotion rule used during relation resolution — has no dedicated wiki coverage. Adding it would close one of the more load-bearing gaps in the wiki: anyone wiring a relation against a SERIAL PK needs to know that the FK column will be emitted as `INTEGER`, not `SERIAL`.

## What the code actually exposes

The module exports three things, all from `sql-types.ts`:

### 1. `enum COLUMN_TYPE`

A closed enum of ~50 PostgreSQL types. The full inventory at the time of writing:

- **Numeric:** `SMALLINT`, `INTEGER`, `BIGINT`, `SMALLSERIAL`, `SERIAL`, `BIGSERIAL`, `DECIMAL`, `NUMERIC`, `REAL`, `DOUBLE_PRECISION`, `MONEY`.
- **Character/binary:** `CHAR`, `VARCHAR`, `TEXT`, `BYTEA`.
- **Date/time:** `DATE`, `TIME`, `TIME_WITH_TIME_ZONE`, `TIMESTAMP`, `TIMESTAMP_WITH_TIME_ZONE`, `INTERVAL`.
- **Boolean & UUID:** `BOOLEAN`, `UUID`.
- **Structured:** `JSON`, `JSONB`, `XML`.
- **Geometric:** `POINT`, `LINE`, `LSEG`, `BOX`, `PATH`, `POLYGON`, `CIRCLE`.
- **Network:** `CIDR`, `INET`, `MACADDR`, `MACADDR8`.
- **Bit-string:** `BIT`, `BIT_VARYING`.
- **Text-search:** `TSVECTOR`, `TSQUERY`.
- **Range:** `INT4RANGE`, `INT8RANGE`, `NUMRANGE`, `TSRANGE`, `TSTZRANGE`, `DATERANGE`.

The enum is the entire universe of column types decorators can declare. There is no escape hatch — passing a string that isn't a `COLUMN_TYPE` member is a compile error. This is the type-system surface that backs the `[[0004-parameterized-sql-only]]` invariant for DDL: every DDL identifier is enum-bound, not user-string-bound.

### 2. `getColumnTypeDefinition(sql, type)`

A `switch` that maps each `COLUMN_TYPE` to a parameterized SQL fragment using Bun's tagged-template form (e.g. `COLUMN_TYPE.TIMESTAMP_WITH_TIME_ZONE` → `` sql`TIMESTAMP WITH TIME ZONE` ``). The default branch throws `UnsupportedColumnTypeError`. This function is what `Database.createBaseTables()` calls when emitting the per-column DDL fragment.

### 3. `toForeignKeyType(type)`

The non-obvious one. When a relation's foreign-key column inherits its type from the target's primary-key column, SERIAL types are **demoted**:

```ts
case COLUMN_TYPE.SERIAL:      return COLUMN_TYPE.INTEGER;
case COLUMN_TYPE.SMALLSERIAL: return COLUMN_TYPE.SMALLINT;
case COLUMN_TYPE.BIGSERIAL:   return COLUMN_TYPE.BIGINT;
default:                       return type;  // identity for everything else
```

Why: `SERIAL` is PostgreSQL syntactic sugar for an `INTEGER` column with a sequence and a `DEFAULT nextval(...)`. A foreign-key column that *references* a SERIAL PK does not own the sequence — it just stores the integer value. Emitting `SERIAL` on the FK column would (a) create a bogus second sequence, and (b) attach a `DEFAULT` to a column that should always be set to the parent row's id. Demoting to the underlying integer type is the correct DDL.

`toForeignKeyType` is called from `MetadataStorage.resolveRelations()` when the relation hasn't been given explicit FK columns:

```ts
relation.columns = primaryColumns.map((pk) => ({
  name: `${relation.propertyName}_${pk.propertyName}`,
  type: toForeignKeyType(pk.type),
  referencedProperty: pk.propertyName,
}));
```

The user never sees this transform — they declare `@ToOne(() => User)` and the FK column type is derived. But anyone reasoning about the emitted DDL needs to know: *"if `User.id` is SERIAL, the `Post.author_id` FK column will be emitted as INTEGER."*

## Why this matters

Two specific places where the absence of wiki coverage shows up:

- **STI + relations.** A user who reads `[[Single-Table Inheritance]]` and `[[Relation Target Thunk]]` and `[[Autogeneration]]` still doesn't know the SERIAL-demotion rule. They might write a manual migration matching their entity declarations and end up with a `SERIAL` FK that fights with the parent's sequence.
- **`@PrimaryColumn` type choice.** The `[[Autogeneration]]` page documents the difference between `clientSide` and `dbSide`, and gives `SERIAL + dbSide: () => undefined` as the canonical example. But why `SERIAL` and not `INTEGER`? The answer is in `getColumnTypeDefinition` — `SERIAL` emits the DDL that creates the sequence. Without a sql-types page, that connection is implicit.

## Suggested wiki coverage

A new `wiki/modules/sql-types.md` (or `wiki/concepts/Column Types.md` if you prefer the concept bucket — they overlap), with three sections:

1. **The closed `COLUMN_TYPE` enum.** Brief inventory grouped as above. Make the "no string escape hatch" point explicit — this is the type-system surface for DDL identifiers and parallels `[[Parameterized SQL]]` for DML.
2. **DDL fragment generation.** One-paragraph description of `getColumnTypeDefinition`. Note that it's the only DDL-fragment producer; everything in `Database.createBaseTables` flows through it.
3. **Foreign-key type demotion.** Document `toForeignKeyType`. Show the three demotions (`SERIAL` → `INTEGER`, `SMALLSERIAL` → `SMALLINT`, `BIGSERIAL` → `BIGINT`) and the reasoning. Cross-link from `[[Autogeneration]]` and `[[Relation Target Thunk]]`.

## Wiki pages affected

| Page | Action |
|---|---|
| `wiki/modules/sql-types.md` (new, or `wiki/concepts/Column Types.md`) | Create the page above. |
| `wiki/concepts/Autogeneration.md` | After the `dbSide` SERIAL example, add a sentence: *"`SERIAL` is the canonical `dbSide` type because PostgreSQL emits an `INTEGER` column plus a sequence; foreign keys that reference this PK get the column type demoted to `INTEGER` automatically — see `[[modules/sql-types]]` § Foreign-key type demotion."* |
| `wiki/concepts/Relation Target Thunk.md` | The "When the Thunk Is Called" section step 3 says *"fill in foreign-key column names and other resolved fields."* — expand to mention type demotion: *"Step 3 also derives each FK column's type by calling `toForeignKeyType(pk.type)`, which demotes SERIAL/SMALLSERIAL/BIGSERIAL to their underlying integer types so the FK column doesn't claim its own sequence. See `[[modules/sql-types]]`."* |
| `wiki/components/MetadataStorage.md` | "Read Path & Lazy Resolution" describes `resolveRelations` filling in `null` FK columns. Add the type-demotion clause inline. |
| `wiki/index.md` | Add the new page to whichever section it lives in (`modules/` or `concepts/`). |
| `wiki/brief.md` | One bullet under "Method-shape facts": *"`COLUMN_TYPE` is closed; SERIAL/SMALLSERIAL/BIGSERIAL FKs are auto-demoted to their integer underlying types — see `[[modules/sql-types]]`."* — only if you want this in the agent-facing brief. Probably yes; it's the kind of "hidden but load-bearing" rule the brief exists for. |

## Source citations

- `src/core/sql-types/sql-types.ts` lines 5–53 — the `COLUMN_TYPE` enum.
- `src/core/sql-types/sql-types.ts` lines 55–66 — `toForeignKeyType` with the three demotions.
- `src/core/sql-types/sql-types.ts` lines 68–167 — `getColumnTypeDefinition` switch.
- `src/core/sql-types/sql-types.errors.ts` — `UnsupportedColumnTypeError` (the default-branch throw target).
- `src/core/metadata/metadata.ts` lines 84–103 — `resolveRelations`, the call site for `toForeignKeyType`.
- `src/core/database/database.ts` lines 48–58 — `getColumnTypeDefinition` call site in `createBaseTables`.
