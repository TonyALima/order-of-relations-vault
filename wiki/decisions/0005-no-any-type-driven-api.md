---
type: decision
title: "ADR 0005 — Strict no-any; type-driven public API"
status: accepted
date: 2026-04-29
deciders:
  - "Tony Albert"
supersedes: []
superseded_by: ""
context: "TypeScript users expect compile-time guarantees from a typed ORM."
created: 2026-04-29
updated: 2026-04-29
tags:
  - decision
  - adr
  - typescript
  - api
related:
  - "[[TypeScript]]"
  - "[[sources/welcome]]"
sources:
  - "../.raw/welcome.md"
---

# ADR 0005 — Strict no-`any`; type-driven public API

## Context

ORMs that lean on `any` (`Repository.find(criteria: any)`, `(row: any).foo`) erase the value of writing TypeScript at all. Conditional types and generics let the compiler verify entity shapes, partial inputs, and result rows — but only if `any` is forbidden.

## Decision

**`@typescript-eslint/no-explicit-any` is enforced strictly across the codebase.** The library leans on generics, conditional types, and `unknown` to expose compile-time guarantees to consumers. Concretely: `create()`'s signature requires non-optional fields and rejects partial entities at compile time, rather than failing at runtime with a `NOT NULL` violation.

## Consequences

### Positive

- Consumer-facing errors fire in the IDE, not in production.
- The type system encodes invariants the runtime would otherwise enforce too late (missing required column, wrong column type, unknown relation).
- Refactors are safer: renaming a column on the entity class breaks every miswritten call site.

### Negative / trade-offs

- Internal type plumbing is heavier. Helper types (`Required<>`, `Pick<>`, mapped types over the entity's `@Column` keys) carry the load that `any` would have hidden.
- Compile errors can be cryptic when conditional types resolve through several layers. Documentation has to compensate.
- Library author productivity is somewhat slower — reaching for `unknown` + a narrowing cast is friction that `any` would have removed.

### Neutral

- Pushes the library toward smaller, more orthogonal types. Big bag-of-options types are the first place `any` creeps back in.

## Alternatives Considered

- **Allow `any` in internal-only modules, ban in public API** — rejected: internals leak into public types more often than not.
- **Use `unknown` everywhere instead** — partially adopted: `unknown` is the right answer when a value's shape is genuinely opaque. The decision here is specifically against `any`.

## References

- `.raw/welcome.md` § "Type-driven API: no `any`"
