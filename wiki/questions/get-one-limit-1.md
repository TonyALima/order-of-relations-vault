---
type: question
title: "Should QueryBuilder.getOne() emit LIMIT 1 instead of slicing?"
question: "Switch getOne() from client-side row[0] to a SQL-level LIMIT 1?"
answer_quality: open
created: 2026-04-29
updated: 2026-04-29
tags:
  - question
  - open
  - query-builder
  - performance
status: open
related:
  - "[[QueryBuilder]]"
  - "[[Lazy Query Builder]]"
  - "[[sources/query-builder-design]]"
sources:
  - "../.raw/query-builder-design.md"
---

# Should `QueryBuilder.getOne()` emit `LIMIT 1` instead of slicing?

> [!note] Status: **open** (deferred; explicitly named in the source)

## Question

`QueryBuilder.getOne()` today calls `getMany()` and returns `rows[0] ?? null`. Should it instead emit a `LIMIT 1` clause in the SQL, so the database returns at most one row in the first place?

## Why it matters

- **Performance on large filtered sets.** Today, a `getOne()` against a `where` that matches 50,000 rows fetches all 50,000, ships them over the wire, parses them client-side, and discards 49,999. With `LIMIT 1`, the database stops after the first match.
- **Memory pressure.** The intermediate full-result `T[]` exists in JS heap space, even if 99.998% of it is immediately thrown away.
- **Honesty.** "Get one" semantically commits to one. Today the implementation doesn't reflect that.

## Why the current behavior exists

The source: *"It does **not** add `LIMIT 1` to the SQL today; it slices the result client-side. That's fine for the small result sets the current API targets, and it's a one-line change when it stops being fine."*

So this is a deliberate "ship the simpler thing, swap when it bites" call — not an oversight.

## What would change in the codebase

Approximate change surface:

- `src/query-builder/query-builder.ts` (or wherever `getOne` lives) — the single-line swap from `(await this.getMany())[0] ?? null` to a `LIMIT 1` clause appended to the composed SQL, with the result-extraction adjusted accordingly.
- Tests — add a test that confirms a large result set doesn't materialize fully under `getOne` (could be done by counting bind invocations in a mocked driver, or by an integration test against PostgreSQL with `EXPLAIN ANALYZE` in CI).

## Open considerations

- **`getOne()` is currently semantically equivalent to "the first row of `getMany()`".** With `LIMIT 1`, ordering matters: you'd want a stable order, but the builder doesn't yet support `orderBy`. So shipping `LIMIT 1` alone may surface non-determinism that the slicing approach quietly hid.
- **Coupling to ordering.** Once `orderBy` lands as a builder feature, `LIMIT 1` becomes the obviously-correct choice. Until then, the deterministic-row-zero question is genuine.

## What would close this question

A decision either way:

- **"Ship `LIMIT 1` now"** — accepts non-deterministic behavior across calls until `orderBy` lands.
- **"Wait for `orderBy`"** — keeps slicing until the builder also supports stable ordering, then switch both at once.
- **"Always slice"** — settles that the simpler implementation is acceptable indefinitely, and removes this question.

## Confidence

**Open** — the source explicitly defers this; no decision logged.

## Related Questions

- *(none filed yet — but `orderBy` support is implicitly upstream of any "ship `LIMIT 1` with deterministic results" answer.)*

## Sources

- `.raw/query-builder-design.md` §§ "Terminal methods", "Trade-offs and open questions"
