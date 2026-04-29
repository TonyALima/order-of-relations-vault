---
type: decision
title: "ADR 0003 — Singleton DI container, intentionally minimal"
status: accepted
date: 2026-04-29
deciders:
  - "Tony Albert"
supersedes: []
superseded_by: ""
context: "Need ergonomic injection of repositories without competing with serious DI frameworks."
created: 2026-04-29
updated: 2026-04-29
tags:
  - decision
  - adr
  - di
  - container
related:
  - "[[Dependency Injection Container]]"
  - "[[sources/welcome]]"
sources:
  - "../.raw/welcome.md"
---

# ADR 0003 — Singleton DI container, intentionally minimal

## Context

Repositories need to be wired into services. Three positions are available:

1. Hand-roll: consumers `new Repository(...)` themselves and pass instances around.
2. Use a serious DI library (Inversify, tsyringe).
3. Build a small, in-house container.

Position 1 is unergonomic past two services. Position 2 imports a major dependency for what is, in OOR's case, a single use case (inject a repository into a service).

## Decision

**A single `Container` holds service singletons.** `@Service` wraps a constructor so `@Inject` and `@InjectRepository` fields populate automatically. The container is intentionally minimal — single scope (singleton), no async resolution, no lifecycle hooks.

## Consequences

### Positive

- Zero external DI dependency.
- The 90% case (wire a repository into a service) is one decorator.
- The container is small enough to read end-to-end — no surprises.

### Negative / trade-offs

- No request-scoped instances. Anything that needs scoping has to manage it manually.
- No async factories. If a service needs async init, it has to expose an `init()` method and the consumer has to call it.
- Will not satisfy users who want full IoC. They should reach for Inversify or tsyringe instead.

### Neutral

- Sets a clear scope: the container exists for repository injection. If it accumulates features beyond that, that is a smell to push back on.

## Alternatives Considered

- **Inversify / tsyringe** — rejected: too much surface for the actual need, and they push consumers to adopt their patterns project-wide.
- **No container, manual wiring** — rejected: the ergonomics are bad enough that consumers would invent their own container anyway.

## References

- `.raw/welcome.md` § "DI container as a singleton"
- [[Dependency Injection Container]]
