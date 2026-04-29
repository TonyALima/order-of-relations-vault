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
