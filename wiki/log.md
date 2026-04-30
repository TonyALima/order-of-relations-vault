---
type: meta
title: "Operation Log"
created: 2026-04-29
updated: 2026-04-30
tags:
  - meta
  - log
status: evergreen
related: []
sources: []
---

# Operation Log

Append-only record of every wiki operation. Newest entries on top. Never edit past entries — file new corrections instead.

---

## 2026-04-30 — lint | Health check after PK-aware-repository-methods ingest

**Report:** [[lint-report-2026-04-30]].

**Headline: ✅ wiki health green.** All three new pages from the PK-brand ingest carry complete frontmatter and ≥7 inbound wikilinks each. Zero orphans, zero stale claims, zero unresolved contradictions.

**Findings (after filtering 33 false positives — backtick-wrapped format docs, Obsidian Bases file references, escaped-pipe table cells, prior lint report contents):**

- **1 genuine dead wikilink** — `[[idx-discriminator-collision]]` in `wiki/sources/drift-d5-discriminator-index.md:85`, framed in-line as *"Optional future open question"*. Author-intentional placeholder; pre-existing; not fixed.
- **1 minor frontmatter drift** — `wiki/meta/_index.md` had `page_count: 0` despite a lint report existing. **Fixed:** bumped to 2 (counts both lint reports) and added entries for both reports plus the `[[issues|Issue tracker]]` Base.

**Files touched:**

- `wiki/meta/lint-report-2026-04-30.md` — new.
- `wiki/meta/_index.md` — frontmatter `page_count: 0 → 2`; `updated: 2026-04-29 → 2026-04-30`; body now lists both lint reports + the issues Base.
- `wiki/log.md` — this entry.

**Suggested follow-up (deferred, not done in this lint):**

- Optional: file `wiki/questions/idx-discriminator-collision.md` to upgrade the placeholder hint into a tracked open question.
- Optional: revisit `wiki/comparisons/orms-summary.md` to surface "compile-time PK enforcement via runtime-erased brand" as a distinctive contribution row. Tracked as an active thread in `wiki/hot.md`.

---

## 2026-04-30 — ingest | PK-aware Repository methods

**Source:** `.raw/pk-aware-repository-methods.md` (post-implementation design memo, 250 lines).

**Headline:** the compile-time PK-enforcement direction that `repository-contract` started for `create()` is now finished for `findById` / `delete` / `update`. A new structural brand `PrimaryKey<V>` lives at the field-declaration site; four Repository signatures change to consume it; the silent `update({ name: 'x' })` bug on autogen entities is closed at the type level.

**Pages created:**

- [[sources/pk-aware-repository-methods]] — source synthesis with the full alternatives analysis (A: brand on field type *(chosen)*, B: generic `Repository<T, PK>`, C: runtime-only).
- [[PrimaryKey Brand]] — concept page. Brand asymmetry (`PrimaryKey<V>` is a subtype of `V`, so branded → unbranded works freely; the reverse needs a cast) is the load-bearing ergonomic trick that keeps call sites brand-free.
- [[0008-pk-aware-compile-time]] — first post-rollout ADR. Alternatives B and C documented for posterity; B's footgun (runtime/compile-time disagreement of `PK` generic vs `@PrimaryColumn` metadata) is the disqualifying argument for a publishable library.

**Pages updated:**

- [[Repository]] — Operations table now shows `findById(key: PKInput<T>)` / `delete(key: PKInput<T>)` / `update(entity: UnbrandedT<T> & PKInput<T>)` / `create(entity: UnbrandedT<T>): Promise<PKOutput<T>>`. Runtime section: `create()` returns `PKOutput<T>` (branded), not `Partial<T>`. `requirePrimaryKey` parameter widened to `PKInput<T> | UnbrandedT<T>`.
- [[Repository Pattern]] — fourth refinement note added; `create()` example comment updated; `@PrimaryColumn` § now mentions both overloads carry the brand term.
- [[Autogeneration]] — silent-`update({ name: 'x' })` bug now noted as closed by the brand work; example entities updated to declare `id?: PrimaryKey<number>` / `id?: PrimaryKey<string>` / `externalId!: PrimaryKey<string>`.
- [[Conditions Proxy]] — proxy type changed from `FieldConditionBuilder<T[K]>` to `FieldConditionBuilder<Unbrand<T[K]>>`; refinement note added explaining why `c.id?.eq(1)` accepts a plain literal.
- [[ECMAScript Stage-3 Decorators]] — new § "The constraint-flip pattern (read-only)" generalizing the pattern that now governs both nullability and the brand.
- [[index]] — `page_count: 58 → 61`; new entries in Decisions, Concepts, Sources sections.
- [[concepts/_index]] / [[decisions/_index]] / [[sources/_index]] — counts and listings refreshed.
- [[hot]] — refreshed below.
- [[log]] — this entry.

**Manifest:** `.raw/.manifest.json` updated with the new source's hash and pages_created/pages_updated arrays.

**Key insight:** the `PrimaryKey<V>` brand had to live at the *declaration site* — Stage-3 decorators can READ a field's declared type but cannot inject type information into it. Option B (a `PK` generic on `Repository<T, PK>`) was disqualified because the declaration-site decorator and the construction-site generic could disagree silently. Putting the brand on the field unifies the two: `@PrimaryColumn`'s overload constraint and the field type are checked against each other at the same call.

---

## 2026-04-30 — correction | Remove `support-date-operators` open question

**Reverts part of the previous entry.** [[support-date-operators]] (filed earlier today as *medium / M*) is removed. Owner's call: the existing nine-method `FieldConditionBuilder` surface — `eq`, `ne`, `gt`, `gte`, `lt`, `lte`, `isNull`, `isNotNull`, `in` — already supports date columns since `T[K] = Date` flows through the value parameter. Date queries like `u.createdAt!.gt(new Date(...))` work today. Anything that *doesn't* work is a bug, not an open question, and will be tracked through code tests rather than a wiki issue.

