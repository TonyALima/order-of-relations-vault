---
type: meta
title: "Hot Cache"
updated: 2026-04-30T00:00:00
created: 2026-04-29
tags:
  - meta
  - cache
status: evergreen
related: []
sources: []
---

# Recent Context

## Last Updated

2026-04-30 (later in day). **Major ingest:** `.raw/pk-aware-repository-methods.md` — a post-implementation design memo finishing the compile-time PK-enforcement direction for `findById` / `delete` / `update` / `create`. Three pages created (one source, one concept, one ADR). Five pages updated. Wiki page count: 58 → 61. ADR count: 7 → 8 (first post-rollout ADR).

## Latest ingest (2026-04-30) — `PrimaryKey<V>` brand work

**What shipped (per source — every snippet matches code):**

- New brand: `PrimaryKey<V> = V & { readonly [__pkBrand]: true }` (intersection type, runtime-erased) plus `Unbrand<V>` to strip it.
- `@PrimaryColumn`'s two overloads now require the brand on the field type:
  - With autogen → `NullableField<V> & NullablePrimaryKey<V>` (field must be `?` AND branded).
  - Without autogen → `NotNullableField<V> & PrimaryKey<V>` (field must be `!` AND branded).
- Repository signatures changed:
  - `findById(key: PKInput<T>): Promise<T | null>`
  - `delete(key: PKInput<T>): Promise<void>`
  - `update(entity: UnbrandedT<T> & PKInput<T>): Promise<void>`
  - `create(entity: UnbrandedT<T>): Promise<PKOutput<T>>` (return changed from `Partial<T>` to `PKOutput<T>`, branded)
  - `findOne` / `findMany` UNCHANGED (they take `FindOptions<T>`, not raw entity shapes)
- `Conditions<T>` switched to `FieldConditionBuilder<Unbrand<T[K]>>` so `c.id?.eq(1)` accepts a plain literal.
- `requirePrimaryKey`'s parameter widened from `Partial<T>` to `PKInput<T> | UnbrandedT<T>` (the new strict shapes aren't assignable to `Partial<T>` because of brand asymmetry).

**Brand asymmetry (the load-bearing ergonomic trick):**
`PrimaryKey<V>` is a SUBTYPE of `V`. So branded → unbranded works freely (intersection narrows to its left operand); unbranded → branded requires a cast. Outputs stay branded; inputs accept unbranded values via `Unbrand<V>`. Round-trip `repo.update(await repo.findById({ id: 1 }))` works without casts. Brand only shows up at the **declaration** site.

**Why option A (brand) over B (generic `Repository<T, PK>`) or C (runtime-only):**

1. Consistency with the existing compile-time direction (`create-required-fields` set the precedent).
2. Option B's footgun: `Repository<User, 'name'>` would typecheck silently and disagree with `@PrimaryColumn` metadata at runtime. Unacceptable for a publishable library — declaration-site enforcement closes that gap.
3. Option C closes the silent-bug case but leaves `findById({})` compiling. Step backward from the existing direction.

