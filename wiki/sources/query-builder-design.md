---
type: source
title: "Query Builder Design"
source_type: design-note
author: "Tony Albert"
date_published: 2026-04-29
url: ""
confidence: high
key_claims:
  - "QueryBuilder is mutable, not immutable-by-copy — short-lived, single-owner."
  - "Internal state today is one field: private conditions: Condition[] = []."
  - "applyOptions() replaces where-conditions wholesale — last-call-wins."
  - "Terminal methods today are getMany() and getOne() only. No getCount, no getExists, no streaming yet."
  - "getOne() slices client-side; it does NOT add LIMIT 1 to the SQL today."
  - "where callback signature: (conditions: Conditions<T>) => (Condition | undefined)[]."
  - "Conditions<T> = { [K in keyof T]?: FieldConditionBuilder<T[K]> } — the ? is what makes typo-tolerance type-check; UndefinedWhereConditionError catches it at runtime with the offending index."
  - "Operator tokens come from a static opFragments map keyed by a closed enum; never from user input."
  - "IN composes as `${col} IN ${sql(c.value)}` — Bun's sql(array) form binds each element; empty array produces valid IN () that matches nothing."
  - "sqlJoin is the only sanctioned way to join SQL fragments — used by Repository.create/delete/update and the builder itself."
  - "No OR / no nested boolean trees — flat AND list. Honest constraint."
  - "No qb.raw() escape hatch — would weaken parameterized-only invariant."
created: 2026-04-29
updated: 2026-04-29
tags:
  - source
  - query-builder
status: stable
related:
  - "[[sources/welcome]]"
  - "[[sources/architecture-overview]]"
  - "[[Lazy Query Builder]]"
  - "[[Conditions Proxy]]"
  - "[[QueryBuilder]]"
  - "[[sqlJoin]]"
sources:
  - "../.raw/query-builder-design.md"
---

# Query Builder Design

## Summary

The pin-down note for `QueryBuilder<T>`: why lazy, why mutable, what state it holds, the exact `where`-callback signature, the SQL-composition safety guarantees, and what's deliberately deferred. Refines two earlier wiki pages with concrete corrections (mutable not immutable; only two terminal methods today) and surfaces two **open questions** the source itself flags as deferred design choices.

## Key Claims

- **Lazy by design** for three reasons: composition (helpers can layer on a builder without re-issuing), single round-trip (one SQL statement, no client-side filtering), deferred decisions (inheritance discriminator strategy resolved at terminal time against a stable metadata snapshot).
- **Mutable single-owner object.** Internal state is one field today: `private conditions: Condition[] = []`. `applyOptions()` returns `this`; discriminator setters push directly. No copy-on-write.
- **`applyOptions()` is replace, not accumulate.** Calling it twice **replaces** the where-conditions wholesale. Last-call-wins. Partial composition is a future feature, not an accidental property.
- **The `where` callback signature** is the most opinionated piece of the API:
    ```ts
    where: (conditions: Conditions<T>) => (Condition | undefined)[]
    ```
   Callback (not object literal). Returns array. Entries may be `undefined` at the type level (`Conditions<T>` is `Partial`); rejected at runtime with `UndefinedWhereConditionError` carrying the offending **index**.
- **Mapped type for type-safety:** `Conditions<T> = { [K in keyof T]?: FieldConditionBuilder<T[K]> }`. `u.age.gt(18)` accepts `number`; typo on property is compile error; `?` allows `u.name?.eq(...)` to type-check on optional/inherited columns.
- **Two terminal methods today:** `getMany(): Promise<T[]>` and `getOne(): Promise<T | null>`. No `getCount()`, `getExists()`, or streaming yet.
- **`getOne()` slices client-side**, does **not** emit `LIMIT 1`. Acknowledged as suboptimal for large filtered sets — explicit open question.
- **SQL composition safety mechanics:**
  - Operators live in a static `opFragments` map of pre-built `sql\`=\``-style fragments, keyed by a closed `Condition['op']` enum. Never built from user input.
  - Column names go through Bun's `sql(identifier)` form.
  - Values go through normal parameter binding.
  - `IN` composes as ``sql`${col} IN ${sql(c.value)}` ``; Bun's `sql(array)` binds each element; empty array → valid `IN ()` (matches nothing).
- **`sqlJoin` is the only sanctioned fragment-joiner.** Used by `Repository.create`, `Repository.delete`, `Repository.update`, and the builder itself. The "hand-rolled `reduce` over fragments" pattern is explicitly rejected as a footgun.
- **Repository ↔ Builder seam (clarified):** repository owns by-key operations and writes (`findById`, `create`, `update`, `delete`); builder owns composition; `findOne(options?)` / `findMany(options?)` are repository convenience wrappers that internally instantiate a builder.
- **Deferred deliberately:** OR / nested boolean trees, `orderBy` / `limit` / `offset`, joins, raw escape hatch.

## Entities Mentioned

- [[order-of-relations]] — concrete references to `Repository.create`, `Repository.delete`, `Repository.update`, the builder file, and the test suite that pins `IN ()` behaviour.
- [[Bun]] — the `sql` tagged-template form, including `sql(identifier)` and `sql(array)` overloads.

## Concepts Introduced

- [[sqlJoin]] — the only sanctioned helper for joining SQL fragments.

## Concepts Refined

- [[Lazy Query Builder]] — corrected from "conceptually immutable" to **mutable, single-owner**. Terminal-methods list narrowed to `getMany`/`getOne` (drop `getCount`).
- [[Conditions Proxy]] — exact mapped-type signature added; partial-mapped-type rationale; `UndefinedWhereConditionError` carries the offending index.
- [[query-lifecycle]] — Step 5 (SQL composition) deepened with the `opFragments` static map and `sql(c.columnName)` mechanism.
- [[Repository Pattern]] — the read/write split sharpened to "by-key + writes" vs. "composition."

## Components Introduced

- [[QueryBuilder]] — the concrete class with its single-field state, `applyOptions()` semantics, and terminal methods.

## Open Questions Filed

- [[get-one-limit-1]] — should `getOne()` emit `LIMIT 1` instead of slicing client-side?
- [[apply-options-accumulation]] — should `applyOptions()` accumulate where-conditions instead of replacing them?

## Notes

The author's "throughline" for the page: *keep the builder small, keep the SQL parameterized, keep the types honest. Everything else is future work, and the current shape is meant to absorb that work without rewrites.* That sentence is the criterion against which any future builder-API addition should be evaluated.
