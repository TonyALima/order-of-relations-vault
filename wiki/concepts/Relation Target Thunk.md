---
type: concept
title: "Relation Target Thunk"
complexity: intermediate
domain: "Decorator patterns"
aliases:
  - "getTarget thunk"
  - "Forward-reference closure"
  - "Lazy entity reference"
created: 2026-04-29
updated: 2026-04-29
tags:
  - concept
  - decorators
  - relations
status: stable
related:
  - "[[MetadataStorage]]"
  - "[[entity-registration]]"
  - "[[sources/decorator-metadata-storage]]"
sources:
  - "../.raw/decorator-metadata-storage.md"
---

# Relation Target Thunk

## Definition

In OOR, `@ToOne` (and any future relation decorator) declares its target entity as a **closure that returns the constructor** — a *thunk* — rather than a direct reference:

```ts
@ToOne(() => User)   // <-- closure
declare author: User;
```

The thunk is stored on `RelationMetadata` as `getTarget()`. Resolution calls it later, after all classes are loaded.

## How It Works

A direct constructor reference at decoration time fails on circular entity graphs because of JavaScript's **temporal dead zone (TDZ)**:

```ts
@Entity(db)
class Post {
  @ToOne(User)   // <-- ReferenceError if User isn't yet declared
  declare author: User;
}

@Entity(db)
class User { /* ... */ }
```

The decorator on `Post.author` runs as the `Post` class is evaluated — *before* `User`'s class declaration has executed. Reading `User` here throws.

A closure dodges the TDZ:

```ts
@ToOne(() => User)
```

The `User` reference is now inside a function body, evaluated only when the function is called. Resolution happens later — specifically, during `MetadataStorage`'s relation-resolution pass, after every `@Entity` has run.

## Why It Matters

- **Circular graphs become trivial.** `User` has many `Post`s; each `Post` belongs to a `User`. Both directions can reference each other at decoration time without ordering constraints.
- **Forward references work.** A relation can target a class that hasn't been declared yet. As long as it *is* declared (and decorated) by the time storage resolution runs, the thunk succeeds.
- **No string-keyed lookup.** Some ORMs use `@ToOne('User')` to dodge the same TDZ problem, then resolve the string against a name registry. The thunk approach keeps things type-safe — TypeScript checks the closure body — without sacrificing the deferred resolution.

## When the Thunk Is Called

Inside [[MetadataStorage]]'s `resolveRelations` pass (Step 4 of [[entity-registration]]):

1. For each `RelationMetadata` in storage, invoke `getTarget()`.
2. Look up the returned constructor in the storage `Map`.
3. If found, fill in foreign-key column names and other resolved fields.
4. If **not** found (the target was never registered with `@Entity`), throw `RelationTargetNotFoundError` with the target class name and the source's relation path (e.g., `posts.author`).

This is the canonical "I forgot to import the related entity, so its decorator never ran" failure mode — caught at first storage read, with enough context to diagnose.

## Pitfalls

- **The closure must capture the constructor by reference**, not by value. `@ToOne(() => User)` works; `(() => 'User')` does not — the resolver compares constructors by identity.
- **Lazy resolution defers the error.** A typo'd target won't fail at decoration time; it'll fail at the first `db.getMetadata().get(...)` after the relation is registered. Most user code triggers that read early enough that the deferral is invisible.
- **The thunk runs once per resolution pass.** Side effects in the closure body (don't put any) would fire repeatedly if the resolution flag is reset by a late `set()`.

## Connections

- [[MetadataStorage]] — calls the thunk during relation resolution; throws `RelationTargetNotFoundError` on miss.
- [[entity-registration]] — Step 4 ("First read triggers resolution") is where thunks are invoked.
- [[ECMAScript Stage-3 Decorators]] — the language semantics that make TDZ a real problem at decoration time.

## Sources

- `.raw/decorator-metadata-storage.md` §§ "What's stored", "Failure modes"
