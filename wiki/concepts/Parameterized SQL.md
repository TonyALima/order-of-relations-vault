---
type: concept
title: "Parameterized SQL"
complexity: intermediate
domain: "Database security"
aliases:
  - "Prepared statements"
  - "Bound parameters"
created: 2026-04-29
updated: 2026-04-29
tags:
  - concept
  - sql
  - security
status: seed
related:
  - "[[0004-parameterized-sql-only]]"
  - "[[sources/welcome]]"
sources:
  - "../.raw/welcome.md"
---

# Parameterized SQL

## Definition

**Parameterized SQL** separates the query *shape* (a string with placeholders) from its *values* (passed alongside as a discrete array). The database driver — not string concatenation — is responsible for binding values into the query plan, which makes SQL injection impossible by construction for the parameterized parts.

In Bun's `SQL` driver this is the tagged-template form `sql\`SELECT * FROM users WHERE id = ${id}\``. The interpolation produces a placeholder + a bound parameter, never a substring of the SQL text.

## How It Works

When the driver receives `sql\`SELECT * FROM users WHERE id = ${id}\``, it:

1. Builds the SQL text `SELECT * FROM users WHERE id = $1`.
2. Builds a parameter array `[id]`.
3. Sends both to PostgreSQL via the wire protocol.
4. PostgreSQL plans the query with `$1` as a placeholder and binds the value at execution time.

The user-supplied value is **never** part of the SQL text the planner sees. Even a value containing `'; DROP TABLE users;--` is treated as a single literal, not as SQL syntax.

The escape hatch — Bun's `sql.unsafe(text)` — bypasses this entirely by emitting `text` directly into the SQL string. OOR forbids `sql.unsafe` library-wide (see [[0004-parameterized-sql-only]]).

## Why It Matters

- **Injection-free by construction.** No interpolation = no injection. This is the only complete defense; everything else (escaping, allowlists, regex filters) has known bypasses.
- **Plan caching.** Parameterized queries with the same shape can share a cached plan in PostgreSQL, which can be a measurable win at scale.
- **Audit clarity.** A reviewer scanning a diff can confirm "no `sql.unsafe`" mechanically; "is this string interpolation safe?" requires per-call reasoning.

## Examples

Safe, the only allowed form in OOR:

```ts
const id = req.body.id; // user input
const rows = await sql`SELECT * FROM users WHERE id = ${id}`;
```

Forbidden in OOR:

```ts
// Banned: sql.unsafe is not used anywhere in the codebase.
const rows = await sql.unsafe(`SELECT * FROM users WHERE id = ${id}`);
```

Constraint: identifiers (table names, column names) cannot be parameterized in standard SQL — they are not values. When a query needs a dynamic identifier, it must come from an allowlist or a quoting helper, never raw user input.

## Connections

- [[0004-parameterized-sql-only]] — the ADR fixing the rule.
- [[Lazy Query Builder]] — the builder that has to produce parameterized SQL at terminal compile time.
- [[Bun]] — the runtime providing the `SQL` driver.

## Sources

- `.raw/welcome.md` § "SQL safety: parameterized only"
