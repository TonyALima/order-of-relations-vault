---
type: decision
title: "ADR 0007 — Bun as the single toolchain"
status: accepted
date: 2026-04-29
deciders:
  - "Tony Albert"
supersedes: []
superseded_by: ""
context: "Toolchain sprawl (node + ts-node + npm + jest + webpack) creates friction with no payoff for a small ORM."
created: 2026-04-29
updated: 2026-04-29
tags:
  - decision
  - adr
  - toolchain
  - bun
related:
  - "[[Bun]]"
  - "[[sources/welcome]]"
sources:
  - "../.raw/welcome.md"
---

# ADR 0007 — Bun as the single toolchain

## Context

A small ORM's toolchain easily ends up at five components: a runtime (`node`), a TypeScript executor (`ts-node`/`tsx`), a package manager (`npm`/`pnpm`), a test runner (`jest`/`vitest`), and a bundler (`webpack`/`tsup`/`rollup`). Each has its own configuration, version drift, and edge cases.

Bun, since version 1.x, ships all five as a single binary with zero-config defaults: native TypeScript, built-in test runner, `bun install`, native `.env` loading, and `bun build`.

## Decision

**Bun is the only toolchain.** OOR uses `bun test`, `bun install`, `bun build`, and direct `bun ./script.ts` execution. No `node`, no `ts-node`, no `npm`/`pnpm`, no `jest`, no `webpack`/`tsup`.

## Consequences

### Positive

- One binary, one install. New contributors run `bun install` and the project works end-to-end.
- Native TypeScript: no `tsconfig.json` plumbing for execution (`tsconfig.json` is still the source-of-truth for strictness, but Bun reads it directly).
- Speed: `bun install` and `bun test` are noticeably faster than the npm/jest equivalents.
- Single source of truth for `.env` loading.
- Bun's `SQL` driver is what powers OOR's PostgreSQL access — the toolchain choice and the runtime choice reinforce each other.

### Negative / trade-offs

- Bun is younger than the Node ecosystem. Edge-case incompatibilities exist and are encountered rarely but disruptively.
- Some libraries assume `node` and trip on Bun's APIs. Most are dropping support gaps over time, but the risk is non-zero.
- A consumer of OOR doesn't *have* to use Bun (the published npm package is portable) — but contributors do.

### Neutral

- The toolchain choice is partially coupled to ADR 0006 (TDD): `bun test` is what makes the test loop fast enough to be habitual.
- And to the use of Bun's native `SQL` driver for PostgreSQL — which is the runtime backbone of [[Repository Pattern]] execution.

## Alternatives Considered

- **Node + tsx + pnpm + vitest** — rejected: four binaries, four configs, no real win over Bun for this project's scale.
- **Deno** — rejected: even more divergent from npm conventions; would complicate publishing to npm.

## References

- `.raw/welcome.md` § "Bun as the toolchain"
- [[Bun]]
