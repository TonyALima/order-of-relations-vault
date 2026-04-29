---
type: concept
title: "Schema Migrations"
complexity: foundational
domain: "Database lifecycle"
aliases:
  - "Migrations"
  - "Schema evolution"
created: 2026-04-29
updated: 2026-04-29
tags:
  - concept
  - schema
  - migrations
  - planned
status: seed
related:
  - "[[Database]]"
  - "[[schema-create-drop]]"
  - "[[sources/welcome]]"
  - "[[sources/architecture-overview]]"
sources:
  - "../.raw/welcome.md"
  - "../.raw/architecture-overview.md"
---

# Schema Migrations

## Definition

**Schema migrations** are versioned, ordered, replayable transformations that evolve a populated database from one schema state to another **without destroying data**. They are the production-time counterpart to [[Database]]'s `create()` / `drop()`, which are full-rebuild operations only safe against an empty database (or in tests).

Examples of operations that need migrations:

- Adding a column to an existing table without losing rows.
- Renaming a column or changing a type (`int` → `bigint`, `text` → `varchar(255)`).
- Adding or removing indexes / unique constraints / foreign keys.
- Backfilling new columns with values computed from existing rows.
- Recording which migrations have already been applied (typically a `schema_versions` or `schema_migrations` table).

> [!gap] Not yet implemented
> As of 2026-04-29, OOR has **no migration system** in `src/`. [[sources/welcome]] lists *"Schema-based migrations evolve the database over time"* as one of the five pillars; [[sources/architecture-overview]] § "Database, Schema, and Lifecycle" explicitly defers it: *"Schema migrations beyond `create` / `drop` belong in [[Schema Migrations]] — currently a placeholder."*
>
> What exists today: [[Database]]`.create()` and `[[Database]].drop()` (full-rebuild only). What is missing: every operation listed above.
>
> This page exists so that the dead `[[Schema Migrations]]` wikilink in [[Database]] and [[schema-create-drop]] resolves to a real concept page, and so that a future `.raw/schema-migrations.md` ingest has a natural target to deepen.

## Why a separate concept (not a `Database` method)

Migrations need things `Database.create()` doesn't:

- **State.** Migrations track what's been applied; a fresh `create()` doesn't.
- **Ordering.** Migrations must run in a defined sequence; `create()` is a single set-of-tables operation.
- **Reversibility (or explicit one-way).** Some migration models support `up()`/`down()`; others are forward-only with a roll-forward fix-it discipline.
- **Idempotence at the migration level.** Re-running an applied migration is a no-op (or an error); `create()` is fundamentally not idempotent against a populated DB.
- **Off-cycle execution.** Migrations may be triggered as part of a deploy, not at every process startup. `Database.create()` runs in tests every time.

Bundling these into `Database` would inflate it past its current "three jobs" scope (connection / metadata host / schema lifecycle). The single-responsibility argument that keeps `Database` small applies in reverse: migrations want their own home.

## Future scope (decisions to make when this is designed)

When OOR's migration system is actually built, these are the load-bearing choices that should each become an ADR:

1. **Versioning model:** sequential integers (`0001_init.sql`)? timestamps (`20260429-add-email.ts`)? content-addressable hashes? Sequential is simplest; timestamps avoid merge-conflicts on concurrent branches.
2. **Forward-only vs. up/down:** does every migration need a `down()` that reverses it? Forward-only is operationally simpler (and aligns with most modern advice — see Stripe's blog, Square's "no rollbacks" philosophy); up/down is more flexible but adds maintenance burden.
3. **TS-first vs. SQL-first:** are migrations TypeScript files that emit SQL via the [[Lazy Query Builder|builder]] / [[sqlJoin]], or raw `.sql` files? TS-first composes with the type system; SQL-first is more transparent.
4. **Runner architecture:** does the runner live in OOR, or does the user wire their own? An in-library runner is ergonomic; user-controlled gives flexibility.
5. **Migration table location:** OOR-owned (`oor_schema_versions`) or user-named?
6. **Relationship to `Database.create()`:** does `create()` write a "version 0" baseline that migrations build on, or do migrations and `create()` describe the same end-state via different paths?

None of these is decided. None should be decided here in this stub — when it's time, file an ADR.

## What this page becomes when implemented

The seed status will flip to `developing` and then `stable` as:

- A future `.raw/schema-migrations.md` is ingested (the design rationale).
- One or more ADRs land under `wiki/decisions/` (the load-bearing choices above).
- A `wiki/components/MigrationRunner.md` (or similar) component page captures the concrete class.
- A `wiki/flows/migration-apply.md` flow page walks through a single up-migration end to end.

The current page is the placeholder that holds the spot.

## Connections

- [[Database]] — owner of `create()` / `drop()`; future migration runner will live alongside (or above) it.
- [[schema-create-drop]] — the existing flow; migrations are the *not-this* path for production data.
- [[Layered Architecture]] — wherever the migration runner lands, it will respect the downward-only dependency rule (it'll read [[MetadataStorage]] for the target schema, write SQL via [[Bun]]'s `SQL`, and not be imported by anything below it).
- [[sources/welcome]] / [[sources/architecture-overview]] — both name "schema migrations" as planned but unbuilt.

## Sources

- `.raw/welcome.md` § "What OOR Is" (lists schema migrations as a pillar)
- `.raw/architecture-overview.md` § "Database, Schema, and Lifecycle" (explicitly defers)
