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

- **Direct operations** — `findOne`, `create`, `update`, `delete`. These execute SQL synchronously (well, asynchronously, but eagerly: the returned `Promise` settles with the row(s) or affected count). Used for the trivial case.
- **Composition entry point** — `find()`. This does **not** execute SQL. It returns a [[Lazy Query Builder]] that accumulates clauses and runs only when a terminal method (`getMany`, `getOne`, `getCount`) is called.

This split is the load-bearing design of the persistence API. Trivial cases stay trivial (`repo.findOne({ id })` is one line); composed cases scale up through a separate type without bloating the repository's surface.

`Repository<T>` is itself injected via the [[Dependency Injection Container]]: a service annotates a field with `@InjectRepository(Entity)` and the container wires the singleton.

## Why It Matters

- Establishes a **clean boundary**: persistence concerns are isolated in one class per entity. Domain services don't issue SQL; they call `repo.X()`.
- Enables **substitution** for tests: a test can inject a fake repository without touching SQL plumbing.
- The trivial/composed split prevents the API from becoming a bag of overloaded methods (the failure mode that haunts older ORMs).

## Examples

```ts
@Service
class UserService {
  @InjectRepository(User)
  private repo!: Repository<User>;

  async getActive(): Promise<User[]> {
    // composed read — uses the QueryBuilder
    return this.repo.find()
      .where({ active: true })
      .orderBy("createdAt", "DESC")
      .getMany();
  }

  async byId(id: number): Promise<User | null> {
    // trivial read — direct on the repository
    return this.repo.findOne({ id });
  }
}
```

## Connections

- [[0002-repository-with-lazy-query-builder]] — the ADR fixing the boundary.
- [[Lazy Query Builder]] — the type returned by `find()`.
- [[Dependency Injection Container]] — the wiring mechanism for `@InjectRepository`.
- [[Repository Contract]] — the per-method contract (page populated by a future ingest of `.raw/repository-contract.md`).

## Sources

- `.raw/welcome.md` § "Repository pattern with a lazy query builder"
