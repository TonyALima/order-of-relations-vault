# Query Builder Design

This is a design note, not an API reference. The README covers how to *call* the builder; this page is about *why it has the shape it does*. The headline decision — "lazy query builder" — is already taken in [[Welcome]]; here I unpack what that decision actually buys, and where the seams between the builder, the [[Repository Contract]], and the parameterized SQL layer fall.

---

## Lazy by design

`Repository.findMany()` and `Repository.findOne()` both internally construct a `QueryBuilder<T>`, run `applyOptions()`, and call a terminal method. The builder itself does not execute SQL on instantiation, on `applyOptions()`, or on any clause-adding helper. Execution happens only inside `getMany()` / `getOne()`.

That split exists for three reasons:

- **Composition.** The builder can be passed to a helper that conditionally adds `inheritance` or extra conditions. If `find()` ran SQL eagerly, every helper would have to either re-issue the query or accept already-fetched rows — both worse.
- **Single round-trip.** All clauses collapse into one SQL statement. There is no chain of intermediate `SELECT`s, no client-side filtering after the fact.
- **Deferred decisions.** The discriminator strategy for inheritance (`SUBCLASSES` vs `ONLY`) is resolved at terminal time, not at construction. That means subclass discovery walks the metadata map *once*, when the SQL is actually about to be issued, against a metadata snapshot that's already stable.

The cost is one extra call (`.getMany()` / `.getOne()`) compared to a hypothetical `findAll(options)` that just returned `Promise<T[]>`. The repository absorbs that cost on the caller's behalf for the common cases — see [[Repository Contract]].

---

## Clause accumulation

Internally the builder holds a single piece of mutable state:

```ts
private conditions: Condition[] = [];
```

That's it today. Where-conditions, the inheritance discriminator filter, and (eventually) ordering and pagination all reduce down to entries in this array — or, for pieces that don't fit, would each get their own dedicated field.

The builder is **mutable**, not immutable-by-copy. `applyOptions()` returns `this`, and the discriminator setters push directly onto `conditions`. I picked mutable for two reasons:

- The builder is a short-lived, single-owner object. The repository constructs it, hands it to one terminal call, and lets it go. There is no scenario today where two consumers hold the same builder and expect divergent state.
- Immutable builders pay for safety I don't need with allocations on every clause. If a future use case shows up where sharing matters (e.g. caching a partial query), I'd prefer to introduce a `clone()` rather than retrofit copy-on-write into every method.

If you call `applyOptions()` twice, the second call **replaces** the where-conditions wholesale (`this.conditions = results`). That's a deliberate "last call wins" — partial composition of where-clauses is a future feature, not an accidental one.

---

## The `where` callback shape

The single most opinionated piece of the API is the `where` shape:

```ts
where: (conditions: Conditions<T>) => (Condition | undefined)[]
```

Three things to notice:

- It's a **callback**, not an object literal. The builder hands you a `Conditions<T>` proxy whose keys are the entity's properties and whose values are field-level builders (`eq`, `ne`, `gt`, `in`, `isNull`, …). This is what lets `u.name.eq('Alice')` autocomplete and type-check `'Alice'` against `T['name']`.
- It returns an **array**. Each element is one `Condition`, ANDed together at SQL composition time. No nested `and`/`or` tree today — if you need OR, you'll need a future API extension. Honest constraint, not an oversight.
- Entries may be **`undefined`** *at the type level* (because the proxy is `Partial`), but `undefined` is rejected at runtime with `UndefinedWhereConditionError`, carrying the offending index. The proxy is `Partial` so TypeScript stays quiet when a property has no `@Column`; the runtime error is what catches the actual mistake. The error message points the user straight at the missing decorator.

The `Conditions<T>` proxy is built by reflecting over the entity's column metadata at builder time — see [[Decorator Metadata Storage]] for where that metadata lives.

---

## Terminal methods

Two today: `getMany()` and `getOne()`.

- `getMany(): Promise<T[]>` — returns every matching row. Empty result is `[]`, never `null`. Cardinality is "zero-or-more".
- `getOne(): Promise<T | null>` — returns the first row of `getMany()` or `null`. It does **not** add `LIMIT 1` to the SQL today; it slices the result client-side. That's fine for the small result sets the current API targets, and it's a one-line change when it stops being fine.

Notably absent: `getCount()`, `getExists()`, streaming. All future work. The shape `getX(): Promise<X>` is the pattern; new terminals slot in without disturbing the clause API.

---

## SQL composition safety

The hard rule from [[Welcome]]: parameterized SQL only, no `sql.unsafe`. The builder's job is to honour that rule even when assembling clauses dynamically.

