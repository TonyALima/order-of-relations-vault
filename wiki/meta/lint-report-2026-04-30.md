---
type: meta
title: "Lint Report 2026-04-30"
created: 2026-04-30
updated: 2026-04-30
tags:
  - meta
  - lint
status: developing
---

# Lint Report: 2026-04-30

Health check after the ingest of `.raw/pk-aware-repository-methods.md` (3 pages created, 11 pages updated). First lint pass on the post-PK-brand vault.

## Summary

- **Pages scanned:** 82 total markdown files (61 content pages by `wiki/index.md`'s tally, plus 13 `_index.md` hubs + 7 top-level meta + 1 archived lint report).
- **Real issues found:** **1** intentional placeholder (pre-existing, author-by-design).
- **False positives identified:** 33 (backtick-wrapped format docs, Obsidian Bases file references, escaped pipes in markdown table cells, prior lint report contents).
- **New ingest health:** ✅ green — all three new pages have complete frontmatter and ≥7 inbound wikilinks. No new dead links introduced.
- **Auto-fix candidates:** 1 minor (`wiki/meta/_index.md` page_count drift `0 → 1`).

## Wiki health overall: ✅ green

| Check | Status |
|---|---|
| Orphan pages | ✅ Zero orphans across all 82 markdown files. |
| Frontmatter completeness (new + updated pages) | ✅ All 9 new/updated pages carry `type` / `title` / `created` / `updated` / `tags` / `status`. |
| Folder `_index.md` page_count drift | ⚠️ 1 of 13 — `wiki/meta` claims 0, actual 1. |
| Master `index.md` stale entries | ✅ Every wikilink in `index.md` resolves. |
| Inbound wikilinks to new pages | ✅ `[[PrimaryKey Brand]]` (9), `[[sources/pk-aware-repository-methods]]` (8), `[[0008-pk-aware-compile-time]]` (7). |
| Active unresolved contradictions | ✅ Zero. The 2026-04-30 ingest's refinement notes on Repository / Repository Pattern / Autogeneration / Conditions Proxy / ECMAScript Stage-3 Decorators all describe **completed** transitions, not unresolved conflicts. |

## Orphans

**None.** Every non-meta page has at least one inbound wikilink.

## Dead Wikilinks

### Genuine — author-intentional placeholder (1)

- `wiki/sources/drift-d5-discriminator-index.md:85` → `[[idx-discriminator-collision]]` — described in-line as *"Optional future open question"*. Author is leaving a hint about a possible future page rather than committing to file it. **Recommendation:** keep as-is (matches the convention of forward-pointing hints), OR file a `wiki/questions/idx-discriminator-collision.md` stub if you want to convert the hint into a tracked open question. No urgency.

### False positives identified (33)

These look like dead links to a regex but are not in practice:

| Pattern | Where | Why it's not a dead link |
|---|---|---|
| `` `[[Architecture Overview]]` `` etc. (×6) | `wiki/sources/welcome.md:74` | Inside inline-code backticks — documenting the wikilink format the upcoming ingests would use. |
| `` `[[Note Name]]` `` (×2) | `wiki/overview.md:63` and prior lint report | Inside inline-code backticks — documenting the wikilink syntax. |
| `[[oor-architecture.canvas\|OOR Architecture]]` | `wiki/overview.md:79` | Resolves to `canvases/oor-architecture.canvas` (Canvas file outside `wiki/`). Obsidian resolves it; my regex didn't follow the escaped pipe in a table cell. |
| `[[modules/sql-types\|...]]`, `[[_index\|...]]` (×4) | `wiki/comparisons/*.md` | Escaped pipe inside a markdown table cell. Real targets resolve. |
| `[[issues\|Issue tracker]]` (×4) | `wiki/index.md`, `wiki/hot.md`, `wiki/questions/_index.md` | Resolves to `wiki/meta/issues.base` (Obsidian Base file, not `.md`). Working in Obsidian. |
| `[[examples/<name>]]` | `wiki/examples/_index.md` | Placeholder syntax inside instructions. |
| `lint-report-2026-04-29.md` contents (×16) | `wiki/meta/lint-report-2026-04-29.md` | Past lint report documents historical dead links; per the `log.md` audit-trail convention, lint reports likewise preserve their original text. |

## Frontmatter Gaps

None across the 9 new/updated pages. Spot-checked Repository.md, Repository Pattern.md, Autogeneration.md, Conditions Proxy.md, ECMAScript Stage-3 Decorators.md, brief.md, plus all three new pages — all carry the six required fields.

## Stale Claims

Spot-checked the five pages that the new ingest refined for residual stale claims after the edits:

- ✅ `wiki/components/Repository.md` — Operations table updated; `create()` runtime section now says `Promise<PKOutput<T>>`; failure-modes table notes which type-system check fires before each runtime gate; new "PK-aware compile-time enforcement" callout filed.
- ✅ `wiki/concepts/Repository Pattern.md` — Refinement-4 callout added; example comment corrected (`PKOutput<User>` not `Partial<User>`); `@PrimaryColumn` § now mentions both overloads carry the brand term.
- ✅ `wiki/concepts/Autogeneration.md` — Silent-`update` bug now noted as closed by the brand work; example entities updated to declare `id?: PrimaryKey<number>` / `id?: PrimaryKey<string>` / `externalId!: PrimaryKey<string>`.
- ✅ `wiki/concepts/Conditions Proxy.md` — `Conditions<T>` formal definition now uses `Unbrand<T[K]>`; refinement note + sources updated.
- ✅ `wiki/concepts/ECMAScript Stage-3 Decorators.md` — New `## The constraint-flip pattern (read-only)` section generalizes the pattern to cover both nullability and the brand.
- ✅ `wiki/brief.md` — `create()` row updated; new `findById` / `delete` / `update` rows added.

The single remaining grep hit for `Partial<T>` is the historical-correction phrasing on `wiki/brief.md:49` (*"signature is `T` ... not `Partial<T>`. Returns `PKOutput<T>` ..."*) — preserved as documentation of the *transition*, not a stale claim.

## Cross-Reference Gaps

None new. The new pages link out to nine related pages (Repository, Repository Pattern, Autogeneration, Conditions Proxy, ECMAScript Stage-3 Decorators, ADR 0005, ADR 0008, source-PK-aware, source-repository-contract). The five updated pages all add reciprocal links.

## Folder Page-Count Drift

| Folder | Claimed | Actual | Status |
|---|---|---|---|
| `wiki/meta` | 0 | 1 | ⚠️ off-by-one — `lint-report-2026-04-29.md` exists but `_index.md` `page_count: 0` was never bumped when the prior lint report shipped. |

(All 12 other `_index.md` files have correct counts.)

**Recommendation:** bump `wiki/meta/_index.md` `page_count: 0 → 1` (or 2 once this current report is committed and counted). One-line fix, safe to auto-apply.

## Naming-convention compliance

All new pages follow the per-category conventions documented in `CLAUDE.md`:

| Page | Folder | Convention | Compliance |
|---|---|---|---|
| `PrimaryKey Brand.md` | `concepts/` | Title Case with spaces | ✅ |
| `pk-aware-repository-methods.md` | `sources/` | lowercase-kebab mirroring `.raw/` | ✅ |
| `0008-pk-aware-compile-time.md` | `decisions/` | `NNNN-short-title.md` | ✅ |

## Content-quality spot-check

Style guide compliance on the three new pages:

- ✅ Declarative present tense throughout (no "X basically does Y").
- ✅ Source citations on every load-bearing claim — every `Why we chose A`, `Brand asymmetry`, and `Method signatures` block points back to the `.raw/pk-aware-repository-methods.md` source or a sibling synthesis page.
- ✅ Uncertainty correctly framed — out-of-scope items (`pk<V>` helper, branding non-PK fields, auto-injecting decorator) are tagged as `Out of scope` rather than left as ambiguous gaps.
- ✅ No `> [!contradiction]` callouts needed — the source supersedes the older `Partial<T>`-on-`create()` claim from `repository-contract`, but explicitly as a follow-up extension rather than a conflict; the refinement notes document the transition.

## DragonScale opt-in checks

- **Address Validation (Mechanism 2):** ❌ Not enabled (`scripts/allocate-address.sh` and `.vault-meta/address-counter.txt` not present). Section skipped per spec.
- **Semantic Tiling (Mechanism 3):** ❌ Not enabled (`scripts/tiling-check.py` not present). Section skipped per spec.

## Recommended fixes

| Severity | Fix | Auto-fix safe? |
|---|---|---|
| Low | `wiki/meta/_index.md` `page_count: 0 → 1` (or 2 once this report is committed) | ✅ Yes — one frontmatter line. |
| Optional | File a stub `wiki/questions/idx-discriminator-collision.md` to convert the hint in `drift-d5` into a tracked open question | ❌ Needs author judgment — current "Optional future" framing is a deliberate forward pointer, not a dead link. |

## Suggested next actions

- Commit the new pages + the updated meta files. (The PostToolUse hook may have already staged them.)
- A future ingest of any remaining `.raw/` follow-up memos can use this report as the baseline for the next health snapshot.
- Consider revisiting `wiki/comparisons/orms-summary.md` to surface "compile-time PK enforcement on `findById`/`update`/`delete` via a runtime-erased brand" as a row in the contribution matrix — none of TypeORM / Drizzle / Prisma do this, per the existing comparison pages. Tracked as an active thread in `wiki/hot.md`.
