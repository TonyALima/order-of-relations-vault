---
type: domain
title: "Flows"
subdomain_of: ""
page_count: 4
created: 2026-04-29
updated: 2026-04-29
tags:
  - meta
  - flows
status: developing
related:
  - "[[index]]"
sources: []
---

# Flows

End-to-end sequences. A flow page walks through a single observable behavior step by step — *what happens when the user does X*. Distinct from `concepts/` (which defines what a thing is) and `modules/` (which describes a chunk of code).

## Flow pages

- [[entity-registration]] — decorator evaluation order; field decorators populate `context.metadata` under three symbol keys, `@Entity` validates and commits to `MetadataStorage`, first read triggers lazy resolution.
- [[query-lifecycle]] — six-step walkthrough of `Repository.findMany` (Repository entry → metadata → conditions proxy → validation → SQL composition → execution).
- [[lifecycle-of-a-create]] — seven-step walkthrough of `Repository.create` (type-check → metadata → `requirePrimaryKey` gate → autogeneration → INSERT-with-RETURNING → return PK).
- [[schema-create-drop]] — two-pass `CREATE TABLE` (tables, then `ALTER TABLE ADD FOREIGN KEY`); topologically reversed `DROP`.

> [!gap] Still to come
> `migration-apply.md` — pending a future schema-migrations source.
