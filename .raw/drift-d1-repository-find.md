# Drift Correction — `Repository.find()` does not exist

This is a correction note. Earlier wiki pages — including the agent-facing brief — describe `Repository<T>.find(): QueryBuilder<T>` as the canonical "handoff" that returns a builder without running SQL. **That method is not implemented.** The wiki overstated the public surface.

## What the code actually exposes today

`src/core/repository/repository.ts` exports a `Repository<T>` class with exactly six public methods, in declaration order:

```
findMany(options?: FindOptions<T>): Promise<T[]>
findOne(options?:  FindOptions<T>): Promise<T | null>
findById(key: Partial<T>):          Promise<T | null>
create(entity: T):                   Promise<Partial<T>>
delete(key: Partial<T>):             Promise<void>
update(entity: T):                   Promise<void>
```

There is no `find()`. There is no other method that returns a `QueryBuilder<T>` to the caller. Every read path the repository owns is **terminal** — the builder is constructed inside `findMany` / `findOne` (`new QueryBuilder<T>(this.entity, this.db).applyOptions(options).getMany()` and `.getOne()`), used once, and discarded. Examples confirm this: `examples/basic-crud/services/UserService.ts` and `examples/inheritance/services/UserHierarchyService.ts` only ever call the six methods above.

Direct access to a `QueryBuilder<T>` *is* possible — `QueryBuilder` is exported transitively and constructible as `new QueryBuilder<T>(EntityClass, db)` — but that is a library-internal escape hatch, not a documented user-facing API. The repository does not facilitate it.

## Why this matters

Two of the most-loaded wiki pages — `wiki/components/Repository.md` and the agent brief — frame the read surface around `find()` as the "composition entry point" and treat `findMany` / `findOne` as *sugar* over it. That framing is wrong on its face when there is no `find()`. It also misleads agents about how to compose queries: the right answer today is "use `findMany({ where: ... })` and live with last-call-wins on `applyOptions`," not "grab the builder via `find()` and chain."

## Way to resolve

1. **Drop `find()` from the documented surface entirely.** Reframe the read API as: *"`findMany` / `findOne` are the composition entry points; they accept `FindOptions<T>` and execute one SQL statement. There is no caller-facing builder handoff today."* This is more honest but loses the design seam that `find()` represented.

## Wiki pages affected

| Page | Current claim | Action |
|---|---|---|
| `wiki/brief.md` | *"`find()` returns a `QueryBuilder`; `findOne` / `findMany` are sugar."* | Remove `find()` clause; either drop the sentence or replace with *"`findMany` / `findOne` are the composition entry points; `findById` is the by-key reader."* |
| `wiki/components/Repository.md` | Operations table lists `find()`; section "`find()` is a handoff" runs ~5 lines | Remove the row from the table; remove or rewrite the entire `## find() is a handoff` section. The "sugar" framing of `findOne` / `findMany` needs rephrasing — they are *not* sugar over a non-existent method. |
| `wiki/concepts/Repository Pattern.md` | *"`find()` (returns a `QueryBuilder<T>` and runs no SQL), plus `findOne(options?)` / `findMany(options?)` which are sugar"* and example `const builder = userRepository.find();` | Remove the example line; rewrite the surface description without the "sugar" framing. |
| `wiki/concepts/Lazy Query Builder.md` | *"In OOR the type is `QueryBuilder<T>`, returned by `Repository<T>.find()`."* and *"the builder returned by `Repository.find()` (when present in the API surface)"* | The second occurrence already hedges with *"when present in the API surface"* — that hedge is correct; align the first occurrence with it. Replace *"returned by `Repository<T>.find()`"* with *"constructed inside the repository's read methods (`findMany`, `findOne`) and discarded after one terminal call."* |
| `wiki/sources/repository-contract.md` (if it cites `find()` from the original raw) | *as written* | Leave the source synthesis as-is for fidelity, but add a top-level "**Drift note**" callout pointing at this file. |

## Source citations

- `src/core/repository/repository.ts` — full method list, lines 1–183. No `find` symbol.
- `src/index.ts` — public exports; only `Repository` is exported, no `find` helper.
- `examples/basic-crud/services/UserService.ts` — actual call sites: `findMany`, `findById`, `create`.
- `examples/inheritance/services/UserHierarchyService.ts` — actual call sites: `findMany({ inheritance })`, `create`.