**Closes:** the silent `update({ name: 'x' })` bug on autogen entities. Autogen PKs are declared `id?: PrimaryKey<number>` (the `?` is load-bearing for `create()`'s "omittable" semantics); previously `update(entity: T)` accepted `{ name: 'x' }` because `T['id']` was optional. Now `PKInput<T>` requires every PK key non-undefined regardless of the optional modifier on `T`.

## Convention reinforced (2026-04-30)

Open questions are for **genuine design decisions**, not for "is feature X working." If functionality is technically supported but might have bugs, that's a code-test concern, not a wiki issue. The retraction of `support-date-operators` is the canonical example: dates already flow through `FieldConditionBuilder<Date>` via existing comparison operators; no design choice is pending.

## Comparison convention (established 2026-04-30, still in scope)

Every page in `wiki/comparisons/` follows this skeleton:

1. Frontmatter `verdict:` — one-line contribution claim.
2. `## What OOR brings that's new` — 3–5 bullet contribution summary, top of page.
3. `## Overview` — what question this comparison answers about OOR's place in the field.
4. `## Comparison` — dimensions table (~10 rows). Equal-value rule applies.
5. `## OOR's contribution, dimension by dimension` — long-form prose, positive framing.
6. `## Why OOR matters in a crowded market` — closing argument.
7. `## Sources`.

**Equal-value rule:** features with a fully-scoped open-question page (axes spelled out, decorator surface defined, change-surface described) are part of OOR's contribution. The new `support-and-or-conditions` does **not yet** clear the equal-value bar for the comparison pages — scoped enough to be tracked, not yet scoped enough to claim. Promote when an ADR closes it.

**Note:** the PK brand work just shipped (2026-04-30) and may be worth surfacing on the comparison pages — "compile-time PK enforcement on `findById`/`update`/`delete` via a runtime-erased brand" is a distinctive contribution. None of TypeORM / Drizzle / Prisma do exactly this. Defer until a comparison-update pass; keep this note as a reminder.

## Comparison contribution headlines (TCC-defense form)

- **vs TypeORM** — "TypeORM-shaped ergonomics with modern guarantees."
- **vs Drizzle** — "The OO ergonomic profile, modernized."
- **vs Prisma** — "TypeScript itself as the schema."
- **Stage-3 vs legacy** — "Right place, right time on the dialect transition."

## Key facts (current state, post-2026-04-30 PK work)

- **Repository surface = exactly 6 methods.** `findMany`, `findOne`, `findById`, `create`, `delete`, `update`. There is no `find()`.
- **`findById`/`delete` accept `PKInput<T>`** (every PK key, all required, all non-undefined, **unbranded**). Not `Partial<T>` anymore.
- **`update` accepts `UnbrandedT<T> & PKInput<T>`** — full entity shape, plus PK fields required regardless of `T`'s optional modifier.
- **`create` returns `PKOutput<T>`** (PK fields only, **branded**) — not `Partial<T>`.
- **`FindOptions<T>` carries TWO fields**: `where` (replaces conditions) and `inheritance` (pushes a discriminator predicate). Untouched by the PK brand work.
- **`Conditions<T>` uses `Unbrand<T[K]>` per field.** `c.id?.eq(1)` accepts a plain literal.
- **`FieldConditionBuilder<V>` exposes 9 methods, uniform across `V`.**
- **`@PrimaryColumn` enforces the brand at the declaration site** via the constraint-flip pattern (second use of the pattern after `@Nullable` / `@NotNullable`).
- **STI schema-create emits THREE statements per root**: `CREATE TABLE`, `ALTER ADD COLUMN discriminator`, `CREATE INDEX idx_discriminator`.
- **`COLUMN_TYPE` is a closed enum (~50 PG types)**; `toForeignKeyType` demotes SERIAL/SMALLSERIAL/BIGSERIAL → INTEGER/SMALLINT/BIGINT for FK columns.

## Drift outcomes (still in scope)

- **D1 — `Repository.find()` does not exist.** Only `findOne` / `findMany` / `findById`.
- **D3 — `FindOptions.inheritance` is real.** `InheritanceSearchType` (`ALL` / `ONLY` / `SUBCLASSES`).
- **D4 — `FieldConditionBuilder` operator inventory.** Nine methods, uniform across `V`.
- **D5 — implicit `idx_discriminator` index.** Three statements per STI root.

## Open Questions (7 — tracked as issues — see [[issues|Issue tracker]])

- 🔓 [[support-one-to-many]] — *high / L* — implement `@OneToMany` / `@ManyToOne` (the `TO_MANY` enum member is currently dead).
- 🔓 [[support-and-or-conditions]] — *high / L* — boolean tree in `where` (AND / OR / NOT, nested groups).
- 🔓 [[support-many-to-many]] — *medium / L* — implement `@ManyToMany` with a synthesized join table.
- 🔓 [[decorator-order-independence]] — *medium / S* — order-independent `@Column` / `@Nullable`.
- 🔓 [[support-user-indexes]] — *medium / M* — `@Index` / `@Unique` + `CREATE INDEX` emission.
- 🔓 [[get-one-limit-1]] — *low / S* — `getOne()` slicing vs `LIMIT 1`?
- 🔓 [[apply-options-accumulation]] — *low / S* — `applyOptions()` replace vs accumulate?

## Active threads

- **Comparison pages** — should be revisited to surface the PK brand work as a distinctive contribution row in [[orms-summary]].
- **Drift backlog:** `src/core/orm-error/`, `src/decorators/nullable/`, `src/decorators/relation/` still uncovered.
- **`examples/relations/` page is a stub** — files weren't opened in the drift pass; expand once read.
- **Migration footprint of the PK brand work** — every example fixture and entity in tests needs `PrimaryKey<V>` on PK fields. If any haven't been migrated yet, they will fail to typecheck against the new `@PrimaryColumn` overloads. Worth a code spot-check.
- **Suggested next action:** `lint the wiki` — three new pages added with cross-references on five existing pages; quick sweep for orphans / dead links / frontmatter gaps.
