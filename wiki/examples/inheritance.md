---
type: example
title: "examples/inheritance"
example_path: "examples/inheritance/"
created: 2026-04-29
updated: 2026-04-29
tags:
  - example
  - inheritance
  - sti
status: stable
related:
  - "[[Single-Table Inheritance]]"
  - "[[QueryBuilder]]"
  - "[[MetadataStorage]]"
sources:
  - "../.raw/drift-m6-examples-pointer.md"
  - "../.raw/drift-d3-find-options-inheritance.md"
---

# examples/inheritance

Single-table-inheritance scenario with `User` and `AdminUser extends User`. **The only place in the codebase that demonstrates the `inheritance` `FindOptions` field in use.**

## Why It Is Load-Bearing

The `inheritance` field on `FindOptions<T>` and the `InheritanceSearchType` enum are shipped, exported, and exercised here — but until [[sources/drift-d3-find-options-inheritance]] was filed, the wiki didn't document them at all. An agent reading the wiki without the example would be unable to write a subclass-scoped query. **Reading this scenario is the fastest way to understand how STI reads compose.**

## What It Demonstrates

| Decorator / API | File |
|---|---|
| `@Entity(db) class User` (the STI root) | `examples/inheritance/entities/User.ts` |
| `@Entity(db) class AdminUser extends User` (the subclass) | `examples/inheritance/entities/AdminUser.ts` |
| `findMany({ inheritance: InheritanceSearchType.SUBCLASSES })` | `examples/inheritance/services/UserHierarchyService.ts` — `listSubClassUsers()` and `listSubClassAdmins()` |
| Bare `findMany()` (default `ALL`) — reads every row in the inherited table | `examples/inheritance/services/UserHierarchyService.ts` |
| One repository per subclass type (`new Repository(User, db)`, `new Repository(AdminUser, db)`) | `examples/inheritance/services/UserHierarchyService.ts` |
| `db.create()` emitting STI DDL — `CREATE TABLE` + `ALTER ADD COLUMN discriminator` + `CREATE INDEX idx_discriminator` | `examples/inheritance/db.ts`, `examples/inheritance/index.ts` |

## What to Read First

`examples/inheritance/services/UserHierarchyService.ts` — it's the single most under-cited file in the codebase. Every claim the wiki makes about `InheritanceSearchType.SUBCLASSES` traces back to its `listSubClassUsers()` / `listSubClassAdmins()` methods.

## See Also

- [[Single-Table Inheritance]] § Reading Across the Hierarchy — the user-facing API doc.
- [[QueryBuilder]] § `applyOptions()` — implementation of the `inheritance` branch.
- [[MetadataStorage]] — `resolveInheritance` / `discriminator` wipe rule that makes the predicate appear and disappear with the existence of siblings.
- [[schema-create-drop]] § STI emissions — what `db.create()` actually emits for this scenario.
- [[sources/drift-d3-find-options-inheritance]] — the gap this example exposes.
- [[sources/drift-m6-examples-pointer]] — why this page exists.
