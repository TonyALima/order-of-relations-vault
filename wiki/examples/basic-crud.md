---
type: example
title: "examples/basic-crud"
example_path: "examples/basic-crud/"
created: 2026-04-29
updated: 2026-04-29
tags:
  - example
  - crud
  - repository
status: stable
related:
  - "[[Repository Pattern]]"
  - "[[Repository]]"
  - "[[Autogeneration]]"
  - "[[lifecycle-of-a-create]]"
sources:
  - "../.raw/drift-m6-examples-pointer.md"
---

# examples/basic-crud

The **minimum-viable CRUD scenario.** The smallest end-to-end demonstration: one entity, one repository, three Repository methods.

## What It Demonstrates

| Decorator / API | File |
|---|---|
| `@Entity(db)` | `examples/basic-crud/entities/User.ts` |
| `@PrimaryColumn({ type: SERIAL, autogeneration: { dbSide: () => undefined } })` | `examples/basic-crud/entities/User.ts` |
| `@Column @NotNullable` | `examples/basic-crud/entities/User.ts` |
| `new Repository(User, db)` (no DI yet) | `examples/basic-crud/services/UserService.ts` |
| `repo.create(entity)` | `examples/basic-crud/services/UserService.ts` |
| `repo.findById(key)` | `examples/basic-crud/services/UserService.ts` |
| `repo.findMany(options?)` | `examples/basic-crud/services/UserService.ts` |
| `db.connect()` + `db.create()` wiring | `examples/basic-crud/db.ts`, `examples/basic-crud/index.ts` |

`User` is the canonical "User" the rest of the wiki keeps mentioning. When a wiki page says "imagine a `User` entity," this is the actual file.

## What It Does Not Cover

- Inheritance — see [[examples/inheritance]].
- Relations — see [[examples/relations]].
- The `inheritance` `FindOptions` field — only [[examples/inheritance]] exercises it.
- DI / `@Service` / `@InjectRepository` — those decorators are not yet implemented (see [[0003-singleton-di-container]]). The service constructs its repository directly.

## See Also

- [[Repository]] — the class this scenario exercises.
- [[Repository Pattern]] — the concept.
- [[Autogeneration]] — the `dbSide` strategy used on `User.id`.
- [[lifecycle-of-a-create]] — the end-to-end write flow this scenario triggers.
- [[sources/drift-m6-examples-pointer]] — why this page exists.
