---
type: question
title: "Should decorator order be made order-independent for @Column / @Nullable?"
question: "Can we make @Column and @Nullable on the same property work regardless of which one is the inner decorator?"
answer_quality: open
created: 2026-04-29
updated: 2026-04-29
tags:
  - question
  - open
  - decorators
  - design-review
status: open
related:
  - "[[ECMAScript Stage-3 Decorators]]"
  - "[[entity-registration]]"
  - "[[MetadataStorage]]"
sources:
  - "../.raw/decorator-metadata-storage.md"
---

# Should decorator order be made order-independent for `@Column` / `@Nullable`?

> [!note] Status: **open** (filed for future resolution, not yet decided)

## Question

Can we eliminate the decorator-order constraint between `@Column` / `@PrimaryColumn` and `@Nullable` / `@NotNullable`, so that both orderings work?

```ts
// Currently works — @Nullable inner (runs first), @Column reads its entry.
@Column({ type: COLUMN_TYPE.TEXT })
@Nullable
nickname?: string;

// Currently throws MissingNullabilityDecoratorError.
@Nullable
@Column({ type: COLUMN_TYPE.TEXT })
nickname?: string;
```

## Why it matters

- **Footgun.** `MissingNullabilityDecoratorError` names the property and decorator, but doesn't tell the user that *the cause is the order they wrote them in*. New contributors to OOR will hit this; experienced ones will hit it after a refactor.
- **Doesn't reflect a meaningful difference.** A consumer's intent — "this column is nullable" — is the same regardless of which line they wrote `@Nullable` on. The order constraint encodes implementation timing, not user intent.
- **Decorator order is invisible at PR review.** A reviewer skimming `@Column ... @Nullable` vs `@Nullable ... @Column` is unlikely to notice. The compiler doesn't catch it; only the runtime does, at module load.

## Current behavior (so the question doesn't decay)

Implementation lives in:

- `../order-of-relations/src/decorators/nullable/nullable.ts` — declares `NULLABLE_KEY = Symbol('nullable')`. `@Nullable` / `@NotNullable` set `Map<string, boolean>` entries keyed by property name.
- `../order-of-relations/src/decorators/column/column.ts` (lines 31–37) — `@Column`'s `registerColumn` reads `NULLABLE_KEY[propertyName]`, throws `MissingNullabilityDecoratorError` if `undefined` and the column is not primary, then bakes the resolved `nullable` field into the pushed `ColumnMetadata`.

Stage-3 applies decorators on a field **bottom-up** (the inner decorator runs first), so today the working order is `@Column` outer, `@Nullable` inner. See [[ECMAScript Stage-3 Decorators]] § "Decorator order matters" and [[entity-registration]] § Step 1.

## Sketch of the design space

Three approaches worth considering. None is endorsed here — this question is open precisely so the choice can be deliberate.

### Option A — Defer the join to `@Entity`

`@Column` doesn't read `NULLABLE_KEY` at all. It pushes a `ColumnMetadata` with `nullable: undefined` (or some sentinel) onto `COLUMNS_KEY`. `@Nullable` continues to write `NULLABLE_KEY` as today.

`@Entity` (which already runs last) does the join: for each entry in `COLUMNS_KEY`, look up the corresponding `NULLABLE_KEY` entry, fill in `nullable`, and throw `MissingNullabilityDecoratorError` if missing.

**Pros:** Simplest. The error message can now name "missing `@Nullable`" without conflating it with order. Order genuinely doesn't matter — both decorators have written their slot by the time `@Entity` runs.

**Cons:** The error is reported with the class context (`@Entity` knows which class it is, but emitting per-property errors is fine). Validation moves from "decoration time" to "still decoration time, just slightly later" — no real semantic loss.

### Option B — Mutual lookup at write time

Both decorators check the *other* bucket. If `@Column` runs first, it pushes onto `COLUMNS_KEY` with `nullable: undefined`. When `@Nullable` runs, if it finds a matching entry in `COLUMNS_KEY`, it patches it; otherwise it writes to `NULLABLE_KEY` as today. `@Column`, when it runs, similarly checks `NULLABLE_KEY` first (current behavior).

**Pros:** Each decorator self-heals, so order in either direction works.

**Cons:** Two write paths per decorator, both of which can fail to land if a sibling decorator never runs. More state machine surface area; harder to reason about during PR review. The "mutual patching" pattern is a smell — it spreads the invariant across more code.

### Option C — Stage-3 `addInitializer` to defer

`@Column` and `@Nullable` both register `addInitializer` callbacks that fire after class construction. The callbacks reconcile their bucket states.

**Pros:** Fits Stage-3's intended deferred-work hook.

**Cons:** Heavier. `addInitializer` runs at instance time, not class time — wrong granularity for *class-level* metadata. Also moves errors from decoration time to first-instance time, which is a regression on the [[ECMAScript Stage-3 Decorators]] / [[entity-registration]] property "errors at decoration time, not first-use time."

## Things to verify before deciding

- Are there any decorators OOR plans to add (`@Index`, `@Unique`, `@Default`) that would have the **same** sibling-coordination shape? If yes, the design choice should solve the general case, not just `@Column`/`@Nullable`. **Option A scales to N siblings**; Option B scales as O(N²) write paths.
- What does TypeORM/MikroORM/Drizzle do with their equivalent (their nullable-on-column pattern)? A look at prior art could short-circuit the analysis.
- Does the current error message catch this in practice? If users hit the error and immediately see what to do, the footgun cost is low. If they don't, it's high.

## What would change in the codebase

If Option A is chosen:

- `column.ts` — drop the `NULLABLE_KEY` read and the `MissingNullabilityDecoratorError` throw. Push column with `nullable: undefined`.
- `entity.ts` — add a join pass over `COLUMNS_KEY` and `NULLABLE_KEY` before commit; throw `MissingNullabilityDecoratorError` from here instead.
- Tests — add a test asserting that `@Nullable @Column ...` (reversed order) succeeds.
- The `> [!warning]` callouts on [[ECMAScript Stage-3 Decorators]] and [[entity-registration]] flip from "must be inner" to "either order works."

## Confidence

**Open** — no decision yet. This page exists to keep the question discoverable until it's decided.

## Related Questions

- *(none filed yet)*

## Sources

- `.raw/decorator-metadata-storage.md` (current symbol scheme)
- `../order-of-relations/src/decorators/nullable/nullable.ts`, `../order-of-relations/src/decorators/column/column.ts` (current implementation, verified 2026-04-29)
