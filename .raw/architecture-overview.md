# Architecture Overview

This note is the conceptual map. It explains how the pieces of OOR fit together and which direction the dependencies run. For the why behind the big calls (Stage-3 decorators, lazy query builder, no `any`), see [[Welcome]]. For deep dives into individual subsystems, see [[Decorator Metadata Storage]], [[Query Builder Design]], and [[Repository Contract]].

---

## The Layered View

OOR has five layers. Dependencies always point downward — a layer never imports from a layer above it.

```
@Entity / @Column / @PrimaryColumn / @ToOne / @Nullable    ← decorators
                       │  write to
                       ▼
              MetadataStorage (Map<Constructor, EntityMetadata>)
                       │  read by
                       ▼
                 Repository<T>
                       │  delegates composed reads to
                       ▼
                 QueryBuilder<T>
                       │  executes via
                       ▼
              Database (Bun SQL connection)
```

A few load-bearing properties of this stack:

- **Decorators are the only writers of metadata.** Nothing outside `src/decorators/` calls `MetadataStorage.set`. Once the entity classes have been evaluated, the metadata is frozen for the lifetime of the process.
- **`MetadataStorage` is owned by a `Database` instance**, not a global. `@Entity(db)` takes the database as an argument and registers the entity against that database's storage. Two databases in the same process means two metadata maps.
- **The repository never builds SQL on its own for read paths.** `findOne`, `findMany`, and `findById` all construct a `QueryBuilder` and delegate. Writes (`create`, `update`, `delete`) build their SQL directly because they don't compose.
- **Bun's `SQL` is the only thing that touches the wire.** The query builder produces tagged-template fragments; nothing in OOR speaks the PostgreSQL protocol.

---

## Lifecycle of a Query

Walking through `userRepo.findMany({ where: (u) => [u.email!.eq('a@b.com')] })`:

