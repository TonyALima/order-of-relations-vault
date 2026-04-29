---
type: question
title: "Should OOR implement support for @OneToMany relations?"
question: "Add a @OneToMany decorator (and likely a paired @ManyToOne, since the FK lives on the inverse side) so users can model collections like User → Posts; today only @ToOne exists, even though RelationType.TO_MANY is already declared in the metadata enum."
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
impact: high
effort: L
decided_by: ""
related:
  - "[[Repository]]"
  - "[[MetadataStorage]]"
  - "[[Database]]"
  - "[[Relation Target Thunk]]"
  - "[[Repository Pattern]]"
  - "[[lifecycle-of-a-create]]"
  - "[[entity-registration]]"
  - "[[schema-create-drop]]"
  - "[[examples/relations]]"
sources: []
---

# Should OOR implement support for `@OneToMany` relations?

> [!note] Status: **open** (no decorator; metadata enum already reserves the slot)

## Question

OOR ships exactly one relation decorator: `@ToOne`. The `RelationType` enum already declares both members — `TO_ONE = 'to-one'` and `TO_MANY = 'to-many'` (`src/core/metadata/metadata.ts:15-18`) — but `TO_MANY` is never constructed anywhere in the codebase (verified by grep, 2026-04-29). Should we ship a real `@OneToMany` decorator, plus the paired `@ManyToOne` (since on a one-to-many the FK lives on the *many* side, not the *one* side)?

## Why it matters

- **Without to-many, OOR can't model the most common shape in real schemas.** "User has many Posts," "Order has many LineItems," "Author has many Books." Today the only way to walk those is to expose the inverse `@ToOne` and query manually with a `WHERE author_id = ...`. That's workable for the codebase author, not for the npm-package audience the [[brief]] targets.
- **The metadata layer already half-anticipates it.** `RelationType.TO_MANY` exists; `RelationMetadata` has a `relationType` discriminator. Implementing `@OneToMany` is partly a matter of *finishing* the work the data model already implies — and partly a matter of fixing the two code paths that currently *assume* every relation has FK columns on the local row.
- **Foundation for `@ManyToMany`.** The standard ORM pattern decomposes many-to-many into two `@OneToMany` halves over a join table. Whatever resolution this question takes will constrain the design space for [[support-many-to-many]] — they should be considered together, not in isolation.
- **Currently load-bearing in the docs.** [[examples/relations]] is already filed as a stub front-door. Without `@OneToMany` it can only ever cover the to-one half of "relations" — a meaningful gap for an ORM literally named *Order of Relations*.

## Current behavior (so the question doesn't decay)

- **Only `@ToOne` exists.** `src/decorators/relation/relation.ts` exports exactly `OneToOneOptions` and `ToOne` — no `OneToMany`, `ManyToOne`, or `ManyToMany` (verified 2026-04-29).
- **`RelationMetadata.relationType` is hard-coded to `TO_ONE`.** `relation.ts:30` writes `relationType: RelationType.TO_ONE`. Nothing else in `src/` constructs a `RelationMetadata` value, so `TO_MANY` is dead enum-land.
- **`MetadataStorage.resolveRelations()` auto-generates FK columns for *every* relation** (`metadata.ts:84-104`). It assumes the FK lives on the local table: it pulls the target's primary columns, demotes their types via `toForeignKeyType`, and writes them as `${propertyName}_${pk.propertyName}` columns on the local row. For a `@OneToMany`, this is the wrong table — the FK belongs on the *target* side.
- **`Repository.create` and `Repository.update` both iterate `meta.relations` blindly** and write into `relation.columns!` (`repository.ts:111-120` and `:173-182`). For a `@OneToMany` there are no local FK columns to write — those passes need to filter by `relationType` (or skip when `columns` is null/empty).
- **`Database.createRelations()` issues `ALTER TABLE … ADD FOREIGN KEY` against the *local* table** (`database.ts:122-150`). For `@OneToMany`, the FK constraint should be emitted on the *inverse* (`@ManyToOne`) side instead — i.e., the same statement, but originating from a different `EntityMetadata`.
- **No fetch-time loading exists.** `Repository.findMany` / `findOne` don't currently follow relations at all (no `JOIN`, no second query). So the "lazy vs eager loading" question for to-many doesn't have an existing pattern to inherit — it has to be invented.

