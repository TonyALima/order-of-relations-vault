---
type: decision
title: "ADR 0001 — Use ECMAScript Stage-3 Decorators"
status: accepted
date: 2026-04-29
deciders:
  - "Tony Albert"
supersedes: []
superseded_by: ""
context: "Need a decorator dialect for entity, column, and relation definitions."
created: 2026-04-29
updated: 2026-04-29
tags:
  - decision
  - adr
  - decorators
related:
  - "[[ECMAScript Stage-3 Decorators]]"
  - "[[sources/welcome]]"
sources:
  - "../.raw/welcome.md"
---

# ADR 0001 — Use ECMAScript Stage-3 Decorators

## Context

OOR maps TypeScript classes to PostgreSQL tables using decorators (`@Entity`, `@Column`, etc.). Two viable decorator dialects exist:

1. **Legacy TypeScript decorators** — `experimentalDecorators: true` + the `reflect-metadata` runtime polyfill. Used by TypeORM, NestJS, Inversify.
2. **ECMAScript Stage-3 decorators** — the standardized proposal, supported natively by modern TypeScript and runtimes. Different signature, no `Reflect.metadata` reliance.

A choice has to be made before any decorator is written, because the two dialects have incompatible function signatures.

## Decision

**We will use ECMAScript Stage-3 decorators exclusively.** Metadata is stored in a custom `metadataStorage` `Map` owned by the library, not on `Reflect`.

## Consequences

### Positive

- No runtime dependency on `reflect-metadata`. One fewer transitive dependency for consumers.
- Aligned with the language proposal pipeline — the dialect is the future, not a legacy compat path.
- Native support in modern toolchains (including [[Bun]]) without extra `tsconfig` flags.
- Library owns its own metadata layout. We can change storage shape without colliding with whatever else uses `Reflect.metadata` in a consumer's app.

### Negative / trade-offs

- The Stage-3 decorator signature is more verbose than the legacy form (e.g., the `context` parameter is mandatory).
- Smaller helper ecosystem. Existing decorator utilities written for legacy decorators don't apply.
- Some IDE tooling and reference material still defaults to the legacy dialect — onboarding contributors costs a paragraph of explanation.

### Neutral

- Forces a custom `MetadataStorage` abstraction (see [[sources/decorator-metadata-storage|Decorator Metadata Storage]] for the pin-down). This is also a benefit for testability, but it is mostly *different*, not *better* than the legacy approach.

## Alternatives Considered

- **`experimentalDecorators` + `reflect-metadata`** — rejected: legacy, requires polyfill, locks the library into a flag that's now considered transitional.
- **Build-time codegen instead of decorators** — rejected: defeats the ergonomic case for an ORM. Decorators are what consumers expect.

## References

- `.raw/welcome.md` § "Stage-3 decorators, no `reflect-metadata`"
- [[ECMAScript Stage-3 Decorators]]
