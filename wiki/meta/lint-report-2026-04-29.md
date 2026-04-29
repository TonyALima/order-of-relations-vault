---
type: meta
title: "Lint Report 2026-04-29"
created: 2026-04-29
updated: 2026-04-29
tags:
  - meta
  - lint
status: developing
---

# Lint Report: 2026-04-29

First lint pass after the bulk-ingest of all five `.raw/` sources.

## Summary

- **Pages scanned:** 54 (44 content + 10 meta-hub `_index.md` / dashboards)
- **Real issues found:** 5 dead wikilinks (across 3 unique targets)
- **False positives identified:** 9 "dead links" that are actually inside backticks (documentation prose), 40+ "empty sections" that are normal heading hierarchy
- **Auto-fixed (2026-04-29):** 2 — `[[Decorator Metadata Storage]]` rename in `0001-stage-3-decorators.md`, and `first_mentioned` in `order-of-relations.md`
- **Resolved by stub creation (2026-04-29):** 2 — both `[[Schema Migrations]]` occurrences now resolve to a new seed concept page
- **Preserved by policy:** 1 — the `[[CLAUDE]]` reference in `log.md:148` is in an append-only log entry; per vault policy, log entries are never edited
- **Pending owner decision:** 0
- **Pending — documentation:** 1 — naming-convention split should be documented in vault `CLAUDE.md`
- **Naming-convention drift:** 0 within each category; CLAUDE.md doesn't document the per-category convention split — informational

## Wiki health overall: ✅ green

| Check | Status |
|---|---|
| Orphan pages | ✅ Zero orphans. Every non-meta page has at least one inbound wikilink. |
| Frontmatter completeness | ✅ Every page has all six required fields (`type`, `title`, `created`, `updated`, `tags`, `status`). |
| `_index.md` page-count drift | ✅ All declared `page_count:` values match actual file counts. |
| Master `index.md` stale entries | ✅ Every wikilink in `index.md` resolves to a real page. |
| Empty sections | ✅ Zero genuinely empty sections after filtering parent-with-subheadings false positives. |
| Active unresolved contradictions | ✅ Zero. (6 `[!contradiction]` text matches are all inside backticks — docs about the pattern, not raised flags.) |
| Open questions tracked | ✅ 3 open questions with closure criteria documented. |

## Dead Links

### Real (5 occurrences across 3 unique targets)

#### `[[Decorator Metadata Storage]]` — 1 occurrence — ✅ FIXED 2026-04-29