## Sketch of the design space

Five axes the design has to decide on. None is endorsed here.

### Axis 1 — Decorator shape

- **Paired `@OneToMany` + `@ManyToOne`** (TypeORM, MikroORM). Owner side declares `@ManyToOne(() => User) author: User` and writes the FK; inverse side declares `@OneToMany(() => Post, post => post.author) posts: Post[]` and is FK-free. Most explicit, two new decorator folders.
- **Inverse-only `@OneToMany` with auto-inferred `@ManyToOne`** — declare the collection on the parent; OOR infers the missing `@ManyToOne` on the child by reading the back-pointer thunk. Smaller surface, but requires running a *resolution pass* over both sides at metadata-resolve time.
- **No `@OneToMany`; require users to declare the inverse `@ToOne` only.** Skips the decorator entirely. Fast to ship, but leaves the parent class unable to type its `posts: Post[]` field as a relation — defeats much of the point.

### Axis 2 — How to express the back-pointer

- **Lambda referencing the inverse property** (`@OneToMany(() => Post, post => post.author)`). Type-safe, refactor-friendly. Same shape used by [[Relation Target Thunk]] for the forward thunk.
- **String name of the inverse property** (`@OneToMany({ target: () => Post, inverseSide: "author" })`). Simpler to parse; loses type safety on the inverse name.
- **Inferred from a single shared symbol** (e.g., both decorators reference the same `relationKey`). Most clever; least readable.

### Axis 3 — Loading strategy

- **Lazy — separate query on access.** A `Post[]` field is initially undefined; reading it triggers `findMany({ where: { author: parent.id } })`. Composes naturally with [[Lazy Query Builder]]. Risks N+1 by default.
- **Eager via opt-in `relations: ["posts"]` in `FindOptions`.** TypeORM-style. Adds a third top-level field to `FindOptions<T>` (alongside `where` and `inheritance`).
- **Always eager.** Simplest semantics, worst performance. Almost certainly wrong, but worth listing for completeness.

### Axis 4 — JOIN vs second query when loading eagerly

- **Single `LEFT JOIN` with row-set inflation in TypeScript** — one round-trip; duplicate parent columns in the result; needs a deduplication pass after the fact.
- **Two queries: parent IDs first, then `WHERE child.fk IN (...)`** — two round-trips; cleaner mapping; better with large parent rows. The pattern Drizzle's relational query builder uses.

### Axis 5 — Cascade on delete

- **No cascade — error if the parent has children** (PostgreSQL default `RESTRICT`). Safest; pushes the choice to the user.
- **`ON DELETE CASCADE` configurable per relation.** Matches what mature ORMs ship.
- **Always cascade.** Surprising; almost certainly wrong.

## Things to verify before deciding

