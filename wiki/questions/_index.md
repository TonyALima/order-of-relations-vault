---
type: domain
title: "Questions"
subdomain_of: ""
page_count: 3
created: 2026-04-29
updated: 2026-04-29
tags:
  - meta
  - questions
status: developing
related:
  - "[[index]]"
sources: []
---

# Questions

Two flavors live here:

- **Filed answers** — questions that got a good answer in chat, saved so future-you (and the TCC defense panel) can find them again. `status: answered`.
- **Open questions** — design questions and implementation issues worth keeping visible until they're decided **and shipped**. `status: open`. By convention in this vault, an open question implies a commitment to implement once decided — there is no "decided but not implemented" state.

When an open question gets resolved, update its `status` to `answered`, fill in the **Answer** section, and (if it produced a design choice) point `decided_by` at the matching ADR under `wiki/decisions/`.

## Issue-tracker frontmatter

Each open question carries:

- `impact: low | medium | high` — cost of leaving it open.
- `effort: S | M | L` — rough implementation size (S ≈ one-line, L ≈ multi-file refactor).
- `decided_by: ""` — wikilink to the ADR that closed it (empty until decided).

The live, sortable view is at [[issues|Issue tracker]] (`wiki/meta/issues.base`). Score formula: `impact × 2 + effort_inverse`, so high-impact + low-effort sorts to the top.

## Open questions

| Question | Impact | Effort |
|---|---|---|
| [[decorator-order-independence]] — `@Column` / `@Nullable` order independence | medium | S |
| [[get-one-limit-1]] — `getOne()` `LIMIT 1` vs slice | low | S |
| [[apply-options-accumulation]] — `applyOptions()` replace vs accumulate | low | S |

## Answered questions

- _none yet_
