# Drift Correction — module pages still empty (`wiki/modules/`)

`wiki/index.md` says: *"Module pages not yet filed; the `[[modules/_index]]` holds the source-tree map (8 directories cataloged with their concerns and layer assignments)."*

This is acknowledged in the wiki itself, so it's not strictly a *drift* — it's an open backlog item. But the source tree is small enough today that filling it in is cheap, and several of the existing component pages already overlap with what these pages would say. Worth deciding whether to close the gap or formally close the bucket.

## Current state of `src/`

The eight directories the wiki refers to are:

```
src/
├── core/
│   ├── database/        — connection + metadata host + schema lifecycle (layer 5 + 2 + adjacent)
│   ├── repository/      — by-key persistence (layer 3)
│   ├── metadata/        — MetadataStorage class (layer 2)
│   ├── sql-types/       — COLUMN_TYPE enum + DDL fragment helpers + FK-type demotion
│   ├── utils/           — sqlJoin + Constructor type helpers
│   └── orm-error/       — abstract OrmError base class
├── decorators/
│   ├── entity/          — @Entity (class decorator)
│   ├── column/          — @Column / @PrimaryColumn (field decorators)
│   ├── nullable/        — @Nullable / @NotNullable (field decorators) + NullableField/NotNullableField type constraints
│   └── relation/        — @ToOne (field decorator). RelationType.TO_MANY exists in the enum but no @ToMany decorator.
└── query-builder/       — QueryBuilder class + Conditions/Condition/FindOptions types + InheritanceSearchType enum (layer 4)
```

(That's actually 11 leaf directories under `src/`, not 8. The "8 directories" claim in the wiki is itself slightly stale.)

## Two paths the wiki agent could take

### Path A — Close the backlog by filing the module pages

For each leaf directory above, create `wiki/modules/<name>.md` with a uniform shape:

- **Module name** (matches directory name).
- **Layer** (per `[[Layered Architecture]]`).
- **Exports** (the public symbols from that directory).
- **Owns** (one-sentence purpose).
- **Reads from / writes to** (downward dependencies — should match the layer rule).
- **See also** (the relevant component / concept / flow page).

Most of these would be very short — three-line stubs are fine. The value is structural: a uniform per-module page set means an agent can navigate from "I'm editing `src/decorators/relation/`" → `wiki/modules/relation.md` → the relevant deeper pages.

### Path B — Close the bucket and delete `wiki/modules/`

If the per-module shape isn't carrying weight that `[[Layered Architecture]]` and the component pages don't already carry, retire `wiki/modules/`. Replace the index entry with a sentence: *"There are no per-module pages; module-level concerns are covered by `[[Layered Architecture]]` (boundaries) and the per-component / per-concept pages (substance)."*

I lean toward **Path A** for the directories that don't already have a component page — specifically `src/core/sql-types/`, `src/core/utils/`, `src/core/orm-error/`, and `src/decorators/nullable/`. The others (`database/`, `repository/`, `metadata/`, `query-builder/`) already have full component pages and a bare module stub would be redundant — those can either be skipped or filed as ~3-line redirects to the component page.

## Concrete proposed page list

If Path A:

| Module path | Suggested wiki page | Already covered by? |
|---|---|---|
| `src/core/database/` | `wiki/modules/database.md` (stub → `[[Database]]`) | `[[Database]]` component page |
| `src/core/repository/` | `wiki/modules/repository.md` (stub → `[[Repository]]`) | `[[Repository]]` component page |
| `src/core/metadata/` | `wiki/modules/metadata.md` (stub → `[[MetadataStorage]]`) | `[[MetadataStorage]]` component page |
| `src/core/sql-types/` | `wiki/modules/sql-types.md` (full page) | **No existing coverage** — see `[[drift-m5-sql-types-module]]` |
| `src/core/utils/` | `wiki/modules/utils.md` (full page) | `[[sqlJoin]]` covers half; `Constructor` type helpers uncovered |
| `src/core/orm-error/` | `wiki/modules/orm-error.md` (full page, short) | **No existing coverage** |
| `src/decorators/entity/` | `wiki/modules/decorators-entity.md` (stub) | `[[entity-registration]]` flow + `[[ECMAScript Stage-3 Decorators]]` |
| `src/decorators/column/` | `wiki/modules/decorators-column.md` (stub) | `[[Autogeneration]]` |
| `src/decorators/nullable/` | `wiki/modules/decorators-nullable.md` (full page, short) | Mentioned in `[[Repository]]` and `[[Repository Pattern]]` but no dedicated page; the `NullableField` / `NotNullableField` type-level mechanic is load-bearing for the `create(entity: T)` contract |
| `src/decorators/relation/` | `wiki/modules/decorators-relation.md` (full page) | `[[Relation Target Thunk]]` covers the thunk pattern; the actual decorator surface (`@ToOne(options)` and the unused `OneToOneOptions.foreignKeys`) isn't documented anywhere |
| `src/query-builder/` | `wiki/modules/query-builder.md` (stub → `[[QueryBuilder]]`) | `[[QueryBuilder]]` component page |

Three of these — `sql-types`, `orm-error`, `decorators-relation` — are also flagged in other drift notes (`[[drift-m5-sql-types-module]]`, etc.), so closing them is high-leverage.

## Wiki pages affected

| Page | Action |
|---|---|
| `wiki/index.md` | Update the "Modules" section either way. If Path A: list the new pages. If Path B: replace with the "no per-module pages" sentence. Also fix the "8 directories" → "11 directories" miscount. |
| `wiki/modules/_index.md` | Either populate with the file list or repurpose into the Path B explanation. |
| `wiki/brief.md` | The brief currently doesn't cite module pages, so likely no change needed. If Path A and the per-module stubs are useful enough, consider adding one bullet to the brief's "Where to dig deeper" table: *"per-module surface … `[[modules/_index]]`."* |
| `wiki/hot.md` | Currently says *"Suggested next action: lint the wiki — run `wiki-lint` to surface orphans, dead wikilinks, frontmatter gaps, and stale claims now that the bulk ingest is complete."* — closing this backlog item is a candidate for the next "active thread." |

## Source citations

- `src/` directory layout: see `find src -type d` output (in `examples/`-equivalent style this is `core/{database, repository, metadata, sql-types, utils, orm-error}` + `decorators/{entity, column, nullable, relation}` + `query-builder/`).
- `wiki/index.md` line 38: *"Module pages not yet filed; the `[[modules/_index]]` holds the source-tree map (8 directories cataloged with their concerns and layer assignments)."*
- `wiki/modules/_index.md` — the source-tree map already mentioned (not re-read in this drift pass; verify before editing).
