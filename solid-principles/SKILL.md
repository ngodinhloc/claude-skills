---
name: solid-principles
description: Use when designing, writing, reviewing, or refactoring object-oriented code (classes, interfaces, modules with behavior) — applies the SOLID principles (Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, Dependency Inversion) to catch design smells and guide structure. Trigger on requests like "review this design", "refactor this class", "is this good OO design", "add a new type/strategy to this", or when a class/module is growing multiple unrelated responsibilities.
---

# SOLID Principles

Apply these five principles when designing or reviewing object-oriented code. They are heuristics for reducing coupling and increasing cohesion, not rules to follow dogmatically — flag violations that would cause real pain (hard to test, hard to extend, hard to change safely), not violations for their own sake. Do not over-apply: introducing interfaces, abstract base classes, or dependency injection for code that will only ever have one implementation is over-engineering, not SOLID.

## S — Single Responsibility Principle

A class/module should have one reason to change — one axis of responsibility, owned by one actor/stakeholder.

- **Smell:** a class mixes unrelated concerns, e.g. business logic + persistence + formatting/serialization in one class. Changes for unrelated reasons (a UI tweak vs. a DB schema change) touch the same file.
- **Fix:** split by responsibility (e.g. separate the domain logic from persistence and from presentation).
- **Don't over-apply:** a class with several small, tightly related methods that all serve one cohesive purpose is fine — SRP is about reasons to change, not method count.

## O — Open/Closed Principle

Code should be open for extension but closed for modification: adding new behavior shouldn't require editing existing, tested code.

- **Smell:** a growing `if/else` or `switch` on a type/enum that must be updated every time a new case is added (e.g. `if (shape.type === 'circle') ... else if (shape.type === 'square') ...`).
- **Fix:** use polymorphism, strategy objects, or a registry so new cases are added by adding new code, not editing a central branch.
- **Don't over-apply:** if the set of cases is genuinely fixed and unlikely to grow, a simple conditional is clearer than premature abstraction.

## L — Liskov Substitution Principle

A subtype must be usable anywhere its base type is expected, without breaking correctness — same behavioral contract, not just matching method signatures.

- **Smell:** a subclass throws on a method the base type promises, weakens a postcondition, strengthens a precondition, or returns something callers of the base type don't expect (classic example: `Square extends Rectangle` breaks callers that independently set width/height).
- **Fix:** don't force an inheritance relationship where the "is-a" doesn't hold behaviorally; prefer composition, or model the shared contract more narrowly.

## I — Interface Segregation Principle

Clients shouldn't be forced to depend on methods they don't use.

- **Smell:** a fat interface where implementers must stub out irrelevant methods (throwing `NotImplementedError`, returning dummy values) because the interface bundles unrelated capabilities.
- **Fix:** split into smaller, role-specific interfaces; a class can implement multiple narrow interfaces instead of one broad one.

## D — Dependency Inversion Principle

High-level modules shouldn't depend on low-level modules; both should depend on abstractions. Depend on interfaces/contracts, not concrete implementations, especially across architectural boundaries.

- **Smell:** business/domain logic directly instantiates or calls a concrete infrastructure class (a specific database client, HTTP client, filesystem call), making it impossible to test in isolation or swap the implementation.
- **Fix:** define the dependency as an interface/abstract contract owned by the high-level module; inject the concrete implementation (constructor injection, factory, DI container).
- **Don't over-apply:** don't introduce an interface for a stable, unlikely-to-change dependency (e.g. a language's standard library) just for the sake of it.

## How to apply this during work

- **Reviewing code:** scan for the smells above; report concrete violations with the specific principle, file/line, and the concrete failure mode it causes (untestable, hard to extend, fragile subclassing) — not just "this violates SRP."
- **Writing new code:** default to simple, direct implementations; reach for a SOLID-motivated abstraction (interface, strategy, injected dependency) only when there's a concrete second case or a concrete testing/coupling problem it solves, not speculatively.
- **Refactoring:** prefer the smallest change that resolves the violation over a full redesign, unless the user asked for a broader refactor.