- **Prior art.** TypeORM ships paired `@OneToMany` / `@ManyToOne` with required `inverseSide` thunk; MikroORM the same; Drizzle uses a relational-builder DSL outside the decorator system entirely. Worth a [[comparisons/_index|comparison]] page before committing — the decision interacts with how OOR positions itself stylistically.
- **Type-level shape of the collection field.** Today `@ToOne` types the field as `TType | undefined` (the decorator's second generic). For `@OneToMany` it should be `TType[] | undefined` (lazy) or `TType[]` (always populated). Decide, and make sure the type lines up with how loading is implemented.
- **Interaction with [[Single-Table Inheritance]].** If the inverse side is an STI subclass, the auto-emitted `WHERE fk = ?` should also push a discriminator predicate so we don't pull siblings. The machinery is already there in `applyOptions` — just needs to be wired.
- **Interaction with the FK auto-generation in `resolveRelations`.** That pass currently runs unconditionally; for `@OneToMany` it should skip column generation on the local side and (perhaps) verify the inverse `@ManyToOne` is present. Decide whether absence is an error or an inference trigger.
- **Where `Database.createRelations()` emits the `ALTER ADD FOREIGN KEY`.** The constraint logically belongs to the FK-owning side. The current loop iterates the local entity's relations; for a to-many, the FK statement has to come from the *inverse* entity's relations instead. Either filter by `relationType === TO_ONE` (and let the `@ManyToOne` declaration drive emission) or change the loop to walk every entity once and emit FK constraints only when `columns !== null`.
- **The repository write-path filter.** `Repository.create` and `update` need a `relation.relationType === RelationType.TO_ONE` guard (or equivalently `relation.columns !== null`) before writing FK values. Without it, `relation.columns!` will throw at runtime once `@OneToMany` lands.

## What would change in the codebase

Approximate change surface for paired `@OneToMany` + `@ManyToOne`, lazy loading, no cascade by default:

- **New `src/decorators/relation/many-to-one.ts`** — owner side. Same shape as `ToOne` but the metadata entry uses `relationType: TO_ONE` (the FK-owning side really *is* a to-one in the metadata model — you only ever have one parent). May literally become an alias for `ToOne` with a clearer name.
- **New `src/decorators/relation/one-to-many.ts`** — inverse side. Declares `relationType: TO_MANY`, captures the inverse-property thunk, and writes a `RelationMetadata` entry with `columns: null` permanently (no local FK).
- **`src/core/metadata/metadata.ts`** — extend `RelationMetadata` with `inverseSide?: () => string` (or similar). Update `resolveRelations` to skip FK auto-generation for `TO_MANY` entries, and to validate that each `TO_MANY` has a matching `TO_ONE` on the target side.
- **`src/core/repository/repository.ts`** — guard the two `meta.relations.forEach` blocks at `:111` and `:173` with `relation.relationType === RelationType.TO_ONE`. Otherwise unchanged for v1 (no eager loading on `findOne`/`findMany` yet).
- **`src/core/database/database.ts`** — `createRelations()` skips `TO_MANY` entries (they own no FK); the existing FK-emission logic stays correct for `TO_ONE` / `MANY_TO_ONE`.
- **`src/core/query-builder/`** — net-new method on `QueryBuilder` (or `Repository`) for loading the collection: `repo.loadRelation(parent, 'posts')` or eager opt-in via `FindOptions.relations`. The exact API depends on Axis 3 / Axis 4.
- **Tests** — owner-only round trip (existing `@ToOne` semantics, just renamed); inverse collection load; cascading delete behavior; STI inverse side; the `RelationMetadata.columns === null` invariant for `TO_MANY`.
- **Docs** — new component pages (`wiki/components/OneToMany.md`, `wiki/components/ManyToOne.md`); flesh out the [[examples/relations]] stub as the canonical use site; update [[Relation Target Thunk]] to cover the inverse-property thunk; update [[brief]] under "what OOR supports today."

## What would close this question

A decision on each axis above plus a shipped implementation. By this vault's convention (`open → answered`), this question stays open until both the design choice is locked and the code is in main. Likely outputs: a new ADR (`0008-to-many-relations` or similar), two new decorator components, and a substantial expansion of [[examples/relations]].

## Confidence

**Open** — no decision yet, no code yet. The metadata enum already reserves `TO_MANY`, but every code path that consumes `RelationMetadata` today silently assumes `TO_ONE`. Filing because (a) the gap is the most-asked-about omission for any ORM-shaped library, (b) the design choice constrains [[support-many-to-many]], and (c) the decision is large enough — five axes, two new decorators, three core-module touch points — to deserve its own ADR rather than getting smuggled into another change.

## Related Questions

- [[support-many-to-many]] — depends on the resolution here; the standard pattern decomposes M:N into two O:M halves over a join table.
- [[support-user-indexes]] — FK columns are obvious index candidates; a naming policy that lands there should also cover the FK indexes implied by `@ManyToOne`.

## Sources

- `../order-of-relations/src/core/metadata/metadata.ts:15-18` — `RelationType` enum (declares `TO_MANY`, never constructed).
- `../order-of-relations/src/decorators/relation/relation.ts` — only `ToOne` exists today.
- `../order-of-relations/src/core/repository/repository.ts:111-120, :173-182` — `create` / `update` blindly write FK columns for every relation.
- `../order-of-relations/src/core/database/database.ts:113-150` — `createRelations()` emits `ALTER ADD FOREIGN KEY` on the local table.
- [[entity-registration]], [[lifecycle-of-a-create]], [[schema-create-drop]] — flow context for the touch points above.
