---
type: source
title: "Welcome to the OOR Vault"
source_type: design-note
author: "Tony Albert"
date_published: 2026-04-29
url: ""
confidence: high
key_claims:
  - "OOR is simultaneously a TCC and a publishable npm package."
  - "Decorators, Repositories, a lazy QueryBuilder, a DI container, and schema migrations are the five pillars."
  - "Stage-3 decorators replace experimentalDecorators + reflect-metadata."
  - "No sql.unsafe, ever — all queries are parameterized."
  - "Strict no-any: the API leans on generics, conditional types, and unknown."
  - "TDD with bun test, with test files colocated next to source."
  - "Bun is the single toolchain — replaces node, ts-node, npm, jest, webpack."
created: 2026-04-29
updated: 2026-04-29
tags:
  - source
  - manifesto
status: stable
related:
  - "[[overview]]"
  - "[[index]]"
sources:
  - "../.raw/welcome.md"
---

# Welcome to the OOR Vault

## Summary

The foundational manifesto for **Order of Relations (OOR)**, a TypeScript ORM for PostgreSQL that is simultaneously an undergraduate thesis (TCC, UNIFEI) and a publishable npm package. The document declares OOR's five architectural pillars (decorators → repositories → lazy query builder → DI container → schema migrations) and enumerates seven load-bearing engineering decisions. It explicitly defers API-surface documentation to the project's `README.md`; the vault is reserved for the *why*.

## Key Claims

- The library maps **decorated TypeScript classes** to PostgreSQL tables. Decorators are the schema definition language.
- `Repository<T>` is the single entry point for an entity's persistence; `Repository.find()` returns a `QueryBuilder<T>` and does **not** execute SQL until a terminal call (`getMany()`, `getOne()`).
- Metadata is stored in a custom `metadataStorage` `Map` owned by the library — there is no `Reflect.metadata` polyfill anywhere.
- A single `Container` holds service singletons; `@Service` + `@Inject` + `@InjectRepository` make repository wiring ergonomic.
- `sql.unsafe` is banned outright. SQL injection is treated as a correctness property, not a hardening pass.
- `@typescript-eslint/no-explicit-any` is enforced strictly. The `create()` signature rejects partial entities at compile time.
- The TCC angle pulls toward conceptual cleanliness; the npm angle pulls toward stability and ergonomics. The vault is where conflicts get resolved.

## Entities Mentioned

- [[Bun]] — the single toolchain (runtime, package manager, test runner, bundler).
- [[PostgreSQL]] — the only target database.
- [[TypeScript]] — the host language; strict-mode no-`any`.

## Concepts Introduced

- [[ECMAScript Stage-3 Decorators]] — the decorator dialect OOR uses, rejecting `reflect-metadata`.
- [[Repository Pattern]] — `Repository<T>` as the persistence entry point.
- [[Lazy Query Builder]] — clauses accumulate; SQL runs only on terminal methods.
- [[Parameterized SQL]] — no `sql.unsafe`, ever.
- [[Dependency Injection Container]] — minimal singleton container for service wiring.

## Decisions Promoted to ADRs

The "High-Level Decisions" section of this source seeded seven ADRs:

- [[0001-stage-3-decorators]]
- [[0002-repository-with-lazy-query-builder]]
- [[0003-singleton-di-container]]
- [[0004-parameterized-sql-only]]
- [[0005-no-any-type-driven-api]]
- [[0006-tdd-rhythm]]
- [[0007-bun-toolchain]]

## Notes

The "Vault Map" section at the bottom of the source lists target wikilinks (`[[Architecture Overview]]`, `[[Decorator Metadata Storage]]`, `[[Query Builder Design]]`, `[[Repository Contract]]`, `[[Schema Migrations]]`, `[[Decisions Log]]`, `[[Open Questions]]`). Most resolve to placeholders today — they are the destinations for upcoming ingests of the four sibling `.raw/` files.
