---
type: domain
title: "Decisions (ADRs)"
subdomain_of: ""
page_count: 0
created: 2026-04-29
updated: 2026-04-29
tags:
  - meta
  - decisions
status: seed
related:
  - "[[index]]"
sources: []
---

# Decisions (ADRs)

Architecture Decision Records. Each ADR captures a *load-bearing* design call and its rationale. New decisions are appended; superseded decisions are marked with `status: superseded` and a link to the replacement.

> [!gap] No ADRs ingested yet
> `.raw/welcome.md` already lists six in-flight decisions worth promoting to ADRs:
> 1. Stage-3 decorators (no `reflect-metadata`)
> 2. Repository pattern with lazy query builder
> 3. DI container as singleton
> 4. SQL safety — parameterized only, no `sql.unsafe`
> 5. Type-driven API — strict no-`any`
> 6. Bun as the toolchain
> 7. TDD as the implementation rhythm

## Decision pages

- _none yet_

## ADR naming convention

`NNNN-short-title.md` where `NNNN` is a zero-padded sequence: `0001-stage-3-decorators.md`, `0002-no-reflect-metadata.md`, ...

## Frontmatter for ADR pages

```yaml
type: decision
status: accepted        # proposed | accepted | superseded | deprecated
date: YYYY-MM-DD
deciders: ["Tony Albert"]
supersedes: []
superseded_by: ""
context: ""
```
