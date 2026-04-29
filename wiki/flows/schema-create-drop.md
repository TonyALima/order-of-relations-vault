---
type: flow
title: "Schema Create / Drop"
trigger: "Caller invokes Database.create() or Database.drop()"
outcome: "Database schema reflects the registered entities (or is torn down)"
created: 2026-04-29
updated: 2026-04-29
tags:
  - flow
  - schema
  - database
status: stable
related:
  - "[[Database]]"
  - "[[MetadataStorage]]"
  - "[[sources/architecture-overview]]"
sources:
  - "../.raw/architecture-overview.md"
---

# Schema Create / Drop

How a `Database` instance materializes (and tears down) the SQL schema corresponding to its registered entities. This is the schema-side counterpart to [[query-lifecycle]] on the read side.

> [!note] Scope
> This covers only `Database.create()` / `Database.drop()`. Migrations beyond create/drop (`ALTER` for schema evolution, data migrations, version tracking) live in `[[Schema Migrations]]` — currently a placeholder.

---

## `Database.create()` — two-pass emission

`create()` walks the entity metadata held by the `Database` (see [[MetadataStorage]]) and emits DDL in **two passes**:

### Pass 1 — base tables

For each entity in metadata, emit a `CREATE TABLE` statement.

- Columns come from `EntityMetadata.columns` (one DDL fragment per `ColumnMetadata`).
- Primary-key columns are flagged via `@PrimaryColumn`; the table-level `PRIMARY KEY` clause is composed from those.
- Foreign-key columns from relations are emitted as plain columns at this stage — **without** the FK constraint.

#### Two skip rules in `createBaseTables`

```ts
if (metadata.columns.length === 0) continue;
if (metadata.discriminator && metadata.discriminator !== metadata.tableName) continue;
```

- **Empty-columns skip:** entities with no columns are skipped (defensive — `@Entity` already rejects classes with no primary column).
- **STI base-table-only skip:** subclass entities (whose `tableName` was rewritten to the root's during `resolveInheritance`, so `discriminator !== tableName`) are skipped. **Only the root entity emits a `CREATE TABLE`.** This is the schema-create implementation of "STI = one table for the whole hierarchy." See [[Single-Table Inheritance]].

#### STI emissions (when `metadata.discriminator !== undefined`)

For an STI root, the base-tables pass emits three statements in order:

1. `CREATE TABLE <tableName> (<columns>, PRIMARY KEY (...))` — same as any entity.
2. `ALTER TABLE <tableName> ADD COLUMN discriminator TEXT NOT NULL;`
3. `CREATE INDEX idx_discriminator ON <tableName>(discriminator);`

The discriminator column **and** its index are added only when `meta.discriminator` is truthy — and `resolveInheritance` wipes the discriminator on entities alone in their table. So a single-class root gets neither the column nor the index until a sibling registers and metadata re-resolves.

> [!warning] Latent index-name collision
> `idx_discriminator` is **shared across every STI table** in a schema. Two STI hierarchies in the same schema → second `db.create()` collides on the duplicate index name. Today invisible (only one hierarchy in the examples); flagged as a known limitation.

### Pass 2 — foreign-key constraints

For each relation in metadata, emit `ALTER TABLE … ADD FOREIGN KEY (…)`.

The two-pass split exists because an `ALTER TABLE … ADD FOREIGN KEY` requires the referenced table to already exist. Emitting all base tables first removes ordering constraints from pass 1: tables can be created in any order without worrying about which entity references which.

## `Database.drop()` — topologically reversed

`drop()` cannot be the literal reverse of `create()` — dropping a table that another table's FK points at fails with a constraint violation.

Instead, `drop()`:

1. Builds the relation graph from metadata.
2. Computes a topological ordering where **referenced** tables precede **referrers**.
3. Walks that ordering in **reverse** — referrers first, then their FK targets.

The result: every foreign-key target is dropped *after* anything that references it. PostgreSQL's `DROP TABLE` succeeds at each step.

## Why both live on `Database`

`Database` already owns the connection (Bun `SQL`) and the metadata. Schema lifecycle is the third job because it needs both — DDL composition reads metadata, and execution writes to the wire. Splitting it onto a separate class would require passing both inputs around without payoff at this scope. See [[Database]].

## Failure Modes

| Where | What | Symptom |
|---|---|---|
| Pass 1 | Entity metadata missing a primary column | `@Entity` already rejects this at registration time, but if reached: `CREATE TABLE` would emit without `PRIMARY KEY` |
| Pass 2 | Relation references an unregistered entity | FK target table doesn't exist; `ALTER TABLE` fails on the wire |
| `drop()` | Cycle in the relation graph | Topological sort cannot complete; an explicit error is raised before any DDL runs |

## See Also

- [[Database]] — the class hosting these methods.
- [[MetadataStorage]] — the data both methods walk.
- [[query-lifecycle]] — the orthogonal flow on the read side.

## Sources

- `.raw/architecture-overview.md` § "Database, Schema, and Lifecycle"
