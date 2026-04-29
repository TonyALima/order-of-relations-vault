---
type: entity
title: "TypeScript"
entity_type: language
role: "Host language — strict mode, no-any, Stage-3 decorators"
first_mentioned: "[[sources/welcome]]"
created: 2026-04-29
updated: 2026-04-29
tags:
  - entity
  - language
status: seed
related:
  - "[[0001-stage-3-decorators]]"
  - "[[0005-no-any-type-driven-api]]"
  - "[[Bun]]"
sources:
  - "../.raw/welcome.md"
---

# TypeScript

## Overview

TypeScript is the host language for OOR. The library is shipped as TypeScript source and consumed as TypeScript by users — the type system is part of the public API surface, not a development-time convenience. Two cross-cutting policies dominate: **Stage-3 decorators** are the only decorator dialect used, and **`@typescript-eslint/no-explicit-any` is enforced strictly**.

## Key Facts

- **Stage-3 decorators only.** No `experimentalDecorators`, no `reflect-metadata`. See [[0001-stage-3-decorators]] and [[ECMAScript Stage-3 Decorators]].
- **Strict no-`any`.** `@typescript-eslint/no-explicit-any` is enforced. The library uses generics, conditional types, and `unknown` to push errors to compile time. See [[0005-no-any-type-driven-api]].
- **Type-driven public API.** `Repository<T>.create()` rejects partial entities at compile time; the [[Lazy Query Builder]] narrows result types as `select()` columns are projected.
- **Executed by [[Bun]].** TS source runs directly under Bun — no separate transpile step in development or test loops.

## Connections

- [[0001-stage-3-decorators]] — decorator-dialect ADR.
- [[0005-no-any-type-driven-api]] — strictness ADR.
- [[ECMAScript Stage-3 Decorators]] — the underlying language feature.
- [[Bun]] — the runtime that executes TS source directly.

## Sources

- `.raw/welcome.md`
