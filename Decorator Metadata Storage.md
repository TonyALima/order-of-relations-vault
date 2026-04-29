# Decorator Metadata Storage

A decorator-driven ORM lives or dies by where it puts the data the decorators collect. This note pins down the choices behind OOR's `MetadataStorage`: what gets recorded, how the decorators write into it, when it gets resolved, and why the shape ended up the way it did.

For the higher-level "no `reflect-metadata`" stance, see [[Welcome]]. This note covers the consequences of that stance in code.

---

## What's stored

Three things, and only three:

- **Entity metadata** — table name, an optional discriminator (for single-table inheritance), and the column/relation arrays the entity contributes.
- **Column metadata** — property name, mapped column name, SQL type, primary flag, nullability, and an optional `autogeneration` strategy.
- **Relation metadata** — property name, relation type (`TO_ONE` for now), nullability, foreign-key columns (resolved lazily), and a thunk `getTarget()` that returns the related entity's constructor.

Anything else — schema names, indexes, cascading rules — is deliberately absent. They'd belong on the same record, but they aren't needed yet, and an empty field is harder to remove later than to add.

The `getTarget` thunk is the one piece worth flagging: relations are declared by **constructor reference inside a closure**, not by string name. It defers resolution past the temporal-dead-zone problems that show up with circular entity graphs.

## The storage shape

```ts
class MetadataStorage {
  private storage = new Map<Constructor, EntityMetadata>();
  private isMetadataResolved = false;
  ...
}
```

A single `Map<Constructor, EntityMetadata>`, keyed by the class constructor itself.

Why a `Map` and not per-class static fields?

- Per-class statics would scatter metadata across user code; a `Map` keeps it owned by the library.
- Iteration over all registered entities is a first-class operation (`[Symbol.iterator]` is part of the public surface — it's what the schema generator and inheritance resolver walk).
- Lookups are O(1) by reference equality on the constructor, which is the natural identity for an entity class.

Why not a `WeakMap`? Because OOR explicitly **wants** to keep entity classes alive for the lifetime of the database connection — the entity registry *is* the schema. A weak reference would be a bug, not an optimisation.

`MetadataStorage` is owned by `Database` (`db.getMetadata()`) rather than by a process-global singleton. Tests build fresh storages constantly; a global would make that miserable.

## How decorators write into it

Stage-3 decorators take `(value, context)` and stash side-effects on `context.metadata` — an object the runtime threads through every decorator on the same class. OOR uses **two symbols** as keys on that object:

```ts
const COLUMNS_KEY = Symbol('columns');
const RELATIONS_KEY = Symbol('relations');
```

Field decorators (`@Column`, `@PrimaryColumn`, `@ToOne`) push entries onto arrays under those keys. The class decorator (`@Entity`) reads them and commits the result to the storage:

```ts
export function Entity(db: Database, mapTableName?: string) {
  return function <T extends Constructor>(value: T, context: ClassDecoratorContext<T>) {
    const tableName = mapTableName ?? String(context.name);
    const columns = (context.metadata[COLUMNS_KEY] as ColumnMetadata[]) ?? [];
    const relations = (context.metadata[RELATIONS_KEY] as RelationMetadata[]) ?? [];

    if (!columns.some((c) => c.primary)) throw new MissingPrimaryColumnError(String(context.name));

    db.getMetadata().set(value, { tableName, columns, relations });
  };
}
```

Two important properties of the Stage-3 timing model are doing real work here:

1. **Field decorators run before the class decorator.** By the time `@Entity` executes, the column and relation arrays are already populated, so it can validate (e.g., "must have a primary column") at decoration time rather than first-use time.
2. **`context.metadata` is shared across all decorators on the same declaration**, so `@Column` and `@ToOne` can hand off to `@Entity` without an external registry, a class hierarchy walk, or a side channel.

## Inheritance

Stage-3 decorators give each subclass a fresh `context.metadata` — it does **not** inherit the parent's metadata object. That's a feature, not a workaround: each class records what it *introduces*, and resolution happens later.

`MetadataStorage` resolves single-table inheritance lazily, on the first `get()` (or first iteration) after a `set()`:

- `getParentTableName` walks `Object.getPrototypeOf(target.prototype)?.constructor` up to the topmost decorated ancestor and adopts its table name. So `AdminUser extends User` ends up writing to the `users` table.
- A `discriminator` is assigned per class — by default, the class's own declared table name. If only one class maps to a given table, the discriminator is wiped. So in the `inheritance` example, `User` sits alone and has no discriminator; once `AdminUser` is registered, both pick up `users` for the table and `'users'` / `'admin_users'` as discriminators.
- Multi-level chains (`Base ← User ← AdminUser`) collapse to the topmost ancestor's table.

Resolution is gated by `isMetadataResolved` and reset on every `set()`, so it's idempotent across additions. The tests in `metadata.test.ts` ("re-resolving after a new entity is added does not corrupt discriminator") pin this contract.

Note that **column inheritance falls out for free** from prototype lookups at runtime — Stage-3 doesn't merge column arrays across the hierarchy, but the parent's metadata is still in the storage under the parent constructor, and the inheritance resolver reads it from there.

## Why not `reflect-metadata`

The high-level reason lives in [[Welcome]]. The concrete consequences inside this file:

- **No `experimentalDecorators` and no `emitDecoratorMetadata`** in `tsconfig.json`. The decorator semantics OOR relies on are the standard Stage-3 ones the runtime ships natively.
- **No `Reflect.getMetadata` / `Reflect.defineMetadata` calls anywhere.** Metadata reads always go through `db.getMetadata().get(ctor)`. There is exactly one path in and one path out.
- **No global polyfill.** `reflect-metadata` patches the global `Reflect` object on import, which is the kind of side effect a library has no business inflicting on its host. Owning the `Map` ourselves keeps OOR a pure import.
- **Runtime control of *when* metadata is written.** With `reflect-metadata` the type emit happens at decoration time and there's no clean hook into it; with our own `Map` the library decides what to record and when, and resolution can be lazy without fighting the runtime.

## Failure modes

`metadata.errors.ts` defines exactly one error today:

- `RelationTargetNotFoundError` — thrown during relation resolution when a relation's `getTarget()` returns a constructor that was never registered with `@Entity`. This is the most common shape of "I forgot to import the related entity, so its decorator never ran" — by the time the user calls `storage.get(Post)`, `User` is missing from the map, and the error fires with both the target class name and the `posts.author` path.

Two adjacent failure paths live elsewhere:

- `MissingPrimaryColumnError` is raised by `@Entity` itself when a class registers no `@PrimaryColumn`. Caught at decoration time, before anything reaches the storage.
- `storage.get(UnregisteredClass)` returns `undefined` rather than throwing. The repository layer is responsible for translating that into a user-facing error, since "no metadata" can mean either "you forgot `@Entity`" or "you passed the wrong class", and the metadata layer can't tell them apart.

See also [[Repository Contract]] and [[Query Builder Design]] for how this metadata is consumed downstream.
