---
type: flow
title: "Entity Registration"
trigger: "TypeScript evaluates a class declaration decorated with @Entity(db)"
outcome: "Database's MetadataStorage contains an EntityMetadata for that constructor"
created: 2026-04-29
updated: 2026-04-29
tags:
  - flow
  - decorators
  - metadata
status: stable
related:
  - "[[MetadataStorage]]"
  - "[[ECMAScript Stage-3 Decorators]]"
  - "[[Database]]"
  - "[[sources/decorator-metadata-storage]]"
sources:
  - "../.raw/decorator-metadata-storage.md"
---

# Entity Registration

The end-to-end sequence that turns a decorated TypeScript class into an `EntityMetadata` entry in a `Database`'s [[MetadataStorage]]. This is the orthogonal flow to [[query-lifecycle]]: that one runs every query; this one runs **once per entity** at module load.

## The Sequence

### Step 1 — Field decorators run

The Stage-3 spec evaluates **field decorators before class decorators**, and within a single field, decorators apply **bottom-up** (the one closest to the property runs first).

Three private symbol keys are used on `context.metadata` — Stage-3's per-class metadata bag — each with its own shape:

```ts
const COLUMNS_KEY   = Symbol('columns');    // ColumnMetadata[]
const RELATIONS_KEY = Symbol('relations');  // RelationMetadata[]
const NULLABLE_KEY  = Symbol('nullable');   // Map<string, boolean>
```

- `@Nullable` / `@NotNullable` set `NULLABLE_KEY[propertyName] = true | false`. They are **inner decorators** — they must run before `@Column` on the same property.
- `@Column` reads `NULLABLE_KEY[propertyName]`, throws `MissingNullabilityDecoratorError` if it's `undefined`, and otherwise pushes a `ColumnMetadata` (with the resolved `nullable` baked in) onto `COLUMNS_KEY`.
- `@PrimaryColumn` skips the `NULLABLE_KEY` check entirely (primary columns are always `nullable: false`) and pushes onto `COLUMNS_KEY` with `primary: true`.
- `@ToOne` pushes a `RelationMetadata` onto `RELATIONS_KEY`. The relation's target is supplied as a [[Relation Target Thunk|closure]] (`() => User`) rather than a direct reference, so circular entity graphs don't trip the temporal dead zone.

`context.metadata` is fresh per class — subclass declarations do **not** inherit the parent's bag.

> [!warning] Decorator-order constraint on every column
> Because `@Column` reads `NULLABLE_KEY` and throws if the entry is missing, `@Nullable` (or `@NotNullable`) must be applied **before** `@Column`. In Stage-3 syntax, that means `@Nullable` is the **inner** decorator — closer to the property:
>
> ```ts
> @Column({ type: COLUMN_TYPE.TEXT })
> @Nullable                              // <-- inner; runs first
> nickname?: string;
> ```
>
> Reversing them throws `MissingNullabilityDecoratorError` at decoration time. `@PrimaryColumn` is exempt.

### Step 2 — Class decorator runs

`@Entity(db, mapTableName?)` is the class decorator. By the time it executes, all field decorators have already populated `context.metadata`.

Its body, paraphrased:

```ts
export function Entity(db: Database, mapTableName?: string) {
  return function <T extends Constructor>(value: T, context: ClassDecoratorContext<T>) {
    const tableName = mapTableName ?? String(context.name);
    const columns = (context.metadata[COLUMNS_KEY] as ColumnMetadata[]) ?? [];
    const relations = (context.metadata[RELATIONS_KEY] as RelationMetadata[]) ?? [];

    if (!columns.some((c) => c.primary)) {
      throw new MissingPrimaryColumnError(String(context.name));
    }

    db.getMetadata().set(value, { tableName, columns, relations });
  };
}
```

Three things happen here:

1. Resolve the table name — defaults to `context.name`.
2. **Validate** — at least one column must be primary. If not: `MissingPrimaryColumnError` is thrown *at decoration time*, before the storage is touched. The class never lands in the registry.
3. Commit — `db.getMetadata().set(value, { tableName, columns, relations })`.

`@Entity` does **not** read `NULLABLE_KEY`. That bucket has already been consumed by `@Column` during Step 1 — the resolved `nullable` field is baked into each `ColumnMetadata` entry. From `MetadataStorage`'s viewpoint, only two buckets exist; from `context.metadata`'s viewpoint, three did.

### Step 3 — Storage flips its resolution flag

`MetadataStorage.set()` resets `isMetadataResolved = false`. No resolution work happens yet — the flag just signals that the next read needs to recompute.

### Step 4 — First read triggers resolution

The first `db.getMetadata().get(SomeEntity)` (or any iteration) after a `set()` runs:

- `resolveInheritance` — walks `Object.getPrototypeOf(target.prototype)?.constructor` for each registered class, adopts the topmost decorated ancestor's table name, and assigns / wipes discriminators. See [[Single-Table Inheritance]].
- `resolveRelations` — for each relation, calls its `getTarget()` thunk and looks up the result in storage. Throws `RelationTargetNotFoundError` if the target isn't registered. Foreign-key column names that decorators left as `null` are filled in here.

After resolution, `isMetadataResolved = true`. Subsequent reads hit the cache.

## Failure Modes

| Where | What | Outcome |
|---|---|---|
| Step 1 (`@Column`) | `@Nullable` / `@NotNullable` not applied (or applied in the wrong order — outside `@Column`) | `MissingNullabilityDecoratorError` thrown at decoration time. `@PrimaryColumn` is exempt. |
| Step 2 | Class has no `@PrimaryColumn` | `MissingPrimaryColumnError` thrown at decoration time; class never registered |
| Step 4 | Relation target's constructor never registered with `@Entity` | `RelationTargetNotFoundError` with target name + relation path (`posts.author`) |
| (Outside this flow) | `storage.get(SomeClass)` for an unregistered class | Returns `undefined`. The metadata layer refuses to throw because it can't disambiguate "forgot `@Entity`" vs "passed the wrong class" — translation is the repository's job |

## Why this ordering is enforceable

Stage-3's evaluation order — **field decorators before class decorators** — is what makes the validation in Step 2 possible. By the time `@Entity` runs, the column array is already complete; "must have a primary column" is decidable now, not at first-use time.

If the language ever changed that ordering (it won't), this flow would have to invert: validation would defer to first read, errors would surface much later, and a misdeclared entity could pollute the storage before being caught.

## See Also

- [[MetadataStorage]] — the destination of step 2's `set()`.
- [[ECMAScript Stage-3 Decorators]] — the language semantics (`context.metadata`, evaluation order).
- [[Single-Table Inheritance]] — what step 4's `resolveInheritance` does.
- [[Relation Target Thunk]] — why the relation target is a closure.
- [[query-lifecycle]] — the per-query flow that consumes the resolved metadata.

## Sources

- `.raw/decorator-metadata-storage.md` §§ "How decorators write into it", "Failure modes"
