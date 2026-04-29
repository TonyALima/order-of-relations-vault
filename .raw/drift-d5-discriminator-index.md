# Drift Correction — schema-create emits an implicit `idx_discriminator` index

`wiki/components/MetadataStorage.md` says, of `EntityMetadata`: *"Schema names, indexes, cascading rules: deliberately absent. They'd belong on the same record but aren't needed yet."* The framing — that user-declarable indexes are deliberately not modelled — is correct and should stay. But the schema-create path **does** emit one specific index implicitly, and the wiki doesn't mention it anywhere.

This is a small drift, but it bites in two ways: (1) anyone reading the wiki and then `EXPLAIN`-ing a query will see an index they weren't told about; (2) anyone trying to hand-write the equivalent DDL won't know to add it.

## What the code actually does

`src/core/database/database.ts`, inside `createBaseTables()`:

```ts
const hasDiscriminator = metadata.discriminator !== undefined;

if (hasDiscriminator) {
  await sql`
    ALTER TABLE ${sql(metadata.tableName)}
    ADD COLUMN discriminator TEXT NOT NULL;
    CREATE INDEX idx_discriminator ON ${sql(metadata.tableName)}(discriminator);
  `.simple();
}
```

So the actual schema emitted for a single-table-inheritance hierarchy is, in order:

1. `CREATE TABLE <tableName> (<columns>, PRIMARY KEY (...))`
2. `ALTER TABLE <tableName> ADD COLUMN discriminator TEXT NOT NULL;`
3. `CREATE INDEX idx_discriminator ON <tableName>(discriminator);`

Two facts worth flagging in the wiki:

- The `idx_discriminator` name is **shared across every STI table**. If two STI hierarchies ever exist in the same schema, both `CREATE INDEX idx_discriminator` statements will collide on the second `db.create()`. Today that's invisible because the examples have one hierarchy; it's a latent bug worth flagging.
- The index is added only when `metadata.discriminator !== undefined` — and recall (`[[Single-Table Inheritance]]`) that the discriminator gets **wiped** by `resolveInheritance` if a class has no siblings. So a single-class table gets neither the discriminator column nor the index. As soon as a sibling registers, the next `db.create()` will add both.

## Other implicit-but-not-user-controllable schema bits worth a callout

While we're here, `createBaseTables` has one more behaviour that the wiki under-documents:

```ts
if (metadata.columns.length === 0) continue;
if (metadata.discriminator && metadata.discriminator !== metadata.tableName) continue;
```

The second `continue` is the **STI base-table-only rule**: subclass entities (whose `tableName` was rewritten to the root's during `resolveInheritance`, so their `discriminator` differs from `tableName`) are skipped. Only the root entity emits a `CREATE TABLE`. This is the actual implementation of "STI = one table for the whole hierarchy" at the schema-create layer. The wiki concept page describes the metadata-layer side of STI but doesn't pin this skip-condition.

## Why this matters

- The MetadataStorage page's "indexes deliberately absent" sentence is technically about *user-declarable* indexes. As written, it reads as if no indexes are created anywhere. A reader will be surprised by `idx_discriminator`.
- The `idx_discriminator` choice is a real performance decision: STI tables are cheaper to query *because* the discriminator column is indexed. That's a small but load-bearing fact for anyone reasoning about query plans.
- The latent name-collision risk is exactly the kind of thing an ADR or a "known limitation" callout exists for.

## Wiki pages affected

| Page | Action |
|---|---|
| `wiki/components/MetadataStorage.md` | After the "Schema names, indexes, cascading rules: deliberately absent" sentence, add: *"There is one exception, owned by the schema-create path rather than this storage: STI tables get an implicit `idx_discriminator` index added by `Database.create()`. See `[[schema-create-drop]]` and `[[Single-Table Inheritance]]`."* |
| `wiki/flows/schema-create-drop.md` | Add a step (or sub-step under `CREATE TABLE`) describing the `ALTER TABLE … ADD COLUMN discriminator` and the `CREATE INDEX idx_discriminator` that follow when `metadata.discriminator !== undefined`. Also document the *"`columns.length === 0` skip"* and the *"subclass entities are skipped (their `tableName` matches the root)"* rules — they're load-bearing for understanding which entities produce DDL. |
| `wiki/concepts/Single-Table Inheritance.md` | After "How It Works" step 4, add a "What schema-create emits" subsection with the three statements above (CREATE TABLE for the root, ALTER ADD COLUMN, CREATE INDEX). Cross-link to `[[schema-create-drop]]`. |
| `wiki/components/Database.md` | The "Schema lifecycle" section currently summarizes `create()` as "1. Emit `CREATE TABLE` … 2. Emit `ALTER TABLE … ADD FOREIGN KEY`." Insert step 1.5: *"For STI hierarchies, ALTER the root table to add the `discriminator TEXT NOT NULL` column and a `CREATE INDEX idx_discriminator` on it."* |
| (Optional, future) `wiki/questions/idx-discriminator-collision.md` | Open question: should `idx_discriminator` be namespaced per-table (e.g. `idx_<tableName>_discriminator`) to avoid the cross-hierarchy collision risk? |

## Source citations

- `src/core/database/database.ts` lines 33–113 — `createBaseTables`, including the early-`continue`s and the `ALTER TABLE` + `CREATE INDEX` block.
- `src/core/metadata/metadata.ts` lines 58–82 — `resolveInheritance` and the discriminator wipe rule that determines whether the index gets emitted.
- `wiki/components/MetadataStorage.md` line ~47 — the "indexes deliberately absent" sentence that needs softening.
