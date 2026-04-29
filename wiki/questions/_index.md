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
- **Open questions** — design questions and review points worth keeping visible until they're decided. `status: open`.

When an open question gets resolved, update its `status` to `answered`, fill in the **Answer** section, and (if it produced a design choice) link to the matching ADR under `wiki/decisions/`.

## Open questions

- [[decorator-order-independence]] — *2026-04-29* — should `@Column` / `@Nullable` work regardless of decoration order?
- [[get-one-limit-1]] — *2026-04-29* — should `QueryBuilder.getOne()` emit `LIMIT 1` instead of slicing client-side?
- [[apply-options-accumulation]] — *2026-04-29* — should `QueryBuilder.applyOptions()` accumulate where-conditions instead of replacing?

## Answered questions

- _none yet_
