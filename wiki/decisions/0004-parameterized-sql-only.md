---
type: decision
title: "ADR 0004 — Parameterized SQL only; no sql.unsafe anywhere"
status: accepted
date: 2026-04-29
deciders:
  - "Tony Albert"
supersedes: []
superseded_by: ""
context: "SQL injection must be a correctness property, not a hardening pass."
created: 2026-04-29
updated: 2026-04-29
tags:
  - decision
  - adr
  - security
  - sql
related:
  - "[[Parameterized SQL]]"
  - "[[sources/welcome]]"
sources:
  - "../.raw/welcome.md"
---

# ADR 0004 — Parameterized SQL only; no `sql.unsafe` anywhere

## Context

Bun's `SQL` driver exposes a tagged-template `sql\`...\`` API for parameterized queries and a `sql.unsafe(text)` escape hatch for raw string interpolation. The escape hatch is convenient but is the canonical SQL-injection vector.

Most ORMs allow `unsafe` on the rationale that "the user knows what they're doing." Empirically, that policy is how injection bugs get shipped.

## Decision

**`sql.unsafe` is banned outright in the OOR codebase.** Every query — whether produced by a repository method, the query builder, or a migration — flows through parameterized templates. SQL injection is treated as a non-negotiable correctness property of the library.

## Consequences

### Positive

- Whole class of vulnerabilities removed from the library's surface by construction.
- No cognitive overhead at PR review time about whether an interpolation is safe.
- Aligns with the strict-typing posture (see [[0005-no-any-type-driven-api]]) — both decisions push errors out at compile time / lint time rather than runtime.

### Negative / trade-offs

- Some otherwise-natural patterns require workarounds. Dynamic identifiers (table names, column names) cannot be parameterized in standard SQL — they must go through an allowlist or quoting helper, never raw interpolation.
- The query builder has to be expressive enough to cover composed cases that lazy contributors might otherwise reach for `unsafe` to solve.

### Neutral

- The rule is enforced by lint, code review, and convention. There is no syntactic mechanism to forbid the import — discipline is the enforcement.

## Alternatives Considered

- **Allow `sql.unsafe` with a comment annotation requirement** — rejected: annotations decay; the next maintainer copies the pattern without the warning.
- **Provide a "trusted-input" helper that wraps `unsafe`** — rejected: same problem with one more layer of indirection.

## References

- `.raw/welcome.md` § "SQL safety: parameterized only"
- [[Parameterized SQL]]
