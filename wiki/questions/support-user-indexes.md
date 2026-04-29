---
type: question
title: "Should OOR support user-defined indexes (@Index / @Unique)?"
question: "Add a way for users to declare indexes on columns and column-tuples, so schema-create emits CREATE INDEX alongside CREATE TABLE."
answer_quality: open
created: 2026-04-29
updated: 2026-04-29
tags:
  - question
  - open
  - decorators
  - schema
  - performance
status: open
impact: medium
effort: M
decided_by: ""
related:
  - "[[Repository]]"
  - "[[QueryBuilder]]"
  - "[[MetadataStorage]]"
  - "[[schema-create-drop]]"
  - "[[Single-Table Inheritance]]"
  - "[[decorator-order-independence]]"
sources: []
---

# Should OOR support user-defined indexes (`@Index` / `@Unique`)?

> [!note] Status: **open** (no implementation; no decorator exists today)

## Question

OOR currently has no way for a user to declare an index. The only index ever emitted is the implicit `idx_discriminator` that [[schema-create-drop]] adds for [[Single-Table Inheritance]] roots. Should we add a user-facing `@Index` (and probably `@Unique`) decorator so consumers can mark hot query columns and column-tuples for indexing, with `schema-create` emitting `CREATE INDEX` alongside `CREATE TABLE`?

## Why it matters

- **Real-world ORM table-stakes.** Any non-trivial application running against PostgreSQL hits a wall the moment a `WHERE` filters on an unindexed column over a large table. Without `@Index`, the only escape hatches are (a) hand-written migration SQL outside OOR, or (b) accepting full-scan cost. Both undercut the value of using OOR end-to-end.
- **The codebase already proves the schema-emission pattern works.** `schema-create` already emits `ALTER ADD COLUMN discriminator` + `CREATE INDEX idx_discriminator` for STI roots ([[schema-create-drop]] §§ "STI roots", [[sources/drift-d5-discriminator-index]]). User-defined indexes would reuse the same emission pass — the wiring exists; only the metadata source is missing.
- **Feeds the decorator-order question.** [[decorator-order-independence]] already flags `@Index` and `@Unique` as expected future siblings of `@Column` / `@Nullable`. The chosen resolution there (Option A — defer joins to `@Entity`) was selected partly because it scales to N coordinating decorators. Filing this question makes that "N" concrete.
- **Composes with the `idx_discriminator` collision risk.** [[hot]] notes a latent question about `idx_discriminator` colliding across STI roots in the same database. Once user-defined indexes exist, name collision becomes a general concern, not an STI-only one — both should probably share a naming policy.

## Current behavior (so the question doesn't decay)

- **No `@Index` / `@Unique` decorator exists.** `src/decorators/` contains only `column/`, `entity/`, `nullable/`, `relation/` (verified 2026-04-29).
- **No index metadata is stored.** [[MetadataStorage]] holds `EntityMetadata` keyed by class; the entry's column list (`columns: ColumnMetadata[]`) has no `index` / `unique` fields. Primary key uniqueness is enforced via `PRIMARY KEY` in DDL, not via a dedicated `@Unique`.
- **The single index OOR emits is the discriminator index.** Hard-coded in the schema-create path; no user input.
- **Workarounds today** are out-of-band: write a `.sql` migration after `schema-create` runs, or extend the database manually.

## Sketch of the design space

Three axes the design has to decide on. None is endorsed here.

### Axis 1 — Where the decorator goes

- **Property-level only** (`@Index column: string;`) — simplest. Forces composite indexes onto the class.
- **Class-level only** (`@Entity({ indexes: [{ columns: ["lastName", "firstName"] }] })`) — uniform; all index info in one place. Less ergonomic for the common single-column case.
- **Both** (`@Index` on properties for single-column, class-level options for composite) — most ergonomic; biggest API surface. TypeORM took this path.

### Axis 2 — `@Unique` as a separate decorator vs `@Index({ unique: true })`

- **Separate `@Unique`** — clearer at call site; mirrors `@Nullable` / `@NotNullable` pattern OOR already uses. Two decorator folders to maintain.
- **Flag on `@Index`** — fewer decorators; the `unique: true` flag reads slightly less clearly than a dedicated decorator. One folder.

### Axis 3 — Naming policy