**Convention reinforced:** open questions are for genuine design decisions, not for missing-but-trivially-derivable functionality. The "future API could narrow further" hint in [[Conditions Proxy]] remains a hint, not a tracked issue, until a concrete design proposal makes it one.

**Files touched:**

- `wiki/questions/support-date-operators.md` — deleted.
- `wiki/questions/_index.md` — `page_count: 8 → 7`, removed from open-questions table.
- `wiki/index.md` — `page_count: 59 → 58`, removed from Questions list.
- `wiki/components/QueryBuilder.md` § Open Questions — entry removed; [[support-and-or-conditions]] remains.
- `wiki/concepts/Conditions Proxy.md` § Open Questions — entry removed; [[support-and-or-conditions]] remains.
- `wiki/hot.md` — refreshed.

The previous log entry's wikilink to [[support-date-operators]] is now dead but preserved as audit trail (per `log.md` convention: "Dead wikilinks inside past log entries are preserved as audit trail and are not lint issues").

---

## 2026-04-30 — questions | Add two open questions on the query builder

**Filed:**

- [[support-and-or-conditions]] — *high / L* — extend the `where` callback from a flat AND list to a real boolean tree (AND / OR / NOT, nested groups). Anchored in [[QueryBuilder]] § Scope's explicit deferral. Three design options sketched (`Or`/`And` combinators à la TypeORM/Sequelize; recursive object tree à la Prisma; method-chaining `.orWhere()` à la TypeORM legacy); change-surface for the combinator option spelled out, including the [[Single-Table Inheritance|STI]] discriminator wrapping rule that breaks once the array becomes a tree. Effort is L because `Condition` becomes a discriminated union and SQL emission becomes recursive with parenthesisation.

- [[support-date-operators]] — *medium / M* — narrow `FieldConditionBuilder<V>` for `V extends Date` with `between` / `before` / `after` / `eqDay` / `inMonth` / `inYear`. Anchored in [[Conditions Proxy]]'s "future API could narrow further" hint — date is the highest-value candidate to test the per-column-type narrowing pattern. Three options: type-conditional method surface (with sub-variants A1 = attach all methods at runtime, A2 = attach by PG type from `MetadataStorage`), standalone `DateOps` namespace, or `op` enum extension only (`between` for everyone). Effort is M because the typing infrastructure (conditional types over `V`) is novel for this codebase but localised to one module.

