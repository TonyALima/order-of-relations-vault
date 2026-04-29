# Drift Correction — `FieldConditionBuilder` operator inventory and per-type narrowing

`wiki/concepts/Conditions Proxy.md` lists operators that don't exist, and claims a per-column-type narrowing that the code does not implement. The errors are localised — the page's overall framing (proxy mechanism, `Partial` rationale, runtime `UndefinedWhereConditionError` catch) is correct and worth keeping. Only the operator inventory and the "type-safe operators" claim need fixing.

## What the code actually exposes

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

That's it. Nine methods, available on **every** column regardless of type. The proxy is built by reflecting over the column metadata in `query-builder.ts`'s `buildConditionsProxy()`:

```ts
proxy[col.propertyName as keyof T] = {
  eq: (value)  => ({ columnName: col.columnName, op: '=',   value }),
  ne: (value)  => ({ columnName: col.columnName, op: '!=',  value }),
  gt: (value)  => ({ columnName: col.columnName, op: '>',   value }),
  gte: (value) => ({ columnName: col.columnName, op: '>=',  value }),
  lt: (value)  => ({ columnName: col.columnName, op: '<',   value }),
  lte: (value) => ({ columnName: col.columnName, op: '<=',  value }),
  isNull:    () => ({ columnName: col.columnName, op: 'IS NULL'     }),
  isNotNull: () => ({ columnName: col.columnName, op: 'IS NOT NULL' }),
  in: (values) => ({ columnName: col.columnName, op: 'IN', value: values }),
};
```

The corresponding closed `Condition['op']` enum is:

```
'=' | '!=' | '>' | '>=' | '<' | '<=' | 'IS NULL' | 'IS NOT NULL' | 'IN'
```

Every builder has every method. The compile-time narrowing is **only on the value parameter** — `FieldConditionBuilder<V>` is generic over `V`, so `u.age!.gt(18)` rejects `gt('eighteen')`. There is no narrowing on which *operators* are exposed for a given `V`. A `Date` column has the same `FieldConditionBuilder` shape as a `string` column, exposing `gt`, `gte`, `lt`, `lte`, `eq`, `ne`, `in`, `isNull`, `isNotNull`. There is no `before`, no `after`, no `matches`, no `like`.

## The two specific wiki claims that are wrong

1. *"Each builder carries the operators valid for the column's type — `.eq`, `.neq`, `.in`, `.lt`, `.gt`, etc."* — wrong on two counts:
   - The method is `ne`, not `neq`.
   - The "operators valid for the column's type" framing implies per-type narrowing that doesn't exist.

2. *"A `Date` column's `FieldConditionBuilder` exposes `.before` / `.after` but not `.matches`; the type system rejects the wrong operator at compile time."* — `before` / `after` / `matches` do not exist anywhere in the codebase. The example `u.createdAt!.after(new Date('2026-01-01'))` won't compile against the real type.

## What the page should say instead

The corrected operator section, mirroring the actual nine methods:

> Each `FieldConditionBuilder<V>` exposes the same nine methods regardless of `V`:
>
> - **Comparison:** `eq(v)`, `ne(v)`, `gt(v)`, `gte(v)`, `lt(v)`, `lte(v)` — produce `=`, `!=`, `>`, `>=`, `<`, `<=` respectively.
> - **Null tests:** `isNull()`, `isNotNull()` — produce `IS NULL` / `IS NOT NULL`.
> - **Set membership:** `in(values: V[])` — produces `IN (...)`. An empty array is a valid no-match `IN ()`; the test suite pins this.
>
> The compile-time guarantee is on the **value type**, not on which operators are available: `gt(v: V)` rejects a value whose type doesn't match `V`. There is no per-column-type narrowing on operator availability — a `Date` column and a `number` column expose the same nine methods. (A future API could narrow further, e.g. exposing string-specific `like` or date-specific `between` only on the right column types, but today the surface is uniform.)

The "no-`any`" claim on the same page is still correct — it follows from the generic-on-`V` typing, not from any per-type narrowing.

## Why per-type narrowing isn't there (worth a sentence in the wiki)

The codebase forbids `any` (`[[0005-no-any-type-driven-api]]`), and the proxy is type-safe in the sense that `u.email!.eq(42)` is rejected when `email: string`. Per-operator narrowing per column type is a *more ambitious* type-system feature — it would require the column metadata to lift the column's TS type into the proxy at the type level, and then conditionally pick the available method set based on it. That's doable (mapped + conditional types) but not done. Calling it "future ergonomics" is honest; calling it shipped is wrong.

## Wiki pages affected

| Page | Action |
|---|---|
| `wiki/concepts/Conditions Proxy.md` | (1) Fix the operator inventory bullet — replace the `eq/neq/in/lt/gt/etc` line with the nine-method list above. (2) Replace the "Type-safe operators" bullet with the corrected "Type-safe values, uniform operator surface" framing. (3) Replace the `before` / `after` example with actual API: e.g. `u.createdAt!.gt(new Date('2026-01-01'))`. The other examples on the page (`u.email!.eq('a@b.com')`, the `u.active!.eq(true)` / `u.deletedAt!.eq(null)` example shape from `[[Lazy Query Builder]]`) are fine as-is. |
| `wiki/components/QueryBuilder.md` | The "Type Flow" section is correct. No change needed except possibly an explicit listing of the nine methods so callers don't have to bounce to the Conditions Proxy page. |
| `wiki/concepts/Lazy Query Builder.md` | The example on line 75 — `where: (u) => [u.active!.eq(true), u.deletedAt!.eq(null)]` — is fine. No change. |
| `wiki/sources/query-builder-design.md` | The original raw at `.raw/query-builder-design.md` already says *"`eq`, `ne`, `gt`, `in`, `isNull`, …"* (note: `ne`, not `neq`) — that source is right. Cross-check the synthesized page against the raw and the code; the drift is in the synthesis pages, not the raw. |

## Source citations

- `src/query-builder/types.ts` lines 1–17 — `Condition['op']` enum and `FieldConditionBuilder<V>` interface.
- `src/query-builder/query-builder.ts` lines 112–129 — `buildConditionsProxy` implementation showing every column gets the same nine methods.
- `src/query-builder/query-builder.ts` lines 73–96 — the `opFragments` static map and the SQL composition for `IN` (including the empty-array case).
- `.raw/query-builder-design.md` — the original source uses the correct method name `ne` and lists nine operators; the synthesized wiki pages introduced the drift.