- **`wiki/decisions/0001-stage-3-decorators.md:56`** — written before that source was ingested. The synthesis page now exists at `wiki/sources/decorator-metadata-storage.md`.
- **Fix applied:** renamed to `[[sources/decorator-metadata-storage|Decorator Metadata Storage]]` and updated the prose from *"when ingested"* to *"for the pin-down"* (since it's now ingested).

#### `[[Schema Migrations]]` — 2 occurrences — ✅ RESOLVED 2026-04-29

- **`wiki/components/Database.md:54`** — *"Migrations beyond create/drop are out of scope for this class — see `[[Schema Migrations]]`"*
- **`wiki/flows/schema-create-drop.md:26`** — *"Migrations beyond create/drop … live in `[[Schema Migrations]]` — currently a placeholder."*
- **Resolution:** Created [[Schema Migrations]] as a seed concept page (option 1 from the original three options). The page defines the concept, explicitly flags that no migration system is implemented in OOR yet via a `> [!gap]` callout, lists the load-bearing design choices a future implementation would have to make (versioning model, forward-only vs. up/down, TS-first vs. SQL-first, runner architecture, migration-table location, relationship to `Database.create()`), and links back to the references. The wikilinks in `Database.md` and `schema-create-drop.md` now resolve cleanly without needing edits.

#### `[[CLAUDE]]` — 2 occurrences — 1 fixed, 1 preserved by policy

- **`wiki/entities/order-of-relations.md:6`** (frontmatter) — ✅ **FIXED 2026-04-29**: `first_mentioned: "[[CLAUDE]]"` → `first_mentioned: "[[overview]]"` (the OOR repo is also referenced in `wiki/overview.md`, which is unambiguously inside the wiki).
- **`wiki/log.md:148`** — ⚠️ **PRESERVED BY POLICY**: this is inside an append-only log entry. Vault rule: *"`wiki/log.md` is append-only: never edit past entries; new entries go at the top."* The dead wikilink is a faithful historical record of what was done at the time — readers can read it as plain context. Editing it would violate the immutability invariant that makes the log trustworthy.
- **Lesson:** future lint passes should treat `wiki/log.md` (and `.raw/`, and `wiki/folds/` if/when added) as **read-only zones** — dead links inside append-only files are not lint issues, they're audit trail. Recommend adding this to the skill convention so future runs skip those files.

### False positives (filtered out — inside backticks, not real links)

These appeared in the dead-link scan but resolve to *prose about wikilink syntax*, not actual rendered links:

- `[[Note Name]]` in `wiki/overview.md:63` — documentation example.
- `[[Architecture Overview]]`, `[[Decorator Metadata Storage]]`, `[[Query Builder Design]]`, `[[Repository Contract]]`, `[[Schema Migrations]]`, `[[Decisions Log]]`, `[[Open Questions]]` in `wiki/sources/welcome.md:74` — all inside a paren-listed code-fenced sequence quoting the source's own Vault Map for context.

No action needed.

## Naming Convention

The vault uses **category-specific** naming, consistent within each category but not uniform across the vault:

| Category | Pattern | Examples |
|---|---|---|
| `concepts/` | Title Case | `Lazy Query Builder.md`, `ECMAScript Stage-3 Decorators.md`. Exception: `sqlJoin.md` (matches the literal function name). |
| `decisions/` | `NNNN-short-title.md` | `0001-stage-3-decorators.md`, … (standard ADR convention; documented in `decisions/_index.md`). |
| `sources/` | lowercase-kebab matching `.raw/` filename | `welcome.md`, `architecture-overview.md`. Symmetric with the manifest's `.raw/` paths. |
| `flows/` | lowercase-kebab | `query-lifecycle.md`, `entity-registration.md`. |
| `questions/` | lowercase-kebab | `decorator-order-independence.md`. |
| `entities/` | Title Case for proper nouns; repo-name kebab for codebases | `Bun.md`, `PostgreSQL.md`, `order-of-relations.md` (matches the GitHub repo name). |
| `components/` | PascalCase matching the exported class name | `Repository.md`, `QueryBuilder.md`, `MetadataStorage.md`, `Database.md`. |

**Issue:** `CLAUDE.md` claims wiki convention is "Title Case with spaces" but practice is per-category. **Suggest** updating `CLAUDE.md` § Conventions to document the actual split, so future contributors don't break either pattern.

**Auto-fix safety:** Needs review — this is documentation, not data; the user should decide the framing.

## Stale Claims

None detected. Every claim that was contradicted by a later source has been resolved (downgraded to a `> [!note] Resolved` callout) or refined in place with a dated `> [!note] Refined` callout. Audit trail: 3 resolved-note callouts, 0 active contradictions, 9 `> [!warning]` callouts (all are footgun warnings on real constraints, not stale claims).

## Cross-Reference Gaps

Spot-checked: every entity name (`Bun`, `PostgreSQL`, `TypeScript`, `order-of-relations`) is wikilinked on first mention in pages where it appears. Concept names (e.g., `Repository Pattern`, `Lazy Query Builder`) are wikilinked consistently. No gaps found.

## Frontmatter Gaps

None. Every page has `type`, `title`, `created`, `updated`, `tags`, `status`.

## Stale Index Entries

`wiki/index.md` and every `_index.md` file's `page_count` and listing are in sync with the on-disk reality.

## Open Questions

Tracked under `wiki/questions/` with `status: open` and closure criteria documented:

- 🔓 [[decorator-order-independence]]
- 🔓 [[get-one-limit-1]]
- 🔓 [[apply-options-accumulation]]

These are not lint issues — they are deliberate design-review points. Listed here for visibility only.

## Recommended Actions

Status as of 2026-04-29:

1. ✅ **Done** — renamed `[[Decorator Metadata Storage]]` → `[[sources/decorator-metadata-storage|Decorator Metadata Storage]]` in `0001-stage-3-decorators.md:56`.
2. ✅ **Done** — `first_mentioned` in `order-of-relations.md` retargeted to `[[overview]]`.
3. ✅ **Done** — created [[Schema Migrations]] seed concept page (option 1, stub). Both dead-link occurrences in `Database.md` and `schema-create-drop.md` now resolve.
4. ⏸️ **Pending** — document naming-convention split in vault `CLAUDE.md`'s "Conventions" section.
5. ⏸️ **Skipped by policy** — `[[CLAUDE]]` in `log.md:148`: append-only log entry, not editable. Future lint runs should classify `log.md` as a read-only zone.

## What was checked but reported zero issues

For confidence: these scans ran and found nothing actionable.

- Orphan pages (zero)
- Frontmatter gaps (zero)
- Empty sections (zero, after filtering hierarchy false positives)
- Stale `_index.md` page-counts (zero drift)
- Master `index.md` stale entries (zero)
- Active unresolved contradictions (zero)
- DragonScale Address Validation (skipped: feature not enabled — no `scripts/allocate-address.sh`, no `.vault-meta/`)
- Semantic Tiling (skipped: feature not enabled — no `scripts/tiling-check.py`)

## Vault-state snapshot

- 5 sources (all `.raw/` files ingested)
- 7 ADRs
- 11 concepts
- 4 components
- 4 flows
- 4 entities
- 3 open questions
- 1 codebase entity
- **38 content pages + 10 hub/meta = 48 markdown files** (the earlier "54" count includes a few _templates/ entries this lint scoped under `wiki/`)
- Manifest tracks all five sources by hash, with full pages-created and pages-updated arrays
- `.raw/.manifest.json` is the canonical "what's ingested" registry
