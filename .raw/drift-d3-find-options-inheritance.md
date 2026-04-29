# Drift Correction — `FindOptions.inheritance` is a real, undocumented option

This is the largest gap I noticed in the wiki: `FindOptions<T>` carries a second option besides `where` — `inheritance: InheritanceSearchType` — which controls how reads scope across a single-table-inheritance hierarchy. Every wiki reference to `applyOptions(options?)` and `FindOptions<T>` mentions only `where`. The `inheritance` option is shipped, exported, used by the inheritance example, and unmentioned anywhere in `wiki/`.

This needs to be added to the wiki — both in the QueryBuilder reference and in the Single-Table Inheritance concept page (which currently describes only the *metadata* side of STI, not the user-facing query side).

## What the code actually exposes

`src/query-builder/types.ts`:

```ts
export enum InheritanceSearchType {
  ALL = 'ALL',
  ONLY = 'ONLY',
  SUBCLASSES = 'SUBCLASSES',
}

export interface FindOptions<T> {
  where?: (conditions: Conditions<T>) => (Condition | undefined)[];
  inheritance?: InheritanceSearchType;
}
```

`src/query-builder/query-builder.ts` consumes it inside `applyOptions`:

```ts
if (options?.inheritance === InheritanceSearchType.SUBCLASSES) {
  this.setSubClassesDiscriminator();
} else if (options?.inheritance === InheritanceSearchType.ONLY) {
  this.setConcreteClassDiscriminator();
}
```

The two helper methods (`setSubClassesDiscriminator`, `setConcreteClassDiscriminator`) push entries onto `this.conditions` — the same array `where` writes to.

## What each `InheritanceSearchType` value does

`InheritanceSearchType` has three variants. Their semantics, derived from the code:

### `ALL` (the default — no entry, or explicit `ALL`)

No discriminator filter is added. The query reads **every** row in the inherited table, regardless of which subclass each row maps to. The result is typed as `T[]` but may include rows whose discriminator names a sibling subclass; the caller is on the hook for that.

This is the value that **does nothing** — `applyOptions` has no branch for `ALL`. Treat it as the "I know what I'm doing, give me everything in the table" escape hatch.

### `ONLY`

Adds an `=` predicate against the `discriminator` column equal to *this entity's* own discriminator. Reads return rows that map exactly to `T`, never to a subclass.

```ts
// pseudocode of what gets pushed onto conditions[]
{ columnName: 'discriminator', op: '=', value: meta.discriminator }
```

This is the value to use when you want `userRepo.findMany({ inheritance: ONLY })` to return *just* `User` rows and skip every `AdminUser`.

### `SUBCLASSES`

Adds an `IN` predicate against the `discriminator` column whose value list is **every discriminator that is `T` or a descendant of `T`** in the prototype chain. Implementation:

```ts
for (const [t, m] of this.db.getMetadata()) {
  if (t === this.entity ||
      Object.prototype.isPrototypeOf.call(this.entity.prototype, t.prototype)) {
    subclasses.push(m);
  }
}
// pushes: { columnName: 'discriminator', op: 'IN', value: subclasses.map(s => s.discriminator!) }
```

Subclass discovery is via `Object.prototype.isPrototypeOf.call(this.entity.prototype, t.prototype)` — pure prototype-chain walk against the live metadata map. No name strings, no manual registry.

This is what `userRepo.findMany({ inheritance: SUBCLASSES })` does: it returns `User` rows **plus** every subclass's rows, all typed as `User`. The inheritance example (`examples/inheritance/services/UserHierarchyService.ts`) uses it for `listSubClassUsers()` and `listSubClassAdmins()`.

## Discriminator-only-when-needed interaction

The discriminator filter is only added when `meta.discriminator` is truthy. `MetadataStorage.resolveInheritance` (see `[[Single-Table Inheritance]]`) wipes the discriminator on entities that are alone in their table. So calling `userRepo.findMany({ inheritance: ONLY })` against a `User` that has no subclasses is a no-op at SQL level — the predicate is never appended. As soon as a sibling registers and resolution re-runs, the same call starts emitting the predicate. This is the correct, idempotent behaviour, but it's worth flagging in the wiki: *the option's effect depends on whether siblings exist, not just on what the caller passes*.

## Interaction with `where`

