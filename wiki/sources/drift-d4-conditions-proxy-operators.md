---
type: source
title: "Drift D4 — FieldConditionBuilder operator inventory"
source_type: drift-correction
author: "Tony Albert"
date_published: 2026-04-29
url: ""
confidence: high
key_claims:
  - "FieldConditionBuilder<V> exposes exactly nine methods: eq, ne, gt, gte, lt, lte, isNull, isNotNull, in."
  - "Every column gets the same nine methods regardless of value type V — there is no per-type operator narrowing."
  - "The compile-time guarantee is on the value parameter, not on which operators are available."
  - "before, after, matches, like — none exist anywhere in the codebase."
  - "The method is `ne`, not `neq`. The original .raw/ source had it right; the synthesis pages introduced the drift."
created: 2026-04-29
updated: 2026-04-29
tags:
  - source
  - drift
status: stable
related:
  - "[[Conditions Proxy]]"
  - "[[QueryBuilder]]"
  - "[[Lazy Query Builder]]"
sources:
  - "../.raw/drift-d4-conditions-proxy-operators.md"
---

# Drift D4 — FieldConditionBuilder operator inventory

## Summary

[[Conditions Proxy]] listed operators that don't exist (`before`, `after`, `matches`) and claimed a per-column-type operator narrowing the code does not implement. The page's overall framing — proxy mechanism, `Partial` rationale, runtime `UndefinedWhereConditionError` catch — is correct. Only the operator inventory and the "type-safe operators" claim needed fixing.

## What the Code Actually Exposes

`src/query-builder/types.ts`:

```ts
export interface FieldConditionBuilder<V> {
  eq(value: V): Condition;
  ne(value: V): Condition;
  gt(value: V): Condition;
  gte(value: V): Condition;
  lt(value: V): Condition;
  lte(value: V): Condition;
  isNull(): Condition;
  isNotNull(): Condition;
  in(values: V[]): Condition;
}
```

Nine methods, available on **every** column regardless of `V`. The closed `Condition['op']` enum:

```
'=' | '!=' | '>' | '>=' | '<' | '<=' | 'IS NULL' | 'IS NOT NULL' | 'IN'
```

## What the Wiki Got Wrong

1. *"Each builder carries the operators valid for the column's type — `.eq`, `.neq`, `.in`, `.lt`, `.gt`, etc."* — Wrong on two counts: the method is `ne`, not `neq`; and there is no per-type narrowing.
2. *"A `Date` column's `FieldConditionBuilder` exposes `.before` / `.after` but not `.matches`; the type system rejects the wrong operator at compile time."* — None of `before`, `after`, `matches` exist. The example `u.createdAt!.after(new Date('2026-01-01'))` won't compile.

## What Type-Safety Actually Means Here

The compile-time guarantee is on the **value type**, not on which operators are available. `gt(v: V)` rejects a value whose type doesn't match `V` — so `u.email!.eq(42)` fails when `email: string`. But a `Date` column and a `number` column expose the same nine methods. Per-operator narrowing per column type would require lifting column TS type into the proxy at the type level and conditionally picking method sets — doable with mapped + conditional types, but not done. "Future ergonomics" is honest; "shipped" was wrong.

## Why the Drift Mattered

The original `.raw/query-builder-design.md` already had this right (`ne`, nine operators). The drift was introduced during synthesis — a transcription error that propagated to multiple wiki pages. Cross-checking synthesis against the raw caught it.

## Resolution

- [[Conditions Proxy]] — operator inventory bullet replaced with the nine-method list; "Type-safe operators" replaced with "Type-safe values, uniform operator surface"; `before` / `after` example replaced with `gt(new Date(...))`.

## Source Citations

- `src/query-builder/types.ts` lines 1–17 — Condition['op'] enum + FieldConditionBuilder<V>.
- `src/query-builder/query-builder.ts` lines 112–129 — buildConditionsProxy.
- `src/query-builder/query-builder.ts` lines 73–96 — opFragments + IN composition (incl. empty-array case).
- `.raw/query-builder-design.md` — the correct original.
