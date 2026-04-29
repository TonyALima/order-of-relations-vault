---
type: question
title: "Should QueryBuilder.applyOptions() accumulate where-conditions instead of replacing?"
question: "Flip applyOptions() from last-call-wins to additive composition?"
answer_quality: open
created: 2026-04-29
updated: 2026-04-29
tags:
  - question
  - open
  - query-builder
  - api-design
status: open
related:
  - "[[QueryBuilder]]"
  - "[[Lazy Query Builder]]"
  - "[[sources/query-builder-design]]"
sources:
  - "../.raw/query-builder-design.md"
---

# Should `QueryBuilder.applyOptions()` accumulate where-conditions instead of replacing?

> [!note] Status: **open** (deferred; explicitly named in the source)

## Question

`QueryBuilder.applyOptions()` today **replaces** the where-conditions wholesale on every call (`this.conditions = results`). Calling `applyOptions({ where: A })` followed by `applyOptions({ where: B })` discards `A`'s conditions entirely.

Should it instead **accumulate** — AND the new conditions onto the existing array — to match the additive semantics most builder APIs have?

## Why the current behavior exists

The source: *"`applyOptions()` replaces conditions instead of accumulating them. This is consistent today but surprises people who expect builder semantics to be additive across calls. Worth revisiting if a real composition use case appears."*

So:

- **Why replace today:** repositories' `findMany(options?)` / `findOne(options?)` call `applyOptions` exactly once per builder. There is no scenario in the current codebase where two `applyOptions` calls on the same builder are intentional. Replace is the safe default — it can't surprise the dominant call path.
- **Why "this is worth revisiting":** the convention in most ORM/SQL builder ecosystems (Knex, Kysely, TypeORM) is additive. A future caller composing scopes (`baseQuery.applyOptions(scopeA).applyOptions(scopeB)`) would expect AND, get replace, and write a bug.

## Why it matters

- **API surprise.** "Builder" carries an expectation. Users coming from other ecosystems will reach for additive semantics by default, and the failure (silently dropped conditions) is hard to spot at the call site.
- **Composition is the *point* of laziness.** [[Lazy Query Builder]] makes deferred execution possible specifically so layered composition becomes safe. `applyOptions` being non-additive partly defeats that benefit.

## Counter-arguments to flipping

- **Last-call-wins is a real semantics**, used by some libraries deliberately (e.g., when applyOptions is treated as "configuration, not composition"). The choice isn't *wrong*; it's just *different from the dominant convention*.
- **Implicit accumulation can mask bugs.** A forgotten reset between two unrelated query paths can produce a query that ANDs unrelated filters. Replace makes "fresh builder, fresh state" obvious.
- **The fix is one line either way.** This is a deliberate choice now; switching later is a one-line code change but a breaking-API-semantics change.

## What would change in the codebase

If flipped to additive:

- `applyOptions()` body: replace `this.conditions = results` with `this.conditions.push(...results)`.
- Tests — add tests pinning the additive behavior (e.g., two `applyOptions` calls produce a query whose `WHERE` is `(A) AND (B)`).
- Repository wrappers (`findMany`, `findOne`) — verify they don't rely on the replace semantics.
- Documentation — the [[QueryBuilder]] component page's "applyOptions is replace, not accumulate" section flips.

## What would close this question

- **Concrete composition use case appears** — flip to additive, file an ADR if the change is non-trivial.
- **Decision to keep replace permanently** — note here why, mark question as `answered`.
- **Hybrid** — introduce a separate method (`andWhere`?) for additive composition, leave `applyOptions` as replace.

## Confidence

**Open** — the source explicitly conditions a revisit on "if a real composition use case appears."

## Related Questions

- [[get-one-limit-1]] — separately deferred from the same source.

## Sources

- `.raw/query-builder-design.md` §§ "Clause accumulation", "Trade-offs and open questions"
