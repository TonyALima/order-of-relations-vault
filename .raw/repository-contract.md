# Repository Contract

`Repository<T>` is the entity's persistence boundary. The README documents its signatures; this note is about what callers can rely on, what the type system checks at compile time, what is checked at runtime, and how it fails.

The headline rule: a `Repository<T>` is *narrow*. It owns single-row operations against one entity. It does not compose queries (that's the [[Query Builder Design]]), it does not wire services (that's the DI container), and it does not evolve schema (see [[Schema Migrations]]). When an operation has a static answer — "is this row reachable by primary key?" — the repository handles it. Anything else returns a builder and walks away.

---

## Operations and what each one owns

The class exposes:

- `create(entity)` — insert one row, return the primary key.
- `findById(key)` — locate a row by its full primary key.
- `findOne(options)` / `findMany(options)` — short-circuit through the builder for trivial filters.
- `find()` — *not* a terminal call: hands back a `QueryBuilder<T>`.
- `update(entity)` — overwrite a row identified by its primary key.
- `delete(key)` — remove a row by its primary key.

Every method that needs a primary key goes through one private gate, `requirePrimaryKey`, which decides what counts as "complete" depending on whether the column has `autogeneration` declared. That single gate is why the runtime story below is symmetric across `create`, `findById`, `update`, and `delete`.

---

## Compile-time guarantees on `create()`

The big change in commit `92ce5fd` was tightening `create(entity: Partial<T>)` to `create(entity: T)`. The signature now refuses partial inputs, and the entity's own field declarations carry the burden of saying what is and isn't optional.

Concretely, the entity-side decorators shape `T` so that `create()`'s argument type already encodes the contract:

- `@NotNullable` fields are required on `T`. Omitting them is a compile error.
- `@Nullable` fields are optional on `T`. Omitting them is fine.
- `@PrimaryColumn({ autogeneration })` makes the field optional (`NullableField<Value>`) — the caller may omit it, but if they do supply one, it wins.
- `@PrimaryColumn` *without* `autogeneration` keeps the field required (`NotNullableField<Value>`). The caller must provide it.

The cleanest way to read the rule: *required* on the type means *required* at `create()`. The only escape hatch is `autogeneration`, which marks the column "the library will fill this in if you don't."

```ts
class CompileUser {
  @PrimaryColumn({ type: COLUMN_TYPE.INTEGER }) id!: number;
  @Column({ type: COLUMN_TYPE.TEXT }) @NotNullable name!: string;
  @Column({ type: COLUMN_TYPE.TEXT }) @NotNullable email!: string;
}

// @ts-expect-error email required
repo.create({ id: 1, name: 'Alice' });
```

The `@ts-expect-error` tests in `repository.test.ts` (and the broader `nullable` suite that landed in `648d9fa`) are pinning down exactly this — every comment is a compile-time invariant the library has promised not to weaken. See [[Decorator Metadata Storage]] for how `@Nullable` / `@NotNullable` write into the same metadata bucket the column decorators read.

What `create()` does *not* check at compile time: relations and FK shape. Those still go through runtime metadata.

---

## Runtime guarantees on `create()`

`create()` returns `Partial<T>` containing exactly the primary-key fields, populated from `RETURNING`. It does not hand back a hydrated entity — if you want the full row, follow with `findById()`. This is deliberate: `create()`'s job is to commit a row and tell you how to find it again, not to re-read it.

For a column with `autogeneration`:

- `clientSide`: the library calls the function once before the INSERT, writes the value into both the SQL statement and the returned key.
- `dbSide`: the column is omitted from the INSERT entirely; the database fills it, and the value comes back via `RETURNING`.
- An explicit caller value *always* wins over either strategy. Passing `id` to a SERIAL or UUID column is a supported override (see commit `cfe4cc5`).

If a primary-key column has no `autogeneration` and the caller omits it, `create()` throws `IncompletePrimaryKeyError` — the same error path the read/write methods use.

---

## Primary key behaviour: explicit `autogeneration` only

Commit `3aa354b` removed an older convenience: SERIAL columns no longer auto-generate implicitly just because of their type. Auto-generation is now an explicit metadata flag (`autogeneration: { dbSide: ... }` or `{ clientSide: ... }`).

Two reasons for the change:

1. The column *type* and the *generation strategy* are independent dimensions. A `text` UUID column might be client-generated; an `integer` PK might or might not have a sequence. Conflating type with strategy was a leak.
2. The compile-time `create()` story above only works because the metadata explicitly says "this column is omittable." A SERIAL with no `autogeneration` is now a perfectly normal required field at the type level, and the runtime gate agrees: `create({ name: 'x' })` on such an entity is both a compile error and an `IncompletePrimaryKeyError` at runtime if the type check is bypassed.

Composite primary keys follow the same rule per column. Each PK column independently decides whether it's omittable.

---

## `find()` is a handoff

`find()` does not run SQL. It returns a `QueryBuilder<T>` and steps out of the way. Everything about clause accumulation, lazy execution, and join shape lives in [[Query Builder Design]].

`findOne()` and `findMany()` are sugar — they construct a builder, apply `FindOptions`, and call the terminal method for you. Use them when the call site doesn't need to compose; reach for `find()` when it does.

---

## Failure modes

The repository throws exactly one error of its own: `IncompletePrimaryKeyError`. It is raised by:

- `findById(key)` when any PK property is absent from `key`.
- `delete(key)` likewise.
- `create(entity)` when a non-autogenerated PK property is absent.
- `update(entity)` indirectly, when `requirePrimaryKey` runs.

The error carries `entityName` and `missingProperties: string[]`, both populated and asserted by tests. Anything else — connection failure, constraint violation, type mismatch — propagates from the underlying SQL driver unchanged. The repository is intentionally thin around the driver's own errors; wrapping them would obscure the `pg` / Bun SQL diagnostic.

---

## What the contract deliberately leaves out

- **No partial updates.** `update()` takes a full `T`. Field-level merge semantics belong in a builder, not a repository.
- **No identity map.** Two calls to `findById({ id: 1 })` may return two distinct objects. Equality is by value, not by reference.
- **No cascade.** Relations are written when present in the input object and ignored otherwise; the repository never reaches into another table on its own.
- **No transaction management.** Transactions are the caller's job (or a future concern) — a repository takes a `Database` and uses whatever connection that database hands it.

These are not gaps; they are the boundary. Anything composed, reactive, or stateful goes elsewhere.
