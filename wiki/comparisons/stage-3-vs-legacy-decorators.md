---
type: comparison
title: "Stage-3 vs Legacy Decorators"
subjects:
  - "[[ECMAScript Stage-3 Decorators]]"
  - "Legacy TypeScript Decorators"
dimensions:
  - standardization status
  - signature
  - reflect-metadata
  - runtime support
  - ergonomics
  - ecosystem positioning
verdict: "Stage-3 is the dialect TypeScript treats as the default and the language is standardizing. OOR's commitment to it is a bet that the polyfill-free, Reflect-free, toolchain-aligned version of decorator-based ORMs is the one worth building. The contribution is being early on the right side of that transition, with a design that doesn't depend on the legacy affordances."
created: 2026-04-29
updated: 2026-04-30
tags:
  - comparison
  - decorators
  - typescript
  - tc39
status: developing
related:
  - "[[0001-stage-3-decorators]]"
  - "[[ECMAScript Stage-3 Decorators]]"
  - "[[MetadataStorage]]"
  - "[[entity-registration]]"
  - "[[oor-vs-typeorm]]"
sources: []
---

# Stage-3 vs Legacy Decorators

## Why this matters for OOR

Decorators are the foundational mechanism a decorator-based ORM is built on. The dialect choice isn't a stylistic preference — it determines the runtime requirements OOR imposes on its consumers, the metadata storage shape, the toolchain assumptions, and which TC39 trajectory the library is aligned with.

OOR's commitment to ECMAScript Stage-3 decorators ([[0001-stage-3-decorators]]) is the foundational decision the rest of the architecture rests on. This page is the long-form case for why that bet is the right one for a 2026-and-beyond ORM.

> [!key-insight] One syntax, two dialects, one trajectory
> Both decorator dialects use the same `@decorator` source syntax. The difference is in *signatures* and *runtime requirements*. The trajectory the language is on is Stage-3's; legacy decorators are TypeScript-specific and never standardized. Picking legacy in 2026 means picking the deprecation track for a foundational mechanism.

## Comparison

