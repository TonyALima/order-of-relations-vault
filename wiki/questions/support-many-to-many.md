---
type: question
title: "Should OOR implement support for @ManyToMany relations?"
question: "Add a @ManyToMany decorator that emits a join table at schema-create time and exposes the inverse collection on both sides; today neither @OneToMany nor any join-table machinery exists, even though RelationType.TO_MANY is declared in the metadata enum."
answer_quality: open
created: 2026-04-29
updated: 2026-04-29
tags:
  - question
  - open
  - decorators
  - relations
  - schema
status: open
impact: medium
effort: L
decided_by: ""
related:
  - "[[support-one-to-many]]"
  - "[[Repository]]"
  - "[[MetadataStorage]]"
  - "[[Database]]"
  - "[[Relation Target Thunk]]"
  - "[[lifecycle-of-a-create]]"
  - "[[schema-create-drop]]"
  - "[[examples/relations]]"
sources: []
---

# Should OOR implement support for `@ManyToMany` relations?

> [!note] Status: **open** (no decorator; no join-table machinery; depends on [[support-one-to-many]])

## Question

OOR has no way to express a many-to-many relation. The only relation decorator that exists is `@ToOne` (`src/decorators/relation/relation.ts`); the only enum members are `TO_ONE` and `TO_MANY` — neither describes the M:N case explicitly. Should we ship a `@ManyToMany` decorator that (a) introduces a third metadata category, (b) emits a synthesized join table at schema-create time, and (c) handles the bidirectional read/write semantics that come with M:N?

## Why it matters

- **M:N is one of the three canonical relation shapes.** Tag systems, role assignments, course enrollments, friendship graphs — every non-trivial application schema has at least one. Without `@ManyToMany`, OOR consumers fall back to manually declaring the join entity (User, Tag, *and* `UserTag`), all three with explicit `@ToOne` decorators. Workable, verbose, and exposes infrastructure that should stay implicit.
- **It's the natural follow-on to [[support-one-to-many]].** The standard ORM pattern decomposes M:N into two `@OneToMany` halves bridged by a join table. Whatever shape that question lands on directly constrains this one — the join entity is "just" two `@ManyToOne`s, and each side of the M:N is "just" a `@OneToMany` over the join entity. Filing this question now keeps the dependency visible while the upstream design is still open.
- **Schema-emission machinery would have to grow a new capability.** Today `Database.createTables()` and `Database.createRelations()` only emit DDL for entities the user has registered. A `@ManyToMany` requires emitting a *synthesized* table that has no user-facing class. That's a meaningful new pattern in the schema layer (and a meaningful question about what `MetadataStorage` should hold).
- **Touches the [[support-user-indexes]] naming policy.** The two FK columns on a join table are textbook composite-index candidates (`(user_id, tag_id)` for the forward direction, `(tag_id, user_id)` for the reverse). Whatever naming convention `@Index` ships with will need to apply to synthesized join tables too.

## Current behavior (so the question doesn't decay)

- **Nothing in the decorator surface mentions M:N.** No `@ManyToMany`, no `JoinTable`, no `JoinColumn` (verified `src/decorators/`, 2026-04-29).
- **The metadata model has no concept of a join table.** `RelationMetadata.columns` is a flat list of FK columns on the *local* row. There is no `joinTable` field, no `inverseColumns` field, no synthesized-entity machinery.
- **`MetadataStorage` is keyed by user-supplied `Constructor`** (`metadata.ts:42`). A synthesized join table has no constructor — so either the storage shape generalizes to `Constructor | symbol`, or join tables live in a sibling structure.
- **`Database.createTables()` iterates `this.metadata`** (`database.ts:`*early loop*) — it would never see a join table that wasn't a registered entity. Either the loop changes, or `createTables` gains a second pass over synthesized entities.
- **The workaround today is fully manual.** Users declare a `UserTag` class with two `@ToOne` relations and FK-as-PK columns, and walk both sides via `Repository<UserTag>.findMany`. It works but exposes the join entity in application code.

## Sketch of the design space

Six axes the design has to decide on. None is endorsed here.

### Axis 1 — Synthesized vs explicit join entity