1. **Repository entry.** `Repository.findMany` constructs a fresh `QueryBuilder<User>` bound to the entity constructor and the `Database`. No SQL yet.
2. **Metadata resolution.** The query builder calls `db.getMetadata().get(User)`. On first access, `MetadataStorage` runs `resolveInheritance` (computing each entity's `tableName` and `discriminator`) and `resolveRelations` (filling in foreign-key column names that the relation decorators left as `null`). Subsequent calls hit a resolved cache.
3. **Conditions proxy.** The query builder builds a typed proxy from the entity's column metadata — one `FieldConditionBuilder` per column. The user's `where` callback receives that proxy and returns an array of `Condition` objects.
4. **Validation.** The query builder checks for `undefined` entries in the returned array (a common bug where someone wrote `u.foo?.eq(...)` against a non-existent column) and throws `UndefinedWhereConditionError` if it finds any.
5. **SQL composition.** On `getMany()` (or implicitly via `findMany` calling it), the builder maps each `Condition` to a tagged-template fragment and joins them with `sqlJoin`. Operators are pre-tokenised into a fragment table; values stay as parameters. There is no string concatenation of user-supplied data anywhere on this path.
6. **Execution.** The composed `sql\`SELECT … FROM … WHERE …\`` is awaited against the Bun `SQL` connection. Rows come back already shaped as `T[]`.

The same shape applies to `findOne` (which is `getMany().then(rows => rows[0] ?? null)`) and `findById` (which assembles a primary-key `where` from the entity's primary columns).

---

## How Decorators Talk to the Rest of the System

The decorator layer is intentionally narrow. Each decorator does one thing: write a record into `context.metadata` (Stage-3's per-class metadata bag) under a private symbol key.

- `@Column` and `@PrimaryColumn` push into `context.metadata[COLUMNS_KEY]`.
- `@ToOne` pushes into `context.metadata[RELATIONS_KEY]`.
- `@Nullable` and `@NotNullable` populate `context.metadata[NULLABLE_KEY]`, a per-property nullability map. `@Column` and `@ToOne` read it back at registration time and refuse to register a property whose nullability hasn't been declared.
- `@Entity(db)` is the only decorator that takes the database as an argument. It runs last (after the field decorators), pulls the accumulated arrays out of `context.metadata`, asserts that at least one column is primary, and writes the final `EntityMetadata` into `db.getMetadata()`.

This ordering is enforced by the language: field decorators run before class decorators, so by the time `@Entity` fires, the `COLUMNS_KEY` / `RELATIONS_KEY` arrays are already populated. See [[Decorator Metadata Storage]] for the symbol-keyed storage scheme and the inheritance resolution pass.

---

## The Boundary Between Repository and Query Builder

The split is deliberate and worth being precise about:

- **`Repository<T>` owns the entity-shaped operations.** `create`, `update`, `delete`, `findById` — anything where the input or output is *one entity* and the SQL shape is fixed by the metadata. These methods build SQL directly, because there's nothing to compose.
- **`QueryBuilder<T>` owns the composed reads.** Anything where the user supplies clauses (`where`, `inheritance` filtering for class-table inheritance) goes through it. `findMany` and `findOne` on the repository are thin wrappers that instantiate a builder, apply options, and call a terminal method.

The rule of thumb: if the operation's SQL shape can be derived purely from `EntityMetadata`, it belongs on the repository. If it depends on per-call user input that varies in arity, it belongs on the query builder.

[[Repository Contract]] and [[Query Builder Design]] cover the typing rules each side enforces (notably, `create()` requires non-optional fields at compile time, and the `where` callback is typed against the entity's actual column types).

---

## Database, Schema, and Lifecycle

`Database` does three jobs:

1. **Connection.** Wraps a Bun `SQL` instance. `connect(url?)` either uses the supplied URL or falls back to Bun's default (which reads `DATABASE_URL` from the environment).
2. **Metadata host.** Owns the `MetadataStorage` for entities registered against this instance.
3. **Schema lifecycle.** `create()` walks the metadata, emits `CREATE TABLE` for each entity (in two passes: base tables first, then `ALTER TABLE … ADD FOREIGN KEY` for relations), and handles class-table-inheritance discriminators. `drop()` does a topological reverse-walk of the relation graph so foreign-key targets are dropped after their referrers.

Schema migrations beyond `create` / `drop` belong in [[Schema Migrations]] — currently a placeholder.

---

## Services and Dependency Injection

[[Welcome]] frames a DI container as one of the pillars. As of this writing, the implementation in `src/` does not yet ship a `Container`, `@Service`, `@Inject`, or `@InjectRepository`. The current wiring pattern in `examples/` is direct construction:

```ts
export const db = new Database();
// ...
const userRepository = new Repository(User, db);
```

The intended boundary, when DI lands, is that the container is **above** the persistence layer — it composes services and hands them repositories — and never participates in metadata resolution or SQL execution. Repositories must remain constructible without the container so tests and one-off scripts don't pay for it.

---

## Source Tree, by Concern

A short map of where each concept lives. Useful when looking for the right file to touch.

- `src/decorators/` — every decorator. Writes only to `context.metadata` and (for `@Entity`) the database's metadata storage.
- `src/core/metadata/` — `MetadataStorage`, the resolved `EntityMetadata` / `ColumnMetadata` / `RelationMetadata` types, inheritance + relation resolution.
- `src/core/database/` — connection, schema `create` / `drop`, FK-aware drop ordering.
- `src/core/repository/` — `Repository<T>`, the entity-shaped CRUD surface, primary-key validation.
- `src/core/sql-types/` — the `COLUMN_TYPE` enum and the SQL fragment each type maps to.
- `src/core/utils/` — `sqlJoin` (the only sanctioned way to join SQL fragments) and shared types.
- `src/query-builder/` — `QueryBuilder<T>`, the conditions proxy, `FindOptions`, inheritance search modes.
- `src/errors.ts` and per-module `*.errors.ts` — every error type extends a single `OrmError` base, exported from the package root.

When in doubt about where something belongs, follow the layering rule: writers of metadata go in `decorators/`, readers of metadata go in `core/` or `query-builder/`, and nothing under `core/metadata/` is allowed to import from above.
