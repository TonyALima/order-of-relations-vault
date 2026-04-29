---
type: concept
title: "Dependency Injection Container"
complexity: intermediate
domain: "Application architecture"
aliases:
  - "DI container"
  - "IoC container"
  - "Container"
created: 2026-04-29
updated: 2026-04-29
tags:
  - concept
  - di
  - container
status: seed
related:
  - "[[0003-singleton-di-container]]"
  - "[[Repository Pattern]]"
  - "[[sources/welcome]]"
sources:
  - "../.raw/welcome.md"
---

# Dependency Injection Container

## Definition

A **DI container** is an object that owns service instances and resolves their dependencies on demand. In OOR, the `Container` is a singleton holding service singletons. `@Service` registers a constructor; `@Inject` and `@InjectRepository` mark fields to be populated when the service is resolved.

OOR's container is intentionally minimal: single scope (singleton), no async resolution, no lifecycle hooks. Its job is one thing — make `Repository<T>` injection ergonomic.

## How It Works

- `@Service` decorates a class. At class-evaluation time, the decorator registers the constructor in the container's registry under the class itself as the token.
- `@Inject(SomeService)` decorates a field. When the container resolves the owning class, it looks up `SomeService` in the registry, instantiates it once (or returns the cached instance), and assigns it to the field.
- `@InjectRepository(EntityClass)` is a specialization that resolves the `Repository<EntityClass>` for that entity, again from a singleton cache keyed by the entity constructor.

Because everything is singleton-scoped, instance graphs are identical for every consumer in the same process.

## Why It Matters

- **Repositories without ceremony.** A service annotates a field and gets a wired-up `Repository<T>`. No factory function, no manual `new`.
- **Tests can swap.** Tests register a fake registration before resolving the service under test, replacing the real `Repository<T>` with a stub.
- **Bounded scope.** Because the container does *only* repository wiring, it stays small enough to read and reason about. The risk it absorbs is "a 30-line container" rather than "an Inversify config."

## Examples

```ts
@Service
class UserService {
  @InjectRepository(User)
  private users!: Repository<User>;

  @Inject(EmailGateway)
  private email!: EmailGateway;

  async invite(email: string) {
    const user = await this.users.create({ email });
    await this.email.send(user, "welcome");
  }
}

// Somewhere at startup:
const userService = container.get(UserService);
```

## Connections

- [[0003-singleton-di-container]] — the ADR fixing scope and intent.
- [[Repository Pattern]] — the primary thing the container exists to wire.

## Sources

- `.raw/welcome.md` § "DI container as a singleton"
