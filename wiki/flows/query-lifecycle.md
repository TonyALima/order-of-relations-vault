---
type: flow
title: "Query Lifecycle (findMany)"
trigger: "Caller invokes Repository.findMany({ where: ... })"
outcome: "Promise<T[]> resolves with rows shaped as the entity"
created: 2026-04-29
updated: 2026-04-29
tags:
  - flow
  - query-builder
  - repository
status: stable
related:
  - "[[Repository Pattern]]"
  - "[[Lazy Query Builder]]"
  - "[[Conditions Proxy]]"
  - "[[MetadataStorage]]"
  - "[[sources/architecture-overview]]"
sources:
  - "../.raw/architecture-overview.md"
---

# Query Lifecycle (findMany)

End-to-end walkthrough of a single composed read. The example call:

```ts
userRepo.findMany({
  where: (u) => [u.email!.eq('a@b.com')],
});
```

The same shape applies to `findOne` (which is `getMany().then(rows => rows[0] ?? null)`) and `findById` (which assembles a primary-key `where` from the entity's primary columns).

---

## Step 1 — Repository entry

`Repository.findMany` constructs a fresh `QueryBuilder<User>`, bound to the entity constructor and the `Database`. **No SQL is generated yet.**

The repository's role on the read path is purely as a factory + parameter-passer; the real work happens inside the builder.

## Step 2 — Metadata resolution

The query builder calls `db.getMetadata().get(User)`.

On first access for an entity:

- `resolveInheritance` runs: computes `tableName` and `discriminator` for class-table inheritance.
- `resolveRelations` runs: fills in foreign-key column names that the relation decorators (`@ToOne`, etc.) left as `null` at decoration time.

Subsequent accesses hit a resolved cache. Resolution is per-process and once-only.

See [[MetadataStorage]] for the storage shape.

## Step 3 — Conditions proxy

The query builder builds a typed proxy from the entity's column metadata: **one `FieldConditionBuilder` per column**. The user's `where` callback receives that proxy.

The callback returns an **array** of `Condition` objects. Each call like `u.email!.eq('a@b.com')` produces one `Condition`.

See [[Conditions Proxy]] for the proxy mechanism and typing rules.

## Step 4 — Validation

The query builder scans the returned array for `undefined` entries. If any are present, it throws `UndefinedWhereConditionError`.

This catches a common bug: `u.foo?.eq(...)` evaluated against a non-existent column produces `undefined` instead of a `Condition`. Without this check, the query would silently drop the predicate.

## Step 5 — SQL composition

On `getMany()` (or implicitly when `findMany` calls it):

- Each `Condition` is mapped to a tagged-template SQL fragment.
- Operators (`=`, `<>`, `IN`, …) are pre-tokenised into a fragment table; they are **never** built from user input.
- Values stay as parameters in the tagged-template.
- Fragments are joined with `sqlJoin` (the only sanctioned join in the codebase).

There is no string concatenation of user-supplied data anywhere on this path. See [[Parameterized SQL]] and [[0004-parameterized-sql-only]].

## Step 6 — Execution

The composed `sql\`SELECT … FROM … WHERE …\`` is awaited against the [[Bun]] `SQL` connection. Rows come back already shaped as `T[]`.

---

## Failure Modes

| Where | What | Symptom |
|---|---|---|
| Step 2 | Entity never registered with this `Database` | Lookup returns `undefined`; downstream code crashes |
| Step 3 | Column name typo'd in `where` callback | Proxy access returns `undefined`; condition is `undefined` |
| Step 4 | `undefined` condition slipped through | Throws `UndefinedWhereConditionError` (caught here, not at runtime) |
| Step 5 | Operator not in fragment table | Throws at composition time, before the wire |
| Step 6 | Bun `SQL` returns a driver error | Promise rejects with the driver's error |

## See Also

- [[Repository Pattern]] — the entry point.
- [[Lazy Query Builder]] — the type that owns steps 3–5.
- [[schema-create-drop]] — the orthogonal flow on the schema side.

## Sources

- `.raw/architecture-overview.md` § "Lifecycle of a Query"