- **Fully synthesized.** `@ManyToMany(() => Tag) tags: Tag[]` on `User` (and the inverse on `Tag`); OOR generates a `user_tags` table behind the scenes. No user class for the join. Cleanest API; biggest divergence from the "everything is a registered entity" invariant the rest of `MetadataStorage` assumes.
- **Explicit join entity** (TypeORM-style required `@JoinTable`). User declares `UserTag` themselves with two `@ManyToOne`s; OOR's `@ManyToMany` is sugar that walks the user-declared join entity. Smaller new surface in the metadata layer; pushes the join entity back into application code.
- **Synthesized by default with escape hatch.** Synthesize unless the user passes `{ through: () => UserTag }`. Most flexible; biggest API.

### Axis 2 — Owning side vs inverse side

- **Required owning side** (TypeORM). One side carries `@JoinTable()`; the other carries `@ManyToMany(..., inverseSide)`. The owning side controls the join table's name, columns, indexes.
- **Symmetric, no owning side.** Both sides declare `@ManyToMany`; OOR canonicalizes the join table's name (e.g., alphabetical) and column order. Cleaner mental model for users; harder to give per-side configuration knobs (column names, cascade flags) when there's no asymmetry.
- **Single-side declaration.** Only the owning side declares the M:N at all; the inverse side has no decorator. Smallest API; loses the type-level back-pointer on the inverse class.

### Axis 3 — Join table naming

- **Auto** — `<table_a>_<table_b>` with sides sorted alphabetically (or by declaration order on the owning side). Predictable; no user choice.
- **User-supplied** — `@ManyToMany({ joinTable: "user_tags" })`. Full control; risks collisions if users pick names carelessly.
- **Auto with override.** Generate by default, allow `joinTable` override. Matches what other ORMs do.

### Axis 4 — Loading strategy

- Inherits from [[support-one-to-many]] Axis 3. The same lazy-vs-eager-vs-opt-in choice applies, but the eager case requires a *two-hop* JOIN (`user → user_tags → tags`) instead of a single-hop one. Whatever pattern `@OneToMany` settles on, `@ManyToMany` extends — so the eager-loading code path probably wants to be expressed in terms of relation-walks, not hard-coded JOIN counts.

### Axis 5 — Write semantics

- **Replace-on-assignment.** `user.tags = [tag1, tag2]` followed by `repo.update(user)` does `DELETE FROM user_tags WHERE user_id = ?` then re-inserts. Simple to reason about; surprising performance for large sets.
- **Explicit `addRelation` / `removeRelation` methods.** No magic on assignment; users call `repo.addRelation(user, 'tags', tag)` and `repo.removeRelation(...)`. Smaller API surface in the `update` path; bigger API surface on `Repository`.
- **Both.** Most flexible; biggest API. Closest to what TypeORM ships.

### Axis 6 — Cascade

- **No cascade — error on parent delete with non-empty join rows** (PostgreSQL `RESTRICT`). Safest.
- **Cascade only the join rows on parent delete** (`ON DELETE CASCADE` on the join table FKs). Almost always what users want — deleting a `User` should remove their `user_tags` rows but not their `Tag`s. Likely the right default.
- **Configurable per side.** Matches mature ORMs.

## Things to verify before deciding

- **Prior art.** TypeORM requires `@JoinTable` on the owning side; MikroORM auto-generates by default but allows `pivotTable`; Drizzle treats the join table as a first-class user-declared schema with no special M:N decorator. The right reference point depends on how OOR wants to position itself stylistically — an [[comparisons/_index|orm comparison page]] is overdue and would directly inform this.
- **Whether `MetadataStorage` should hold synthesized entities.** If yes, the iteration contract changes (`Iterable<[Constructor | SyntheticKey, EntityMetadata]>`). If no, join tables live in a sibling map and every consumer of `MetadataStorage` (schema-create, schema-drop, repository, query-builder) needs to know to walk both. Worth deciding before implementing — retrofitting is painful.
- **Interaction with [[Single-Table Inheritance]].** If either side of an M:N is an STI subclass, the join table's FK should reference the root table (since that's the actual physical table). The discriminator predicate then lives on read, in the same place [[support-one-to-many]] adds it.
- **Composite primary key on the join table.** Standard pattern is `PRIMARY KEY (user_id, tag_id)` rather than a synthetic surrogate `id`. OOR's current PK machinery (in [[MetadataStorage]] and `schema-create`) supports composite PKs already — verify it works for synthesized entities too.
- **Index on the *inverse* direction.** A `PRIMARY KEY (user_id, tag_id)` covers `WHERE user_id = ?` queries (left-prefix), but `WHERE tag_id = ?` (the inverse-side load) needs its own index. If [[support-user-indexes]] hasn't shipped, the M:N implementation has to either hard-code the second index or wait.