`applyOptions` runs the `where` callback first (which **replaces** `this.conditions` wholesale — see `[[apply-options-accumulation]]`), then the inheritance branch **pushes** onto the same array. Net effect: when a caller passes both `where` and `inheritance`, the inheritance discriminator condition gets ANDed onto the user's conditions, which is what you want. But if a caller calls `applyOptions` twice — once with `inheritance`, once with `where` — the second call's `where` clobbers the discriminator. That's a real footgun and worth at least a sentence in the wiki.

## Why this matters

- The inheritance example is the *single most prominent* use of the API surface that the wiki doesn't cover. An agent reading the wiki and never the examples would be unable to write a subclass-scoped query.
- The `[[Single-Table Inheritance]]` concept page documents how the discriminator is *computed and stored* at the metadata layer, but stops there. The actual user-visible question — "how do I read across a hierarchy?" — has no answer in the wiki. Adding it closes the loop.
- The closed `InheritanceSearchType` enum is exactly the kind of "type-driven public API" that `[[0005-no-any-type-driven-api]]` is supposed to enforce; documenting it is a small expression of that ADR.

## Wiki pages affected

| Page | Action |
|---|---|
| `wiki/components/QueryBuilder.md` | The `applyOptions()` section currently describes only `where`. Add an `inheritance` subsection (or table row) listing the three values, what each emits as SQL, and the discriminator-only-when-needed caveat. Also note that two private-but-actually-public methods (`setSubClassesDiscriminator`, `setConcreteClassDiscriminator`) push onto `this.conditions`. |
| `wiki/concepts/Single-Table Inheritance.md` | After the existing "How It Works" section, add a "Reading across the hierarchy" section that documents the three `InheritanceSearchType` values and shows the example call from `UserHierarchyService.listSubClassUsers()`. |
| `wiki/concepts/Lazy Query Builder.md` | The phrase *"the inheritance discriminator filter both reduce to entries in this array"* hints at the option but never names it. Add a forward-link to the new QueryBuilder / STI sections so the reader can find the user-facing API. |
| `wiki/brief.md` | The brief currently describes the `where` callback shape but not the `inheritance` option. Add one bullet under "Method-shape facts worth knowing": *"`FindOptions<T>` also accepts `inheritance: InheritanceSearchType` (`ALL` / `ONLY` / `SUBCLASSES`) — opts a read into / out of subclass-scoped discriminator filtering. See `[[Single-Table Inheritance]]`."* |
| `wiki/flows/query-lifecycle.md` | Step 4 (validation) and step 5 (composition) describe `where` only. After step 4, add a step (or sub-step) noting that if `options.inheritance` is `ONLY` or `SUBCLASSES`, an additional condition is *pushed* (not replaced) onto the array before composition. |
| `wiki/index.md` (if it has a "User-facing API" surface listing) | Add `InheritanceSearchType` to the list of exported types alongside `Conditions`, `Condition`, `FindOptions`. |

## Suggested example

The cleanest minimal example, lifted from `examples/inheritance/services/UserHierarchyService.ts`:

```ts
class User { /* ... */ }
@Entity(db) class AdminUser extends User { /* ... */ }

const userRepo  = new Repository(User, db);
const adminRepo = new Repository(AdminUser, db);

// Default: every row in the User table — User AND AdminUser rows
await userRepo.findMany();

// Just User rows (discriminator = 'User')
await userRepo.findMany({ inheritance: InheritanceSearchType.ONLY });

// User rows AND every subclass's rows (discriminator IN ['User', 'AdminUser'])
await userRepo.findMany({ inheritance: InheritanceSearchType.SUBCLASSES });
```

## Source citations

- `src/query-builder/types.ts` lines 23–32 — enum and `FindOptions` shape.
- `src/query-builder/query-builder.ts` lines 15–62 — the `setSubClassesDiscriminator` / `setConcreteClassDiscriminator` implementations and the `applyOptions` branching.
- `src/core/metadata/metadata.ts` lines 58–82 — `resolveInheritance` and the "wipe discriminator if singleton" rule.
- `examples/inheritance/services/UserHierarchyService.ts` — the canonical user-facing call site.
- `examples/inheritance/entities/AdminUser.ts` — the `extends User` shape that drives subclass discovery.
