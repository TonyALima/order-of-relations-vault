# Drift Correction — the `examples/` directory is invisible to the wiki

The repo ships an `examples/` directory containing runnable canonical usage. The wiki — including the agent-facing brief — never references it. For an agent doing "how do I actually call this API?" lookups, that's a costly omission: every wiki page describes the API in prose, but the only source-of-truth for "what does a complete, working call site look like?" is `examples/`, and the agent has to discover it without help.

## What `examples/` contains

Three subdirectories, each a self-contained scenario with the same shape (`db.ts` + `entities/` + `services/` + `index.ts`):

### `examples/basic-crud/`

The minimum-viable CRUD scenario.

- `entities/User.ts` — `@Entity` + `@PrimaryColumn({ type: SERIAL, autogeneration: { dbSide: () => undefined } })` + two `@Column @NotNullable` text fields. The canonical "User" the rest of the wiki keeps mentioning.
- `services/UserService.ts` — `new Repository(User, db)` plus `create`, `findMany`, `findById`. Demonstrates the no-DI direct-construction pattern that `[[0003-singleton-di-container]]` flags as "current state."
- `db.ts`, `index.ts` — the wiring. `db.ts` exports the singleton `Database` instance; `index.ts` runs `db.connect()` + `db.create()` + the service.

### `examples/inheritance/`

The single-table-inheritance scenario.

- `entities/User.ts` + `entities/AdminUser.ts` (`extends User`).
- `services/UserHierarchyService.ts` — **the canonical example of the `inheritance` `FindOptions` option** (see `[[drift-d3-find-options-inheritance]]`). Calls `findMany({ inheritance: InheritanceSearchType.SUBCLASSES })` for both the base and the subclass repository.

This file is the most under-cited piece of the codebase from the wiki's perspective: it's the *only* place that demonstrates the `InheritanceSearchType` option, and the wiki currently doesn't document that option at all. Linking to it from `[[Single-Table Inheritance]]` and `[[QueryBuilder]]` would close half the documentation gap on its own.

### `examples/relations/`

The relations / `@ToOne` scenario. (Not opened in this drift pass; the wiki agent should read it before linking.) Likely demonstrates the FK-column-name auto-derivation and the SERIAL→INTEGER FK type demotion documented in `[[drift-m5-sql-types-module]]`.

## Why this matters

- **For agents:** the brief is loaded into every agent session via `CLAUDE.md`. Adding a one-line pointer to `examples/` is the cheapest possible improvement to agent grounding — it shaves N grep operations off every "how do I use the API" question.
- **For the wiki itself:** several pages currently embed inline code samples that duplicate (and sometimes contradict) what's in `examples/`. Linking to `examples/` lets those samples shrink to "see `examples/<scenario>/services/<File>.ts`" while staying authoritative.
- **For drift detection:** if `examples/` is the stated authoritative source, drift detection becomes "do the wiki snippets match the example files?" — a much cheaper check than "do the wiki snippets match the codebase as a whole."

## Suggested wiki coverage

Two distinct things to do, on different scales:

### Small change — add a pointer

A single line in the brief and the index, stating that `examples/` exists and what's in each subdirectory. This is the minimum useful change.

### Larger change — a per-example wiki page

A `wiki/examples/_index.md` plus one page per scenario:

- `wiki/examples/basic-crud.md`
- `wiki/examples/inheritance.md`
- `wiki/examples/relations.md`

Each page would be 5–10 lines: what the scenario demonstrates, which wiki concepts/components it exercises, and a deep link into the relevant `examples/<scenario>/` files. This treats `examples/` the same way `wiki/sources/` treats `.raw/` — every external artifact gets a wiki front-door.

I lean toward the larger change because the inheritance example is load-bearing enough (it's the only place `InheritanceSearchType` is shown in use) that it deserves a dedicated wiki node, not just a brief mention.

## Wiki pages affected

| Page | Action |
|---|---|
| `wiki/brief.md` | Add a bullet to "Where to dig deeper": *"Working call sites — `examples/basic-crud/`, `examples/inheritance/`, `examples/relations/`. Each is self-contained (entities + service + db wiring)."* |
| `wiki/index.md` | Add an "Examples" section between "Sources" and "Entities", or at the end. Either link directly into the `examples/` paths (relative to repo root) or link to the new `wiki/examples/` pages if going with the larger change. |
| `wiki/examples/_index.md` (new, optional) | Index for the per-example pages. Same shape as the other `_index.md` files. |
| `wiki/examples/basic-crud.md` (new, optional) | One-paragraph description: which decorators it exercises (`@Entity`, `@Column`, `@PrimaryColumn`, `@NotNullable`), which Repository methods it demonstrates (`create`, `findMany`, `findById`), and a "see also" linking `[[Repository Pattern]]`, `[[Autogeneration]]`, `[[lifecycle-of-a-create]]`. |
| `wiki/examples/inheritance.md` (new, optional) | The most important new page. Should explicitly call out *this is the only place `InheritanceSearchType` is shown in use*. Cross-link `[[Single-Table Inheritance]]`, `[[QueryBuilder]]`, and the new `inheritance` option section that `[[drift-d3-find-options-inheritance]]` proposes. |
| `wiki/examples/relations.md` (new, optional) | One-paragraph description; cross-link `[[Relation Target Thunk]]` and `[[modules/sql-types]]` (for the FK type demotion). |
| `wiki/concepts/Single-Table Inheritance.md` | Add an "Examples" footnote: *"See `examples/inheritance/` for a runnable demonstration. The service file (`services/UserHierarchyService.ts`) is the canonical use of the `inheritance` `FindOptions` option."* |
| `wiki/components/Repository.md`, `wiki/components/QueryBuilder.md`, `wiki/concepts/Repository Pattern.md` | Where each currently embeds an inline code sample, consider trimming the sample and adding *"See `examples/basic-crud/services/UserService.ts` for the full service-level call site."* |
| `wiki/concepts/Repository Pattern.md` line ~67 | Already says *"Current `examples/` style — direct construction, no DI yet."* — that's the only existing reference, and it's vague. Tighten to *"See `examples/basic-crud/services/UserService.ts` for the canonical no-DI construction pattern."* |

## Source citations

- `examples/basic-crud/{db.ts, index.ts, entities/User.ts, services/UserService.ts}` — the basic-crud scenario.
- `examples/inheritance/{db.ts, index.ts, entities/{User,AdminUser}.ts, services/UserHierarchyService.ts}` — the inheritance scenario, especially `UserHierarchyService.listSubClassUsers()` / `listSubClassAdmins()` for the `InheritanceSearchType.SUBCLASSES` use.
- `examples/relations/` — not opened in this drift pass; the wiki agent should read its files before writing the corresponding wiki page.
