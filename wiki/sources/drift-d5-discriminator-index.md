---
type: source
title: "Drift D5 — schema-create emits idx_discriminator"
source_type: drift-correction
author: "Tony Albert"
date_published: 2026-04-29
url: ""
confidence: high
key_claims:
  - "Database.createBaseTables emits ALTER TABLE ... ADD COLUMN discriminator + CREATE INDEX idx_discriminator on STI tables."
  - "The idx_discriminator name is shared across every STI table — latent collision risk if two STI hierarchies coexist."
  - "The discriminator column + index are added only when meta.discriminator is truthy (resolveInheritance wipes it for singletons)."
  - "createBaseTables also has an STI base-table-only rule: subclasses (whose tableName matches root, but discriminator differs) are skipped."
  - "MetadataStorage.md said 'indexes deliberately absent' — true for user-declarable indexes, but misleading without the schema-create exception."
created: 2026-04-29
updated: 2026-04-29
tags:
  - source
  - drift
status: stable
related:
  - "[[MetadataStorage]]"
  - "[[Database]]"
  - "[[schema-create-drop]]"
  - "[[Single-Table Inheritance]]"
sources:
  - "../.raw/drift-d5-discriminator-index.md"
---

# Drift D5 — schema-create emits idx_discriminator

## Summary

[[MetadataStorage]] said *"Schema names, indexes, cascading rules: deliberately absent."* That framing is correct for *user-declarable* indexes but read as if no indexes are ever created. In fact, `Database.createBaseTables()` emits one specific index implicitly: `idx_discriminator` on every STI root table.

## What the Code Actually Does

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

So a single-table-inheritance root emits, in order:

1. `CREATE TABLE <tableName> (<columns>, PRIMARY KEY (...))`
2. `ALTER TABLE <tableName> ADD COLUMN discriminator TEXT NOT NULL;`
3. `CREATE INDEX idx_discriminator ON <tableName>(discriminator);`

## Two Footnotes

- **Index name collision risk.** `idx_discriminator` is **shared across every STI table** in a schema. Two STI hierarchies in the same schema → second `db.create()` collides. Today invisible (one hierarchy in the examples); latent.
- **Discriminator-only-when-needed.** The index is added only when `metadata.discriminator !== undefined`. [[Single-Table Inheritance]] notes that `resolveInheritance` wipes the discriminator on singletons. So a single-class table gets neither the column nor the index — until a sibling registers.

## STI Base-Table-Only Rule

`createBaseTables` has a second under-documented rule:

```ts
if (metadata.columns.length === 0) continue;
if (metadata.discriminator && metadata.discriminator !== metadata.tableName) continue;
```

The second `continue` is the **STI base-table-only rule**: subclass entities (whose `tableName` was rewritten to the root's during `resolveInheritance`, so `discriminator` differs from `tableName`) are skipped. Only the root entity emits a `CREATE TABLE`. This is "STI = one table for the whole hierarchy" at the schema-create layer.

## Why the Drift Mattered

- The "indexes deliberately absent" sentence was technically about user-declarable indexes; readers were surprised by `idx_discriminator`.
- The `idx_discriminator` choice is a real performance decision: STI tables are cheaper to query *because* the discriminator is indexed.
- The latent name-collision is exactly the kind of thing a "known limitation" callout exists for.

## Resolution

- [[MetadataStorage]] — softened the "indexes deliberately absent" sentence with the schema-create exception.
- [[schema-create-drop]] — added the ALTER + CREATE INDEX step plus the two skip rules.
- [[Single-Table Inheritance]] — "What schema-create emits" subsection.
- [[Database]] — Schema-lifecycle expanded with step 1.5 for STI.
- Optional future: [[idx-discriminator-collision]] open question.

## Source Citations

- `src/core/database/database.ts` lines 33–113 — createBaseTables, including early-continues and ALTER + INDEX block.
- `src/core/metadata/metadata.ts` lines 58–82 — resolveInheritance and discriminator-wipe rule.