Two helpers do the heavy lifting:

- `sql(identifier)` — Bun's tagged-template form for binding identifiers (table and column names) safely.
- `sqlJoin({ sql, items, map, separator })` — composes an array of fragments into one fragment with a separator (default `, `, often overridden to `AND` for where-clauses).

Inside `getMany()`, operator tokens are looked up in a static `opFragments` map of pre-built `sql\`=\``-style fragments. That keeps user input from ever becoming a SQL token: the operator comes from a closed enum (`Condition['op']`), the column name goes through `sql(c.columnName)`, and the value goes through normal parameter binding.

`IN` is the one branch that needs care, because the value is an array. It composes as ``sql`${col} IN ${sql(c.value)}` `` — Bun's `sql(array)` form binds each element as a parameter, so an empty array produces a valid `IN ()` that matches nothing rather than crashing. The test suite pins that behaviour explicitly.

Note that hand-rolled `reduce` over an array of fragments is a footgun (easy to skip a separator, easy to drop the head element); `sqlJoin` exists precisely so I'm never tempted to write that loop inline. Every place in the codebase that needs to join SQL fragments uses it — `Repository.create`, `Repository.delete`, `Repository.update`, and the builder itself.

---

## Type flow

`QueryBuilder<T>` carries the entity type from `Repository<T>` end-to-end. The chain is:

- `Repository<T>` constructor takes `new () => T`.
- `findMany(options?: FindOptions<T>)` → `new QueryBuilder<T>(...)`.
- `applyOptions(options?: FindOptions<T>): this` → preserves `T` for chaining.
- `getMany(): Promise<T[]>` / `getOne(): Promise<T | null>` → results land typed as `T`.

The interesting types are `Conditions<T>` and `FieldConditionBuilder<V>`:

```ts
export type Conditions<T> = {
  [K in keyof T]?: FieldConditionBuilder<T[K]>;
};
```

A mapped type over the entity's keys, each mapped to a per-field builder parameterized by the *field's* type. So `u.age.gt(18)` accepts `number`, `u.name.eq('Alice')` accepts `string`, and a typo on the property name is a compile error.

The `?` on the mapped type is what allows the `u.name?.eq(...)` calls in tests and examples to type-check against entities where some properties live on a base class or are optional. The runtime layer (the proxy + the `UndefinedWhereConditionError`) covers the gap between "TypeScript thinks this might be undefined" and "actually, the column doesn't exist".

---

## Where this ends and the repository begins

The seam is intentional, and worth stating explicitly:

- **Repository owns by-key operations and writes.** `findById(key)`, `create(entity)`, `delete(key)`, `update(entity)` — these are operations where the *shape of the call* is fixed by the entity's primary key. They build trivial queries internally and don't expose a builder, because there's nothing to compose. See [[Repository Contract]].
- **Builder owns composition.** Anything where the caller assembles a query out of clauses — `where`, inheritance scope, future `orderBy`/`limit`/joins — flows through `QueryBuilder<T>`.

`findOne(options?)` and `findMany(options?)` are repository methods that internally construct a builder and call its terminal in one shot. They're a convenience layer for the common case where you don't need the builder reference, not a parallel API.

---

## Trade-offs and open questions

Things I'm consciously deferring:

- **No OR / no nested boolean trees.** Where is a flat AND list. A real boolean DSL (`u.and(u.or(...))`) is significant API surface and I want at least one concrete use case before designing it.
- **No `orderBy`, `limit`, `offset` yet.** Trivial to add as new fields on the builder and new clauses in `getMany()`. Held back because none of the examples need them today and I don't want to ship API I haven't dogfooded.
- **No joins surface.** Relations are loaded eagerly through repository helpers, not through builder-level joins. A `leftJoin` API is the most likely future expansion, and it's also where the parameterized-only rule will get the most stress — joining on user-supplied column names is exactly the territory `sql(identifier)` was designed for, but the API ergonomics are non-trivial.
- **No raw escape hatch.** No `qb.raw(...)`. Adding one would weaken the parameterized-only invariant; I'd rather extend the typed surface than open a hole.
- **`getOne()` slices instead of `LIMIT 1`.** Fine for now, suboptimal for large filtered sets. Listed in [[Open Questions]].
- **`applyOptions()` replaces conditions instead of accumulating them.** This is consistent today but surprises people who expect builder semantics to be additive across calls. Worth revisiting if a real composition use case appears.

The throughline: keep the builder small, keep the SQL parameterized, keep the types honest. Everything else is future work, and the current shape is meant to absorb that work without rewrites.