- **Auto-generated** (`idx_<table>_<col1>_<col2>`) — predictable, no user choice. Composes naturally with a fix to the `idx_discriminator` collision (rename to `idx_<root>_discriminator`).
- **User-supplied** (`@Index({ name: "..." })`) — full control; risks collisions if users pick names carelessly.
- **Auto with override** — generate by default, allow `name` override. Most flexible; matches the pattern other ORMs use.

## Things to verify before deciding

- **Prior art.** TypeORM has `@Index` (property + class), `@Unique` (class-level only). Drizzle has table-level `index()` / `uniqueIndex()`. MikroORM has `@Index` / `@Unique` at both levels. Worth a comparison page before committing to a shape.
- **Partial / functional / GIN-and-friends indexes.** `CREATE INDEX ... WHERE deleted_at IS NULL` and `CREATE INDEX ... USING GIN (col)` are real PostgreSQL features OOR consumers will eventually want. The first one tangles with [[Parameterized SQL]] (the `WHERE` body is a literal SQL string, not a parameterized expression). Decide whether v1 supports any of these or scopes to plain B-tree only.
- **Drop semantics.** [[schema-create-drop]] does topologically reversed `DROP TABLE`. `DROP TABLE` cascades indexes automatically, so `schema-drop` may need no change — but verify, and decide whether to emit explicit `DROP INDEX` for clarity.
- **Interaction with the [[decorator-order-independence]] resolution.** If that question closes with Option A (defer joins to `@Entity`), `@Index` should follow the same pattern from day one — push raw `IndexMetadata` entries onto an `INDEXES_KEY`, let `@Entity` do the join. Don't ship `@Index` with the older "read sibling at decoration time" pattern only to refactor it later.
- **`idx_discriminator` collision fix should land at the same time.** Currently `idx_discriminator` is the same name across all STI roots. Once user indexes exist, every name is collision-prone. A naming policy (`idx_<table>_<cols>`) should apply uniformly to discriminator indexes and user indexes both.

## What would change in the codebase

Approximate change surface for a property-level `@Index` + class-level composite + auto-generated names:

- **New `src/decorators/index/index.ts`** — declares `INDEXES_KEY = Symbol('indexes')`. `@Index` (and probably `@Unique` in a sibling folder) writes `IndexMetadata` entries: `{ columns: string[], unique: boolean, name?: string }`.
- **`src/decorators/entity/entity.ts`** — read `INDEXES_KEY`, copy into `EntityMetadata.indexes: IndexMetadata[]`. If [[decorator-order-independence]] Option A is chosen, this is the same join pass.
- **`src/core/sql-types/`** (or wherever DDL generation lives) — extend the schema-create emission pass: after `CREATE TABLE`, emit one `CREATE [UNIQUE] INDEX <name> ON <table> (<cols>)` per `IndexMetadata` entry. Generate `<name>` as `idx_<table>_<col1>_<col2>` (or `uniq_<table>_<...>`) when not user-supplied.
- **STI discriminator index** — refactor from hard-coded `idx_discriminator` to use the same generator: `idx_<root>_discriminator`. Closes the latent collision question in one stroke.
- **[[MetadataStorage]]** — extend `EntityMetadata` with `indexes: IndexMetadata[]`.
- **Tests** — single-column index, composite, unique-vs-non-unique, name override, integration test asserting `\d <table>` shows the indexes after schema-create.
- **Docs** — new component page `wiki/components/Index.md`, update [[schema-create-drop]] to mention user indexes alongside the discriminator index, update [[brief]] under "what OOR supports today."

## What would close this question

A decision on each axis above plus a shipped implementation. By this vault's convention (`open → answered`), this question stays open until both the design choice is locked and the code is in main. Likely outputs: a new ADR (`0008-user-defined-indexes` or similar) and one or two new decorator components.

## Confidence

**Open** — no decision yet, no code yet. Filed because the gap is real, the schema-emission machinery is already in place, and the design choice interacts with at least two other open questions ([[decorator-order-independence]], the `idx_discriminator` collision).

## Related Questions

- [[decorator-order-independence]] — chosen resolution should constrain how `@Index` reads sibling metadata.
- *(unfiled, from D5)* `idx_discriminator` collision — should fold into the naming policy this question settles.

## Sources

- `../order-of-relations/src/decorators/` (verified 2026-04-29 — no `index/` or `unique/` folder).
- [[schema-create-drop]] — current emission pattern for the implicit discriminator index.
- [[sources/drift-d5-discriminator-index]] — documentation of the existing implicit index.