**Why both at once:** they sit on different layers but both touch the `Condition` shape — AND/OR refactors it into a discriminated union (`leaf` / `and` / `or` / `not`); date operators add tuple-valued leaves (`between`'s value is `[V, V]`). Co-design or co-implementation is the natural path.

**Cross-references added:**

- [[QueryBuilder]] § Open Questions — both new questions added alongside [[get-one-limit-1]] and [[apply-options-accumulation]].
- [[Conditions Proxy]] — new § Open Questions section, since the page's "uniform across V" framing is exactly what date narrowing would change, and the `(Condition | undefined)[]` callback shape is what AND/OR would extend.

**Files touched:**

- `wiki/questions/support-and-or-conditions.md` — new.
- `wiki/questions/support-date-operators.md` — new.
- `wiki/questions/_index.md` — `page_count: 6 → 8`, both questions inserted in the open-questions table at their priority slots (high/L next to one-to-many; medium/M next to user-indexes).
- `wiki/index.md` — `page_count: 57 → 59`, both questions added to the Questions list.
- `wiki/components/QueryBuilder.md` — Open Questions section now lists four entries.
- `wiki/concepts/Conditions Proxy.md` — added Open Questions section.
- `wiki/hot.md` — refreshed.

---

## 2026-04-30 — file | Add `orms-summary` feature matrix

**Filed:** [[orms-summary]] — single-page feature matrix comparing OOR's contributions against TypeORM, Prisma, and Drizzle. ~30 rows grouped under five sections: Foundation, API shape, SQL safety, Modeling, Toolchain.

**Cell vocabulary:** ✅ delivered, ⏳ planned and well-scoped (equal-value rule), 🟡 partial / not first-class, ❌ not delivered, — not applicable (different mechanism). The `—` marker is load-bearing: writing `Stage-3 decorators: Prisma ❌` would be misleading because Prisma doesn't use decorators on classes at all — the question doesn't translate.

**How it relates to the four detailed pages:** the matrix is the at-a-glance scoreboard; [[oor-vs-typeorm]], [[oor-vs-drizzle]], [[oor-vs-prisma]], [[stage-3-vs-legacy-decorators]] hold the long-form thesis behind each row. The summary page closes with a "How to read this matrix" section that names four row-patterns:

1. OOR ✅ and all competitors ✅ — table-stakes the field has converged on.
2. OOR ✅ + TypeORM ✅ but Prisma/Drizzle ❌ — the OO ergonomic profile.
3. OOR ✅ + Prisma/Drizzle ✅ but TypeORM ❌ — modern guarantees TypeORM is locked out of.
4. OOR ✅ alone — distinctive contributions (no `unsafe` SQL escape hatch in the public API or library internals; auto-emitted STI discriminator index; FK demotion at the type level; compile-time rejection of partial entities on `create()`).

**A "Notes on selected cells" section** disambiguates rows where the binary is misleading — e.g., TypeORM scores 🟡 (not ✅) on `Repository<T>` per entity because while the class exists, it's not the *single* entry-point; Prisma scores ✅ on "one mental model" because there's only one query shape, not because it solves the simple-vs-composed split the same way.

**Files touched:**

- `wiki/comparisons/orms-summary.md` — new.
- `wiki/comparisons/_index.md` — `page_count: 4 → 5`, added "Summary matrix" subsection at the top of the comparison pages list.
- `wiki/index.md` — `page_count: 56 → 57`, added the new page to the comparisons list.
- `wiki/hot.md` — refreshed.

---

## 2026-04-30 — refactor | Comparisons reframed as TCC defense material

**What changed:** all four comparison pages ([[oor-vs-typeorm]], [[oor-vs-drizzle]], [[oor-vs-prisma]], [[stage-3-vs-legacy-decorators]]) rewritten with a TCC-defense frame instead of a critical/pragmatic-consumer frame.

**Reframing principles applied:**

1. **Question being answered changed.** Before: "Should I pick OOR or X for my project?" After: "What does OOR contribute that the field didn't already have? Why does it matter in a crowded market?"
2. **Equal-value rule for planned features.** A feature with a fully-scoped open-question page (axes spelled out, decorator surface defined, change-surface described) is now listed as part of OOR's contribution rather than as a "where OOR is behind" gap. Examples promoted to feature-equal: `@Index` / `@Unique` ([[support-user-indexes]]), decorator-order independence ([[decorator-order-independence]]). Seed-only concepts ([[Schema Migrations]]) are not claimed.
3. **Sections removed:**
   - `## When to reach for which` — pragmatic consumer-decision aid that conceded use-cases. Removed from all three ORM-vs-ORM pages.
   - `## Where OOR is behind (by feature, not by design)` — gap callouts that worked against the TCC's "we're selling an idea" frame. Removed from all four pages.
4. **Sections added:**
   - `## What OOR brings that's new` — bullet-list contribution summary up top, replaces the buried "design contribution" framing.
   - `## OOR's contribution, dimension by dimension` — replaces "where they diverge — by design"; same prose backbone but framed as positive contributions, not neutral observations.
   - `## Why OOR matters in a crowded market` — closing argument; replaces the prior `## Verdict` (verdict still in frontmatter).
5. **Tone shift:** from "honest about gaps" to "confident about the design." The accuracy of facts in the dimensions tables is unchanged — what changed is the framing of OOR's positions (e.g., PG-only is now "PostgreSQL, deeply" with a contribution case, not a "narrow scope" concession).
6. **Stage-3 page got a custom reframe.** It's not OOR vs an ORM but the dialect choice OOR's already made. Recast as: "Stage-3 is the bet, and here's why OOR is well-positioned to make it." `## Where legacy wins` renamed to `## What OOR accepts as the cost` — same content, repositioned as known/accepted tradeoffs that match OOR's design rather than defects in the dialect choice.

**Why this matters for the TCC:** the wiki is meant to be the long-form receipts for the thesis defense. Pages that read as neutral comparisons concede the academic argument before it begins. The reframed pages are receipts for the contribution, not receipts for a decision tree.

**Files touched:**

- `wiki/comparisons/oor-vs-typeorm.md` — rewritten end-to-end.
- `wiki/comparisons/oor-vs-drizzle.md` — rewritten end-to-end.
- `wiki/comparisons/oor-vs-prisma.md` — rewritten end-to-end.
- `wiki/comparisons/stage-3-vs-legacy-decorators.md` — rewritten end-to-end.
- `wiki/comparisons/_index.md` — updated convention notes to remove "where OOR is behind" pattern, add equal-value rule, add TCC-frame guidance for future comparisons.
- `wiki/hot.md` — refreshed.

**Convention now established for `wiki/comparisons/`:** every page leads with `## What OOR brings that's new` (bullets), then frontmatter `verdict:`, then dimensions table, then `## OOR's contribution, dimension by dimension` prose, then `## Why OOR matters in a crowded market` closing argument. No consumer-decision sections, no gap callouts.

---

## 2026-04-29 — edit | Drop "Maturity" row from all comparison tables

**What changed:** removed the **Maturity** row from the dimensions tables in [[oor-vs-typeorm]], [[oor-vs-drizzle]], [[oor-vs-prisma]]. The row compared "TCC pre-1.0" against well-funded multi-year projects with tens of thousands of GitHub stars — a dimension where the comparison is structurally meaningless and concedes ground without illuminating anything. Frontmatter `dimensions:` arrays updated to drop `maturity` from each.

**Kept as standing convention:** comparison-table dimensions should be axes where the libraries can be meaningfully compared on architecture or features. Project-maturity metrics belong in entity pages (TypeORM, Prisma, Drizzle), not in OOR-vs-X tables.

---

## 2026-04-29 — file | Build out `wiki/comparisons/` (4 new pages)

**Filed:** four comparison pages closing the "no comparisons ingested yet" gap on [[comparisons/_index]].

- [[oor-vs-typeorm]] — TypeORM v0.3.28 (2025-12). Same shape (decorators, `Repository<T>`, lazy builder, STI), broader scope (12 DBs vs PG-only), weaker safety guarantees (raw SQL fragments are the primary `where()` API; no `sql.unsafe` ban). Receipts for [[0002-repository-with-lazy-query-builder]]'s rejection of "TypeORM-style `Repository.createQueryBuilder()`."
- [[oor-vs-drizzle]] — Drizzle 1.0-beta.22 (2026-04-16). Opposite philosophy: typed SQL composer (`pgTable` + `db.select().from(...).where(eq(...))`), no Repository abstraction, no native STI. Receipts for ADR 0002's rejection of "no repositories, raw query builder only."
- [[oor-vs-prisma]] — Prisma v7.8.0 (Apr 2026, post-Rust-free architecture). Different category — schema-DSL + codegen vs class + runtime metadata. Bundle/runtime objections are now stale; the live wedge is source-of-truth (PSL file + `prisma generate` vs decorated TS class).
- [[stage-3-vs-legacy-decorators]] — dialect deep-dive behind [[0001-stage-3-decorators]]. Stage-3 wins on standardization, polyfill-freedom, toolchain alignment; legacy wins on `design:type` runtime info and parameter decorators (Stage-3 still doesn't have them — the NestJS/Inversify migration blocker).

**Research method:** four parallel agents fetched current upstream docs (typeorm.io, orm.drizzle.team, prisma.io, tc39 proposal-decorators, TypeScript 5.0 release notes) and produced factual briefs grouped by 10-11 dimensions each. Each comparison page then maps those facts to OOR's positioning, with `> [!gap]` callouts where OOR is *behind by feature, not by design* (indexes, migrations, edge runtime support).

**Why the structure:** every comparison follows the same skeleton — frontmatter `verdict` line, dimensions table (~12 rows), "where they converge / diverge — by design" prose, "where OOR is behind (by feature, not by design)" gap callouts cross-linking the relevant open questions ([[support-user-indexes]], [[Schema Migrations]], [[support-one-to-many]] / [[support-many-to-many]] where applicable), "when to reach for which" pragmatic split, one-paragraph verdict, sources.

**Cross-links added:** all four ADRs referenced ([[0001-stage-3-decorators]], [[0002-repository-with-lazy-query-builder]], [[0004-parameterized-sql-only]], [[0005-no-any-type-driven-api]]) — comparisons are readable as receipts for those decisions.

**Files touched:**

- `wiki/comparisons/oor-vs-typeorm.md` — new.
- `wiki/comparisons/oor-vs-drizzle.md` — new.
- `wiki/comparisons/oor-vs-prisma.md` — new.
- `wiki/comparisons/stage-3-vs-legacy-decorators.md` — new.
- `wiki/comparisons/_index.md` — `page_count: 0 → 4`, `status: seed → developing`, listed all four pages with one-line hooks, added "Patterns to follow when adding a new comparison" section.
- `wiki/index.md` — replaced "_none yet_" with the four new entries, bumped `page_count: 52 → 56`.
- `wiki/hot.md` — refreshed.

**Suggested next action:** `lint the wiki` — four new pages with many wikilinks; orphans / dead links / frontmatter gaps should be checked. Also worth verifying that the `Drizzle ORM` and `Prisma` and `TypeORM` entity pages exist (the comparisons link to them as `[[Drizzle ORM]]` / `[[Prisma]]` / `[[TypeORM]]`); if any are missing, those wikilinks will surface in the lint report and the entity pages should be filed.

---

## 2026-04-29 — convention | Add `.inbox/` for cross-context idea capture

**Filed:** new `.inbox/` folder + `.inbox/.processed/` archive + `.inbox/README.md`. Documented in `CLAUDE.md` under Conventions and Operations.

**Use case:** when an idea surfaces while working in a context that doesn't have vault skills/rules loaded — typically a code-agent session in `../order-of-relations`, a vault-unaware chat, or a quick capture from another repo — drop a free-form markdown note into `.inbox/`. Triage happens later in a vault-aware session, which decides per-note whether it becomes an open question, drift correction, ADR seed, direct page edit, or gets discarded.

**Why a new folder vs reusing `.raw/`:** `.raw/` is for shaped, authoritative source documents that ingest synthesizes from. Inbox notes are unshaped, provisional, and may be wrong — treating them as `.raw/` content would (a) violate the "never modify" rule when notes are deleted post-triage, (b) produce a redundant `wiki/sources/X.md` synthesis page per note, and (c) treat half-formed thoughts as authoritative. Different contract → different folder.

**Lifecycle (recorded for future-self):**

1. Drop file → `.inbox/<slug>.md` (any agent / context can do this; no schema required).
2. Triage → in a vault-aware session: `triage my inbox` or `process .inbox/<file>`.
3. Resolve → Claude files into the right wiki location, then moves original to `.inbox/.processed/<slug>.md` (kept as audit trail, not deleted).

**Decision: keep committed**, not gitignored. Cost is rounding error; benefit is durable inbox across machines and the `.processed/` archive becomes a retrospective ("what ideas have I had over this project's lifetime?").

**Files touched:**

- `.inbox/.gitkeep`, `.inbox/.processed/.gitkeep` — directory placeholders.
- `.inbox/README.md` — full convention doc, readable by any agent that drops a file there.
- `CLAUDE.md` — `.inbox/` added to the structure tree; new Conventions bullet; new Operations bullet (`triage inbox`); new "Ingest vs triage" decision table to disambiguate.
- `wiki/hot.md` — note the new convention.

**Why a decision table in CLAUDE.md:** the failure mode I'm guarding against is "future-me drops an idea into `.raw/` because it feels source-shaped, ingest synthesizes it, lint flags the thin synthesis." Three rows of `| Source | Folder | Operation |` makes the right choice obvious at glance — much cheaper than re-deriving the rule each time.

---

## 2026-04-29 — file | New open question: support user-defined indexes

**Filed:** [[support-user-indexes]] — add `@Index` / `@Unique` decorators so users can declare indexes on columns and column-tuples; `schema-create` should emit `CREATE INDEX` alongside `CREATE TABLE`.

**Grounded in current state** (verified 2026-04-29 against `../order-of-relations`):

- `src/decorators/` contains only `column/`, `entity/`, `nullable/`, `relation/` — no `@Index` or `@Unique` decorator exists.
- The only index OOR emits today is the implicit `idx_discriminator` for STI roots (per [[sources/drift-d5-discriminator-index]]).
- The schema-emission pattern is already proven; only the metadata source is missing.

**Initial triage:** `impact: medium`, `effort: M`, `score: 6`. Sits between [[decorator-order-independence]] (7) and the two `low / S` query-builder deferrals (5).

**Why medium not high:** workarounds exist (out-of-band `.sql` migrations); will climb to high once OOR has real users running non-trivial table sizes.

**Why M not S:** new decorator(s), new metadata key, schema-create wiring, naming policy, decisions across three independent design axes (decorator location, `@Unique` shape, naming).

**Cross-links recorded:**

- Constrains [[decorator-order-independence]] — confirms `@Index` is a real (not hypothetical) future sibling decorator. Reinforces the case for Option A.
- Folds in the unfiled `idx_discriminator` collision question from [[hot]] — a naming policy applies uniformly to discriminator and user indexes.

**Files touched:**

- `wiki/questions/support-user-indexes.md` — new question page.
- `wiki/questions/_index.md` — added to the impact/effort table.
- `wiki/index.md` — Questions section updated.
- `wiki/hot.md` — Open Questions section updated; `idx_discriminator` note re-pointed at this question.

---

## 2026-04-29 — refactor | Turn `wiki/questions/` into an issue tracker

**Decision:** keep open questions in `wiki/questions/` (no new `issues/` folder) and extend each page with `impact` + `effort` + `decided_by` frontmatter. Add a Bases view at `wiki/meta/issues.base` for sorting and filtering.

**Schema added** (per open question):

- `impact: low | medium | high` — cost of leaving the issue open.
- `effort: S | M | L` — rough implementation size.
- `decided_by: ""` — wikilink to the ADR that closes it; empty until decided.

**Lifecycle convention** (recorded for future-self): in this vault, `status: open` implies a commitment to implement once decided. There is no "decided but not implemented" state — when a question closes, it closes because the change shipped. Status moves `open → answered`; nothing in between.

**Score formula** (in `issues.base`): `impact × 2 + effort_inverse`, where `high=3, medium=2, low=1` and `S=3, M=2, L=1`. Range 3–9; higher is better. Weighting impact 2× ensures high-impact + L effort outranks low-impact + S effort.

**Initial assignments:**

- [[decorator-order-independence]] — medium / S — score 7. Footgun scales as more sibling decorators (`@Index`, `@Unique`, `@Default`) get added; Option A is the smallest fix.
- [[get-one-limit-1]] — low / S — score 5. Source explicitly defers; latent dependency on `orderBy` for determinism.
- [[apply-options-accumulation]] — low / S — score 5. No current double-`applyOptions` site; revisit when scopes appear.

**Files touched:**

- `wiki/questions/decorator-order-independence.md` — frontmatter (impact / effort / decided_by).
- `wiki/questions/get-one-limit-1.md` — frontmatter.
- `wiki/questions/apply-options-accumulation.md` — frontmatter.
- `wiki/questions/_index.md` — issue-tracker preamble + impact/effort table.
- `wiki/meta/issues.base` — new Bases view (3 views: by-score table, by-impact grouped table, triage cards).
- `wiki/index.md` — Meta section now points at [[issues]]; Questions section shows impact/effort inline.
- `wiki/hot.md` — refreshed.

**Why this shape vs alternatives:**

- *Separate `issues/` folder, rejected* — the question pages already contain problem statement, why-it-matters, design space, change surface, and what-would-close-this. They're issues already; duplicating into a new folder splits the source of truth.
- *Manual ranked list in `_index.md`, rejected* — loses live sort and forces re-ranking by hand on every change.
- *Dataview instead of Bases, rejected* — adds a community-plugin dependency; Bases is native and sufficient for a 3-row tracker.

---

## 2026-04-29 — ingest | Drift correction batch (D1, D3, D4, D5, M3, M5, M6)

**Sources:** seven drift-correction notes filed against the wiki after a code-vs-wiki audit.

- `.raw/drift-d1-repository-find.md` (md5 `656f720d3d1cd69fd6663827c3302a06`) — `Repository.find()` does not exist
- `.raw/drift-d3-find-options-inheritance.md` (md5 `f4cdb91af2862133474d5de6f90fde8f`) — `FindOptions.inheritance` is undocumented
- `.raw/drift-d4-conditions-proxy-operators.md` (md5 `0b00203e2e32355276f1bcd846e2097f`) — `FieldConditionBuilder` operator inventory wrong
- `.raw/drift-d5-discriminator-index.md` (md5 `d4ded058e75d18f8f9957cf45e4916f9`) — schema-create emits implicit `idx_discriminator`
- `.raw/drift-m3-module-pages.md` (md5 `9490370c861910f589b0b44682c6cc96`) — `wiki/modules/` empty + miscount (8 vs 11 leaves)
- `.raw/drift-m5-sql-types-module.md` (md5 `2648cd5e5f360e7aa158d12c74433460`) — `src/core/sql-types/` undocumented
- `.raw/drift-m6-examples-pointer.md` (md5 `f65ab8569b84f5a5b55921bebe6e6fa9`) — `examples/` invisible to wiki

**Pages created (12):**
- Source syntheses: [[sources/drift-d1-repository-find]], [[sources/drift-d3-find-options-inheritance]], [[sources/drift-d4-conditions-proxy-operators]], [[sources/drift-d5-discriminator-index]], [[sources/drift-m3-module-pages]], [[sources/drift-m5-sql-types-module]], [[sources/drift-m6-examples-pointer]]
- Module: [[modules/sql-types]]
- Examples: [[examples/_index]], [[examples/basic-crud]], [[examples/inheritance]], [[examples/relations]]

**Pages updated (13):**
- [[brief]] — removed `find()` claim; added `inheritance` `FindOptions` bullet; added `COLUMN_TYPE` / FK-demotion bullet; added `examples/` row to "Where to dig deeper".
- [[Repository]] — operations table reduced to six methods; `find() is a handoff` section rewritten as "How reads compose."
- [[Repository Pattern]] — by-key vs. composed split rewritten without `find()`; example trimmed; refinement note added (×4 entry).
- [[Lazy Query Builder]] — definition rewritten ("constructed inside `findMany`/`findOne`"); inheritance forward-link added to state-field paragraph; bottom paragraph rewritten with drift-D1 note.
- [[QueryBuilder]] — `applyOptions()` section expanded with `inheritance` subsection (three values + footgun callout + key-insight on discriminator-only-when-needed).
- [[Conditions Proxy]] — operator inventory replaced with the nine-method list; "Type-safe operators" replaced with "Type-safe values, uniform operator surface"; `before/after` example replaced with `gt`; drift-D4 refinement note added.
- [[Single-Table Inheritance]] — added "Reading Across the Hierarchy" section; added "What Schema-Create Emits" section.
- [[query-lifecycle]] — added Step 4.5 documenting inheritance-condition push.
- [[MetadataStorage]] — softened "indexes deliberately absent" with the schema-create exception; resolveRelations description expanded to mention `toForeignKeyType` demotion.
- [[schema-create-drop]] — Pass 1 expanded with the two skip rules and the STI emissions trio; latent-collision warning callout added.
- [[Database]] — Schema lifecycle bullet 1 expanded with STI emissions.
- [[Autogeneration]] — added "Why `SERIAL` for `dbSide` PKs" callout cross-linking [[modules/sql-types]].
- [[Relation Target Thunk]] — "When the Thunk Is Called" step 3 expanded with `toForeignKeyType` demotion clause.
- [[modules/_index]] — page_count 0 → 1; reframed (no longer "none yet"); added Deferred table.

**Key insights:**
- Audit established a useful pattern: `.raw/drift-*` notes preserve pre-correction claims, post-correction claims, and code citations; the synthesis pages serve as the audit trail and survive re-ingest because hashes are recorded.
- The single biggest documentation gap (D3 — `FindOptions.inheritance`) is now closed; [[examples/inheritance]] front-doors the canonical use site.
- The `find()` framing was load-bearing in agent-facing pages ([[brief]], [[Repository]]); removing it required rewriting the read-API mental model from "handoff + sugar" to "composition entry points."

---

## 2026-04-29 — meta | add `wiki/brief.md` (agent-facing 30-second project card)

**Reason:** owner needs a stable, terse, code-agent-facing single-pager that can be referenced from `../order-of-relations/CLAUDE.md`. Distinct from [[overview]] (vault navigator) and [[hot]] (volatile churn cache).

**Created:** [[brief]] — top-level meta page; hard rules, five-layer architecture, method-shape facts, dig-deeper table, open questions, wire-up snippet for the code repo's `CLAUDE.md`.

**Updated:** [[index]] — added `[[brief]]` under Top-level; `page_count` 39 → 40.

**Design notes:**
- Naming: `brief.md` (literal lowercase, top-level meta convention).
- Status: `evergreen` — body is stable; facts evolve with the wiki.
- All deeper content reached via wikilinks; the brief itself stays small so the code-side agent pays low fixed cost on every session.

---

## 2026-04-29 — ingest | Repository Contract (final source)

- Source: `.raw/repository-contract.md` (md5 `d61de455c9be742ded75c7361b428c9b`)
- Summary: [[sources/repository-contract]]
- **Pages created (4):**
  - Source: [[sources/repository-contract]]
  - Component: [[Repository]] (the concrete class — distinct from the [[Repository Pattern]] concept)
  - Concept: [[Autogeneration]]
  - Flow: [[lifecycle-of-a-create]]
- **Pages refined (1):**
  - [[Repository Pattern]] — split sharpened from "writes vs. reads" to "**by-key vs. composed**." `create()` signature corrected to `T` (not `Partial<T>`). Return shape clarified to `Partial<T>` with PK fields only (not a hydrated entity). New "How decorators shape the contract" subsection explaining the `NullableField<Value>` / `NotNullableField<Value>` mechanism. Third `> [!note] Refined` callout consolidates the page's three rounds of corrections.
- **Pages updated (indexes):** [[index]], [[hot]], [[sources/_index]] (now ✅ all five), [[concepts/_index]], [[components/_index]], [[flows/_index]].
- **Manifest:** added entry for `repository-contract.md`. **All five sources now tracked.**
- **Vault now holds 38 pages.** Five sources, seven ADRs, four flows, four components, eleven concepts, four entities, three open questions.
- **Closing pattern observation:** every ingest after the first refined existing pages. Concept pages saw on average 2 refinement passes each. The `.raw/`-immutable rule + dated `> [!note] Refined` callouts preserved the full audit trail without history rewriting. The wiki is now structurally consistent — no open contradictions; three open questions are deliberately deferred design choices, not unresolved facts.
- **Two design throughlines** worth carrying forward (now from two sources, paired):
  - QueryBuilder side: *"keep the builder small, keep the SQL parameterized, keep the types honest."*
  - Repository side: *"these are not gaps; they are the boundary. Anything composed, reactive, or stateful goes elsewhere."*

---

## 2026-04-29 — ingest | Query Builder Design

- Source: `.raw/query-builder-design.md` (md5 `a98c07d43266f452d3a460c94adfac59`)
- Summary: [[sources/query-builder-design]]
- **Pages created (5):**
  - Source: [[sources/query-builder-design]]
  - Component: [[QueryBuilder]] (the concrete class — distinct from the [[Lazy Query Builder]] concept page)
  - Concept: [[sqlJoin]]
  - Open questions: [[get-one-limit-1]], [[apply-options-accumulation]]
- **Pages refined (3):**
  - [[Lazy Query Builder]] — corrected from "conceptually immutable" to **mutable, single-owner**. Internal state narrowed to a single `conditions: Condition[] = []` field. Terminal-method list narrowed to `getMany`/`getOne` only (dropped `getCount` — future work). `applyOptions()` semantics documented as **replace, not accumulate** with cross-link to the open question. Second `> [!note] Refined` callout added.
  - [[Conditions Proxy]] — exact callback signature (`(conditions: Conditions<T>) => (Condition | undefined)[]`) and the `Conditions<T>` mapped type added; the `?` partial-mapped-type rationale articulated; `UndefinedWhereConditionError` documented as carrying the offending **index**.
  - [[query-lifecycle]] — Step 5 deepened with the `opFragments` static map, `sql(c.columnName)` identifier binding, and the `IN`-with-empty-array note. `sqlJoin` cross-linked.
- **Pages updated (indexes):** [[index]], [[hot]], [[sources/_index]], [[concepts/_index]], [[components/_index]], [[questions/_index]].
- **Manifest:** added entry for `query-builder-design.md`.
- **Two new open questions filed** — both explicitly named as deferred in the source itself: `getOne` slicing vs `LIMIT 1`, and `applyOptions` replace vs accumulate. Each sketches the design space without endorsing.
- **Key insight:** the source's "throughline" sentence is worth treating as the criterion for any future builder-API addition: *keep the builder small, keep the SQL parameterized, keep the types honest.* Recorded on [[sources/query-builder-design]] § Notes.

---

## 2026-04-29 — filed | Open question: decorator-order independence

- Owner observed during the `@Nullable` resolution that requiring a specific decorator order is a footgun, and asked to file the question for future resolution rather than acting on it now.
- Pages created: [[decorator-order-independence]] — `status: open`. Sketches three design approaches (defer-to-`@Entity`, mutual write-time lookup, `addInitializer` deferral) with tradeoffs; identifies the codebase change surface (`column.ts`, `entity.ts`, tests, two `> [!warning]` callouts to flip).
- Pages updated:
  - [[questions/_index]] — reframed: "filed answers" + "open questions"; added the new entry under "Open questions."
  - [[index]] — Questions section now lists the open question with a 🔓 marker.
  - [[ECMAScript Stage-3 Decorators]] and [[entity-registration]] — both `> [!warning]` callouts now mention the open question, so a reader hitting the order constraint sees that it's a known design-review point.
  - [[hot]] — new "Open Questions" section so future sessions surface this immediately.
- Resolution criteria for closing this question (when it's eventually decided):
  - Pick an option from the design space.
  - If the chosen option requires a non-trivial change to decorator wiring, file an ADR under `wiki/decisions/` and link it from the question.
  - Update the question's `status` from `open` to `answered`, fill in the **Answer** section, and move it under the "Answered questions" heading in [[questions/_index]].

---

## 2026-04-29 — resolved | `@Nullable` write path (three symbols)

- Owner pointed to `src/decorators/nullable/nullable.ts` and `src/decorators/column/column.ts` in [[order-of-relations]]. Code spot-check confirmed:
  - `NULLABLE_KEY = Symbol('nullable')` is declared in `nullable.ts` (line 1).
  - `@Nullable` / `@NotNullable` write `Map<string, boolean>` keyed by property name.
  - `@Column` reads it (lines 31–37) and throws `MissingNullabilityDecoratorError` if no entry; `@PrimaryColumn` is exempt.
  - Decorator-order constraint: `@Nullable` must be the *inner* decorator (closer to the property) so Stage-3's bottom-up application order runs it before `@Column`.
- **Resolution shape:** `architecture-overview.md` was correct; `decorator-metadata-storage.md` had elided `NULLABLE_KEY` because its scope was "what flows into `MetadataStorage`" and `NULLABLE_KEY` is consumed by `@Column` before reaching storage. Both `.raw/` files remain immutable; the wiki corrects in place.
- **Pages updated:**
  - [[ECMAScript Stage-3 Decorators]] — restored to three symbols; added a `> [!warning]` decorator-order callout with a working/broken example; consolidated the two earlier "Refined" notes into a single "History of refinements" callout.
  - [[MetadataStorage]] — replaced the single-paragraph write-path with a three-row table (key / shape / writers / readers); added `MissingNullabilityDecoratorError` to the failure-mode table; downgraded the `> [!contradiction]` callout to a `> [!note] Resolved`.
  - [[entity-registration]] — Step 1 rewritten to cover all three keys and the inner-decorator constraint; new `> [!warning]` callout with the working/broken example; failure-mode table gained the `MissingNullabilityDecoratorError` row.
  - [[sources/decorator-metadata-storage]] — "Open Disagreement" section converted to "Resolved disagreement" with the code-line citations and the rationale for the elision.
  - [[hot]] — moved from "Open Contradictions" to "Recently Resolved"; key-recent-facts updated to three symbols and the order-matters constraint.
- **Lesson** (worth carrying into the remaining two ingests): when one source is *narrower in scope* than another, "more specific within its scope" doesn't mean "more authoritative overall." For symbol keys / decorator wiring, prefer the source that explicitly enumerates the decorator file paths (`architecture-overview.md`'s "How Decorators Talk" section) over one that focuses on a downstream consumer.

---

## 2026-04-29 — ingest | Decorator Metadata Storage

- Source: `.raw/decorator-metadata-storage.md` (md5 `c8c1b3f941bb11d046eff106c4c0af58`)
- Summary: [[sources/decorator-metadata-storage]]
- **Pages created (4):**
  - Source: [[sources/decorator-metadata-storage]]
  - Flow: [[entity-registration]]
  - Concepts: [[Single-Table Inheritance]], [[Relation Target Thunk]]
- **Pages refined (2):**
  - [[ECMAScript Stage-3 Decorators]] — corrected to **two** symbol keys (`COLUMNS_KEY`, `RELATIONS_KEY`); the earlier "three symbols including `NULLABLE_KEY`" framing came from `architecture-overview.md` and is contradicted by this source's explicit code. Added second `> [!note] Refined` callout.
  - [[MetadataStorage]] — same two-symbol correction; deepened with `isMetadataResolved` flag, idempotent re-resolution, WeakMap rationale, `getTarget` thunk, and a full failure-mode table (`MissingPrimaryColumnError`, `RelationTargetNotFoundError`, deliberate-`undefined` for unregistered classes). Added `> [!contradiction]` callout for the inter-source disagreement.
- **Pages updated (indexes):** [[index]], [[hot]], [[sources/_index]], [[concepts/_index]], [[flows/_index]].
- **Manifest:** added entry for `decorator-metadata-storage.md`.
- **New contradiction surfaced:** `architecture-overview.md` claimed `@Nullable` populates a third `NULLABLE_KEY` symbol; this source shows only two symbols and an `@Entity` body that doesn't read a third. Both `.raw/` files immutable. Wiki commits to the two-symbol claim (this source is more specific) and flags the disagreement on [[MetadataStorage]] and [[sources/decorator-metadata-storage]] for resolution by code spot-check or a future ingest.
- **Key insight:** the "manifesto vs. engineering reality" pattern continues — this is the third source-pair disagreement (after test-layout and DI-presence). The ingest pattern that emerges: each new source can refine *and* contradict prior sources; resolutions accumulate on the wiki page rather than backfilling into immutable `.raw/` files.

---

## 2026-04-29 — ingest | Architecture Overview

- Source: `.raw/architecture-overview.md` (md5 `4c677429220411089da13a9fb1aba575`)
- Summary: [[sources/architecture-overview]]
- **Pages created (7):**
  - Source: [[sources/architecture-overview]]
  - Flows: [[query-lifecycle]], [[schema-create-drop]]
  - Components: [[MetadataStorage]], [[Database]]
  - Concepts: [[Layered Architecture]], [[Conditions Proxy]]
- **Pages refined (3 concepts + 1 ADR):**
  - [[ECMAScript Stage-3 Decorators]] — `MetadataStorage` is per-`Database`, not library-global. The earlier "library-owned `Map`" framing was wrong; corrected with a dated `> [!note] Refined` callout.
  - [[Repository Pattern]] — read methods (`findOne`/`findMany`/`findById`) delegate to `QueryBuilder`; only writes build SQL directly. The earlier "trivial vs. composed" split was wrong; the actual split is read vs. write. Example code rewritten to drop the aspirational `@InjectRepository` and use direct construction.
  - [[Lazy Query Builder]] — `where` is a callback receiving a typed [[Conditions Proxy]], not a plain object. Example rewritten.
  - [[0003-singleton-di-container]] — added `> [!warning]` implementation-status callout: DI is **not yet shipped** in `src/`; current pattern is direct construction. ADR `status` left as `accepted` (decision is made; only implementation is pending).
- **Pages updated (indexes):** [[index]], [[hot]], [[sources/_index]], [[concepts/_index]], [[components/_index]], [[flows/_index]], [[modules/_index]] (now hosts the full source-tree map from the source).
- **Manifest:** added entry for `architecture-overview.md`.
- **Key insight:** this source is the **engineering reality** layer to welcome.md's manifesto. Two systematic gaps surfaced: (1) DI not yet implemented (parallel to the test-layout slip from the prior ingest), (2) `MetadataStorage` ownership over-promised in welcome. The remaining three `.raw/` files should be read with that pattern in mind.

---

## 2026-04-29 — resolved | Test-layout contradiction

- Owner confirmed: unit tests are colocated as `*.test.ts` next to source under `src/`; integration tests live under the top-level `tests/` directory. Both run under `bun test`.
- Pages updated:
  - [[0006-tdd-rhythm]] — Decision section rewritten to split unit vs. integration; new `## Clarification (2026-04-29)` block added; Positive consequences and Alternatives Considered expanded to reflect the dual layout. Original text replaced (this is the wiki side, not the immutable source).
  - [[sources/welcome]] — `> [!contradiction]` callout downgraded to a `> [!note]` Resolved note pointing at the clarified ADR. The `.raw/welcome.md` file itself is untouched (immutable per vault policy).
  - [[hot]] — "Open Contradictions" section retired; replaced with a "Recently Resolved" note.
- Resolution shape used: **clarification, not supersession.** The ADR's load-bearing claim (TDD rhythm + `bun test`) was correct; only the location sub-claim was under-specified.

---

## 2026-04-29 — recorded | Codebase location

- Owner provided the location of the source repo this vault documents: local at `../order-of-relations`, remote at <https://github.com/TonyALima/order-of-relations>.
- Pages created: [[order-of-relations]] (entity page).
- Pages updated: [[CLAUDE]] (added "Where the code lives" section), [[overview]] (added repo-location table), [[entities/_index]], [[index]].
- Verified the local checkout exists. Discovered a contradiction with [[0006-tdd-rhythm]] / [[sources/welcome]]: the manifesto says tests are colocated rather than in a separate `tests/` tree, but the repo has both — 11 colocated `*.test.ts` files plus 3 in `tests/`. Filed `> [!contradiction]` callouts on both pages. Resolution pending owner.

---

## 2026-04-29 — ingest | Welcome to the OOR Vault

- Source: `.raw/welcome.md` (md5 `6c43e80854db106e0c8ac31648c25c77`)
- Summary: [[sources/welcome|Welcome to the OOR Vault]]
- Pages created (16):
  - Source: [[sources/welcome]]
  - ADRs: [[0001-stage-3-decorators]], [[0002-repository-with-lazy-query-builder]], [[0003-singleton-di-container]], [[0004-parameterized-sql-only]], [[0005-no-any-type-driven-api]], [[0006-tdd-rhythm]], [[0007-bun-toolchain]]
  - Concepts: [[ECMAScript Stage-3 Decorators]], [[Repository Pattern]], [[Lazy Query Builder]], [[Parameterized SQL]], [[Dependency Injection Container]]
  - Entities: [[Bun]], [[PostgreSQL]], [[TypeScript]]
- Pages updated: [[index]], [[hot]], [[sources/_index]], [[concepts/_index]], [[entities/_index]], [[decisions/_index]]
- Manifest: created `.raw/.manifest.json` with delta entry for `welcome.md`.
- Key insight: welcome.md was authored *as* a decision register draft — the seven "High-Level Decisions" map cleanly onto seven ADRs. Deeper concept pages (metadata storage, query-builder mechanics, repository contract) intentionally left as seeds; they will be filled by the four sibling `.raw/` ingests.

---

## 2026-04-29 — Vault scaffolded (GitHub mode)

- Migrated five pre-existing notes (`Welcome.md`, `Architecture Overview.md`, `Decorator Metadata Storage.md`, `Query Builder Design.md`, `Repository Contract.md`) to `.raw/` as kebab-case source documents. They are immutable from this point on.
- Created full wiki scaffold: `index.md`, `log.md`, `hot.md`, `overview.md`, `getting-started.md`.
- Created subfolders for GitHub mode: `modules/`, `components/`, `decisions/`, `dependencies/`, `flows/`.
- Created standard subfolders: `sources/`, `entities/`, `concepts/`, `domains/`, `comparisons/`, `questions/`, `meta/`.
- Each subfolder seeded with `_index.md`.
- Added `_templates/` (source, entity, concept, comparison, question).
- Applied vault visual customization at `.obsidian/snippets/vault-colors.css`.
- Updated `.gitignore` to cover the wiki conventions.
- Created vault `CLAUDE.md` declaring the wiki schema and conventions.

**Next:** ingest the four remaining `.raw/` files (`architecture-overview.md`, `decorator-metadata-storage.md`, `query-builder-design.md`, `repository-contract.md`) to populate `flows/`, `components/`, and the deeper concept pages currently seeded as stubs.