| Dimension | Legacy decorators | Stage-3 decorators (OOR's choice) |
|-----------|-------------------|-----------------------------------|
| **Standardization** | Never standardized. TypeScript-specific extension behind `experimentalDecorators`. Diverges from the TC39 proposal trajectory. | TC39 **Stage 3** (decorator metadata accepted March 2023). On the standardization pipeline; the dialect runtimes will eventually ship natively. |
| **TypeScript flag** | Requires `experimentalDecorators: true` (and usually `emitDecoratorMetadata: true`). | Native since TypeScript 5.0 (March 2023). Active when `experimentalDecorators` is **off**. `tsc --init` defaults to Stage-3 (the legacy flag is commented out). |
| **Class decorator signature** | `(target: Function) => void \| Function` — receives the bare class. | `<T extends new (...a: any) => any>(value: T, context: ClassDecoratorContext<T>) => T \| void` — receives the class and a structured `context`. |
| **Field decorator signature** | `(target: Object, propertyKey: string \| symbol) => void` | `(value: undefined, context: ClassFieldDecoratorContext) => ((initial: T) => T) \| void` |
| **Method decorator signature** | `(target: Object, key: string \| symbol, desc: PropertyDescriptor) => PropertyDescriptor \| void` | `(value: Function, context: ClassMethodDecoratorContext) => Function \| void` |
| **Context object** | None — receives raw `target` and `key`. Side-channel state lives on `Reflect.metadata` or external maps. | First-class `context`: `{ kind, name, static, private, access, addInitializer, metadata }`. In-language replacement for the `Reflect` registry. |
| **`reflect-metadata` requirement** | Universal in practice. Libraries (TypeORM, NestJS DI, Inversify) read `design:type` / `design:paramtypes` via `Reflect.getMetadata`. Requires `import "reflect-metadata"` at process entry. | None. `context.metadata` is in-language. No global registry, no polyfill, no entry-point import. |
| **Library metadata isolation** | Shares the global `Reflect` registry with whatever else the host app uses `Reflect.metadata` for. | Library-owned. OOR's [[MetadataStorage]] is a `Map<Constructor, EntityMetadata>` that can change shape without colliding with anything else. |
| **`accessor` keyword** | No equivalent. | First-class. `accessor x: string;` desugars to a private field plus auto-getter/setter, which decorators can wrap via `{ get, set, init }`. |
| **`addInitializer` hook** | No equivalent — decorators must reach into the constructor manually. | First-class. `context.addInitializer(fn)` schedules `fn` at instance construction. Clean separation of class-evaluation work and per-instance work. |

## What OOR gains by committing to Stage-3

### Standardization track alignment

Legacy decorators are a TypeScript-specific extension that shipped before the TC39 proposal stabilized. They will never be standardized as-is — the proposal that *did* stabilize (Stage 3) has incompatible signatures. Choosing legacy now means choosing a track that will increasingly look like a transitional artifact.

Stage-3 is on the standardization pipeline. Whether it advances to Stage 4 in 2026 or 2027 doesn't change the trajectory: this is the dialect runtimes will eventually ship natively, and the dialect TypeScript treats as the default.

### Polyfill-free runtime

Legacy decorators in practice require `import "reflect-metadata"` at the top of every consuming app's entry point. That's a real cost: one transitive dependency, one polyfill that patches a global, one more thing that has to load before any decorated module evaluates. In a process with multiple decorator-using libraries, all of them write to the same global `Reflect` registry.

OOR ships without `reflect-metadata`. Consumers don't need a polyfill, don't need to remember to import it at process entry, and don't risk registry collisions with other libraries.

### Library-owned metadata

The legacy dialect's `Reflect.metadata` is a global registry. Any library can write to it; libraries that share the same key collide. Stage-3's `context.metadata` is library-owned.

OOR's [[MetadataStorage]] is a `Map<Constructor, EntityMetadata>` per `Database` instance. The storage shape can evolve — new fields, new symbols, new join passes — without colliding with whatever else uses `Reflect.metadata` in a consumer's app. The library has full control over its metadata layout.

### Toolchain alignment

Bun, Deno, and modern Node toolchains favor the standardization track. `tsc --init` defaults to Stage-3. Picking legacy in 2026 means swimming against the current of the language's own configuration defaults.

OOR being Bun-only ([[0007-bun-toolchain]]) compounds this — Bun's TypeScript frontend transpiles Stage-3 decorators natively, and the project's `tsconfig` is uncluttered by transitional flags. The toolchain story is "vanilla modern TypeScript," not "TypeScript with the legacy flags turned on."

### New affordances

Stage-3 introduces two affordances that legacy decorators have no equivalent for:

- **`accessor` keyword.** Lets a field be wrapped with auto-generated getter/setter, intercepted by a decorator's `{ get, set, init }` triple. Useful for change tracking, computed properties, and lazy initialization.
- **`addInitializer` hook.** Schedules work at instance construction, cleanly separated from class-evaluation work. Useful for instance-level initialization that the class declaration shouldn't run.

OOR doesn't use either today, but they're tools available if the design evolves toward needing them. Legacy decorators offer no such growth path.

## What OOR accepts as the cost

The Stage-3 dialect doesn't carry over two affordances that the legacy dialect was famous for. Both are deliberately accepted in OOR's design rather than worked around.

### Explicit column types instead of `design:type` inference

With `emitDecoratorMetadata: true`, the legacy dialect lets a library read the TypeScript type of a decorated property at runtime via `Reflect.getMetadata("design:type", target, key)`. TypeORM uses this to write `@Column()` without a type argument:

```ts
@Column()              // type inferred from `string` via design:type
firstName: string;
```

Stage-3 does **not** emit `design:type`. OOR pays this cost: column types are explicit.

```ts
@Column({ type: COLUMN_TYPE.TEXT })
firstName!: string;
```

The cost is bounded. The [[modules/sql-types|`COLUMN_TYPE`]] enum is closed (~50 PG types) and serves as documentation at the call site. It also surfaces a real TypeScript-vs-PostgreSQL impedance mismatch the inference papers over: `string` could be `TEXT`, `VARCHAR(n)`, `CHAR(n)`, `JSONB`-stringified, or `BYTEA`-base64'd. Forcing the user to pick is more honest than guessing.

### No constructor-parameter DI auto-wiring

The legacy dialect has parameter decorators (`@Inject(Foo) private foo: Foo`) and `design:paramtypes` runtime type info, which together let NestJS-style DI containers auto-wire constructors. Stage-3 has neither (parameter decorators are a separate sub-proposal that has not advanced).

OOR's planned [[Dependency Injection Container]] ([[0003-singleton-di-container]]) is *minimal* — singleton registration via explicit calls, not constructor-parameter auto-wiring. The DI design choice is downstream of the dialect choice: OOR doesn't need parameter decorators, so the gap doesn't apply. For an ORM, this is the right scope.

## Why OOR's bet pays off

The major decorator-using libraries in the JS ecosystem (TypeORM, NestJS, Inversify) are stuck on legacy decorators because they made architectural commitments that depend on `design:paramtypes` for constructor-parameter DI auto-wiring. Migrating those libraries to Stage-3 isn't a cosmetic change — it's a redesign of their core mechanism. The migration cost is substantial enough that none of them has announced a public plan.

OOR doesn't have that lock-in. The library was designed from day one to not need parameter decorators, and the metadata storage is a custom `Map`, not a `Reflect` registry. The Stage-3 path that's expensive for a 7-year codebase is the cheap default for a clean-room library — and it's the path the language is on.

For a TCC, this is the strongest version of "right place, right time": the dialect that the ecosystem will eventually have to migrate to is the dialect a new library can adopt at zero cost. OOR's commitment is a bet that the polyfill-free, Reflect-free, toolchain-aligned version of decorator-based ORMs is the one worth building — and the bet is structurally favorable.

## Sources

- TC39 proposal-decorators: <https://github.com/tc39/proposal-decorators>
- TypeScript 5.0 announcement: <https://devblogs.microsoft.com/typescript/announcing-typescript-5-0/#decorators>
- `experimentalDecorators` in tsconfig docs: <https://www.typescriptlang.org/tsconfig#experimentalDecorators>
- `reflect-metadata`: <https://github.com/rbuckton/reflect-metadata>
- Babel plugin-proposal-decorators: <https://github.com/babel/babel/tree/main/packages/babel-plugin-proposal-decorators>
- Bun TypeScript runtime: <https://bun.sh/docs/runtime/typescript>
- OOR: [[0001-stage-3-decorators]], [[ECMAScript Stage-3 Decorators]], [[MetadataStorage]]
