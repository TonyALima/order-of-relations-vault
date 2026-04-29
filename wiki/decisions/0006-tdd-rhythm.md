---
type: decision
title: "ADR 0006 — TDD as the implementation rhythm"
status: accepted
date: 2026-04-29
deciders:
  - "Tony Albert"
supersedes: []
superseded_by: ""
context: "An ORM is mostly invisible logic; without tests, regressions are silent."
created: 2026-04-29
updated: 2026-04-29
tags:
  - decision
  - adr
  - testing
  - tdd
related:
  - "[[Bun]]"
  - "[[sources/welcome]]"
sources:
  - "../.raw/welcome.md"
---

# ADR 0006 — TDD as the implementation rhythm

## Context

ORMs do most of their work between an API call and a SQL string. The translation logic is hidden inside generic types, decorator metadata, and clause composition. Without tests written *first*, regressions surface as "weird query behavior" reported by consumers — far too late.

## Decision

**Every feature lands by red-green-refactor.** Write a failing test, make it green, then refactor. Test files are colocated next to source: `foo.ts` lives beside `foo.test.ts` rather than under a separate `tests/` tree. Test runner is `bun test`.

## Consequences

### Positive

- The test is the spec. Any change to behavior is necessarily preceded by a change to a test.
- Colocation means tests are discoverable by anyone reading the source — no separate `tests/` mental model.
- `bun test` is fast enough to keep the loop tight.

### Negative / trade-offs

- Slower for the *first* commit of a feature. The discipline pays off across the project's lifetime, not in any individual hour.
- Tooling has to ignore `*.test.ts` in production builds (handled by Bun's defaults).

### Neutral

- Encourages small, testable units. Functions that are awkward to test get refactored before they grow.

## Alternatives Considered

- **Integration-first testing** — partially adopted: end-to-end tests against a real PostgreSQL still exist and are valuable. But the *rhythm* — what gates a commit — is the per-unit failing test.
- **Tests in a separate `tests/` tree** — rejected: distance from the source erodes the habit of writing them.

## References

- `.raw/welcome.md` § "TDD as the implementation rhythm"
