---
type: meta
title: "Operation Log"
created: 2026-04-29
updated: 2026-04-29
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
