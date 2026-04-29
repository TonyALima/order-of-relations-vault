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

**Every feature lands by red-green-refactor.** Write a failing test, make it green, then refactor. Test runner is `bun test`.

Test files split by **scope**:

- **Unit tests** are colocated next to source: `foo.ts` lives beside `foo.test.ts` under `src/`. These exercise a single module in isolation; they make up the red-green-refactor inner loop and gate every commit.
- **Integration tests** live under a top-level `tests/` directory at the repo root. These exercise behaviors that span multiple modules or hit a real PostgreSQL — patterns that don't fit naturally next to any single `src/` file.

Both kinds run under `bun test` and are mandatory for the affected change-set.

## Clarification (2026-04-29)

The original source (`.raw/welcome.md`) framed colocation as absolute ("rather than under a separate `tests/` tree"). Verification against [[order-of-relations]] showed the codebase actually uses **both** layouts — 11 colocated `*.test.ts` files in `src/` plus 3 integration tests in `tests/`. The Decision section above is the resolved rule. The original `.raw/welcome.md` text is left as-is per vault policy; see [[sources/welcome]] for the resolved note pointing here.

## Consequences

### Positive

- The test is the spec. Any change to behavior is necessarily preceded by a change to a test.
- Colocation means unit tests are discoverable by anyone reading the source — they sit beside the file under test.
- Integration tests living in a single top-level `tests/` directory are easy to enumerate, target (`bun test tests/`), and skip in environments where a real PostgreSQL is unavailable.
- `bun test` is fast enough to keep the loop tight.

### Negative / trade-offs

- Slower for the *first* commit of a feature. The discipline pays off across the project's lifetime, not in any individual hour.
- Tooling has to ignore `*.test.ts` in production builds (handled by Bun's defaults).

### Neutral

- Encourages small, testable units. Functions that are awkward to test get refactored before they grow.

## Alternatives Considered

- **Integration-first testing** — partially adopted: end-to-end tests against a real PostgreSQL still exist and are valuable, and now have a designated home in `tests/`. But the *rhythm* — what gates a commit at the unit level — is the per-unit failing test next to the source.
- **All tests in a separate `tests/` tree** — rejected: distance from the source erodes the habit of writing unit tests. The `tests/` directory is for integration scope only.
- **All tests colocated, no `tests/` directory** — rejected (per the 2026-04-29 clarification): integration tests that span multiple modules or require a live database don't have a single natural source file to sit beside. Forcing colocation in those cases produces awkward ownership.

## References

- `.raw/welcome.md` § "TDD as the implementation rhythm"