## What would change in the codebase

Approximate change surface for synthesized join, owning-side `@JoinTable`, lazy loading, cascade-on-parent-delete:

- **New `src/decorators/relation/many-to-many.ts`** and **`src/decorators/relation/join-table.ts`** — two paired decorators, similar to TypeORM's pattern. Owning side: `@ManyToMany(() => Tag) @JoinTable() tags: Tag[]`. Inverse side: `@ManyToMany(() => User, user => user.tags) users: User[]`.
- **`src/core/metadata/metadata.ts`** — extend `RelationType` with a `MANY_TO_MANY` member (or model M:N as a *pair* of synthesized `TO_MANY` entries pointing through a generated entity). Add `joinTableName` and `inverseColumns` to `RelationMetadata`, or introduce a new `JoinTableMetadata` type and store it separately on `EntityMetadata`.
- **`MetadataStorage`** — decide whether synthesized entities live in `storage` keyed by a sentinel `Symbol`, or in a sibling `joinTables: Map<string, EntityMetadata>` field. Update the iteration contract accordingly.
- **`src/core/database/database.ts`** — `createTables` walks both real and synthesized entities. `createRelations` emits FK constraints for join-table columns. `dropTables` drops synthesized tables before parent tables (existing topological sort should already handle this if join tables register their deps correctly).
- **`src/core/repository/repository.ts`** — write path: handle replace-on-assignment (or expose explicit add/remove methods, depending on Axis 5). Read path: load the collection by joining through the synthesized table.
- **`src/core/query-builder/`** — extend the relation-walking logic from [[support-one-to-many]] to two-hop (`user → user_tags → tags`).
- **Tests** — owning + inverse declaration, schema-create round trip (verify `\d user_tags` shows the synthesized table), set replacement, cascade behavior, M:N where one side is an STI subclass.
- **Docs** — new component pages (`wiki/components/ManyToMany.md`, `wiki/components/JoinTable.md`); extend [[examples/relations]] with an M:N example; update [[brief]].

## What would close this question

A decision on each axis above plus a shipped implementation. By this vault's convention (`open → answered`), this question stays open until both the design choice is locked and the code is in main. Almost certainly produces its own ADR (`0009-many-to-many` or similar), and the ADR for [[support-one-to-many]] should land first since this one builds on it.

## Confidence

**Open** — no decision yet, no code yet, and at least one upstream dependency ([[support-one-to-many]]) is also still open. Filed because (a) M:N is the third leg of the relations stool and OOR is *named* Order of Relations, (b) the synthesized-entity question is structurally large enough that surfacing it now is cheaper than discovering it mid-implementation, and (c) it has cross-cutting interactions with at least three other open design questions (one-to-many, user indexes, STI on either side of an M:N).

## Related Questions

- [[support-one-to-many]] — must land first; M:N is conventionally implemented as two O:M halves through a synthesized join entity.
- [[support-user-indexes]] — the inverse-direction index on the join table is the canonical use case for a user-declared composite index; the naming policy chosen there should cover synthesized tables too.

## Sources

- `../order-of-relations/src/core/metadata/metadata.ts:15-18` — `RelationType` enum (no `MANY_TO_MANY` member).
- `../order-of-relations/src/decorators/relation/relation.ts` — only `ToOne` exists today.
- `../order-of-relations/src/core/database/database.ts` — `createTables` / `createRelations` only walk registered entities; no synthesized-entity pattern.
- [[support-one-to-many]] — direct upstream design dependency.
