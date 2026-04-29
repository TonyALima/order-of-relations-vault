---
type: source
title: "Drift D1 — Repository.find() does not exist"
source_type: drift-correction
author: "Tony Albert"
date_published: 2026-04-29
url: ""
confidence: high
key_claims:
  - "Repository<T> exposes exactly six public methods: findMany, findOne, findById, create, delete, update."
  - "There is no find() method that returns a QueryBuilder<T> to the caller."
  - "QueryBuilder is constructed inside findMany / findOne, used once, and discarded."
  - "QueryBuilder is constructible directly (`new QueryBuilder<T>(EntityClass, db)`) but that is a library-internal escape hatch, not a documented user API."
  - "Earlier wiki pages framed find() as the composition entry point and findMany / findOne as sugar — that framing is wrong."
created: 2026-04-29
updated: 2026-04-29
tags:
  - source
  - drift
status: stable
related:
  - "[[Repository]]"
  - "[[Repository Pattern]]"
  - "[[Lazy Query Builder]]"
  - "[[brief]]"
sources:
  - "../.raw/drift-d1-repository-find.md"
---

# Drift D1 — Repository.find() does not exist

## Summary

A correction note flagging that earlier wiki pages — including [[brief]], [[Repository]], [[Repository Pattern]], and [[Lazy Query Builder]] — describe `Repository<T>.find(): QueryBuilder<T>` as the canonical "handoff" that returns a builder without running SQL. **That method is not implemented.** The wiki overstated the public surface.

## What the Code Actually Exposes

`src/core/repository/repository.ts` exports a `Repository<T>` class with exactly six public methods:

```ts
findMany(options?: FindOptions<T>): Promise<T[]>
findOne(options?:  FindOptions<T>): Promise<T | null>
findById(key: Partial<T>):           Promise<T | null>
create(entity: T):                    Promise<Partial<T>>
delete(key: Partial<T>):              Promise<void>
update(entity: T):                    Promise<void>
```

Every read path is **terminal**. The builder is constructed inside `findMany` / `findOne` (e.g. `new QueryBuilder<T>(this.entity, this.db).applyOptions(options).getMany()`), used once, and discarded. The examples (`examples/basic-crud/`, `examples/inheritance/`) only ever call the six methods above.

Direct construction (`new QueryBuilder<T>(EntityClass, db)`) works because `QueryBuilder` is exported transitively, but the repository does not facilitate it — it's a library-internal hatch, not a documented user API.

## Why the Drift Mattered

The "find() returns a builder; findMany / findOne are sugar over it" framing misleads agents about how to compose queries. The correct framing today is: **`findMany` / `findOne` are the composition entry points**; they accept `FindOptions<T>` and execute one SQL statement. There is no caller-facing builder handoff.

## Resolution

Removed the `find()` clause from [[brief]], [[Repository]], [[Repository Pattern]], and [[Lazy Query Builder]]. Reframed the read API around `findMany` / `findOne` as composition entry points and `findById` as the by-key reader. The "sugar" framing is gone — these are not sugar over anything.

## Source Citations

- `src/core/repository/repository.ts` — full method list (lines 1–183). No `find` symbol.
- `src/index.ts` — public exports; only `Repository` is exported, no `find` helper.
- `examples/basic-crud/services/UserService.ts`, `examples/inheritance/services/UserHierarchyService.ts` — actual call sites.

## Pages Affected

- [[brief]] — find() clause removed
- [[Repository]] — find() row removed from operations table; "find() is a handoff" section rewritten
- [[Repository Pattern]] — find() example removed; surface description rewritten
- [[Lazy Query Builder]] — "returned by Repository.find()" replaced with "constructed inside findMany / findOne and discarded"
