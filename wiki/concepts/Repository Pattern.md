---
type: concept
title: "Repository Pattern"
complexity: intermediate
domain: "ORM design"
aliases:
  - "Repository"
  - "Repository<T>"
created: 2026-04-29
updated: 2026-04-29
tags:
  - concept
  - persistence
  - orm
status: seed
related:
  - "[[0002-repository-with-lazy-query-builder]]"
  - "[[Lazy Query Builder]]"
  - "[[sources/welcome]]"
sources:
  - "../.raw/welcome.md"
---

# Repository Pattern

## Definition

A **Repository** is the single object responsible for persistence of one entity type. It mediates between the domain (typed entity classes) and the storage (PostgreSQL rows), exposing CRUD operations and a composition entry point for reads.

In OOR the type is `Repository<T>`, parameterized by the entity class. One repository per entity; one entity per repository.

## How It Works

`Repository<T>` exposes two surfaces:

- **Write operations build SQL directly.** `create`, `update`, `delete` — anything where the SQL shape is fully determined by the entity metadata and the input. No composition, no query builder.
- **Read operations delegate to [[Lazy Query Builder|QueryBuilder]].** `findOne`, `findMany`, `findById` all construct a `QueryBuilder<T>` internally and call its terminal methods. They are **not** direct executors — they are thin wrappers that instantiate a builder, apply options, and call `getMany()` / `getOne()`. From the caller's view the call still returns rows; under the hood, every read goes through the builder.

The boundary is therefore not "trivial methods on Repository, composed methods on QueryBuilder" but rather "**writes on Repository, reads through QueryBuilder**." Reads can still be invoked off the repository for ergonomics; the SQL is composed in one place.

> [!note] Refined 2026-04-29
> An earlier version of this page split the surface as "direct operations execute SQL synchronously" vs. "`find()` returns a builder." Per `.raw/architecture-overview.md`, the actual split is by **direction** (read vs. write), not by **shape** (trivial vs. composed). All reads — even `findById` — delegate to `QueryBuilder`.

`Repository<T>` is currently constructed directly: `new Repository(User, db)`. The originally-planned `@InjectRepository(Entity)` decorator and accompanying [[Dependency Injection Container|DI container]] are **not yet implemented in `src/`** — see the implementation-status callout on [[0003-singleton-di-container]].

## Why It Matters

- Establishes a **clean boundary**: persistence concerns are isolated in one class per entity. Domain services don't issue SQL; they call `repo.X()`.
- Enables **substitution** for tests: a test can inject a fake repository without touching SQL plumbing.
- The trivial/composed split prevents the API from becoming a bag of overloaded methods (the failure mode that haunts older ORMs).

## Examples

Current `examples/` style — direct construction, no DI yet:

```ts
const userRepository = new Repository(User, db);

// Composed read — `where` is a callback receiving a typed proxy.
// See [[Conditions Proxy]].
const active = await userRepository.findMany({
  where: (u) => [u.active!.eq(true)],
});

// findById is a thin wrapper that builds a primary-key `where`
// and delegates to QueryBuilder, just like findMany.
const user = await userRepository.findById(42);
```

Planned ergonomics, **once DI ships** (currently aspirational — see [[0003-singleton-di-container]]):

```ts
@Service
class UserService {
  @InjectRepository(User)
  private repo!: Repository<User>;
  // ...
}
```

## Connections

- [[0002-repository-with-lazy-query-builder]] — the ADR fixing the boundary.
- [[Lazy Query Builder]] — the type all read methods delegate to.
- [[query-lifecycle]] — the end-to-end six-step walkthrough of a read (Repository entry is step 1).
- [[Conditions Proxy]] — the typed object the `where` callback receives.
- [[Dependency Injection Container]] — the planned wiring mechanism for `@InjectRepository` (not yet implemented).
- [[Repository Contract]] — the per-method contract (page populated by a future ingest of `.raw/repository-contract.md`).

## Sources

- `.raw/welcome.md` § "Repository pattern with a lazy query builder"
