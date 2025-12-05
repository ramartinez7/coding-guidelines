# C# Patterns

A catalog of common code smells and refactoring patterns.

## Project Configuration

### [Nullable Reference Types](./nullable-reference-types.md)

Enable nullable reference types with warnings as errors—foundation for compile-time null safety.

### [Assemblies vs. Namespaces](./assembly-isolation.md)

Namespaces organize code but don't enforce boundaries—use separate assemblies for true isolation with `internal` visibility.

### [Validated Configuration (The Crash-Early Config)](./validated-configuration.md)

Configuration as "dictionary of strings" crashes hours later when accessed—use strongly-typed options with startup validation.

## Code Smells

### [Primitive Obsession](./primitive-obsession.md)

Over-use of "primitive" built-in types instead of domain-specific types.

### [Temporal Safety (The Timezone Trap)](./temporal-safety.md)

Using `DateTime` with its hidden `Kind` property—use `DateTimeOffset` or dedicated types instead.

### [Strongly Typed IDs](./strongly-typed-ids.md)

Multiple ID parameters of the same type (Guid, int)—wrap each in a distinct type to prevent swapping.

### [Data Clumps](./data-clump.md)

Values that always travel together appear as separate parameters.

### [Primitive Collections](./primitive-collections.md)

A special case of primitive obsession and data clumps applied to collections.

### [Flag Arguments](./flag-arguments.md)

Boolean or enum parameters that control which code path a method takes—use polymorphism instead.

### [Boolean Blindness](./boolean-blindness.md)

Methods returning `bool` where `true`/`false` meanings are unclear—use descriptive types instead.

### [Enum to Class Hierarchy](./enum-to-class-hierarchy.md)

Enums that determine which parameters are meaningful—model variants as distinct types instead.

### [Enforcing Call Order](./enforcing-call-order.md)

Runtime guards for method call sequences—use types to make invalid orderings uncompilable.

### [Enum State Machine](./enum-state-machine.md)

Enum-based state machines with fields valid only in certain states—model each state as a distinct type.

### [Honest Functions](./honest-functions.md)

Functions that lie about their return type (null, exceptions)—use Result types to make failure explicit.

### [Nullability vs. Optionality](./nullability-optionality.md)

Using `null` to mean "no value" when a dedicated `Option<T>` type would express intent more clearly.

### [Value Semantics](./value-semantics.md)

Mutable reference types for value concepts—use records or structs with structural equality instead.

### [Ghost States](./ghost-states.md)

Objects with partially-loaded or context-dependent properties—use context-specific types instead.

### [Structural Constraints (Refinement Types)](./refinement-types.md)

Standard types with dangerous structure (empty collections)—wrap to enforce constraints like "non-empty" at compile time.

### [Capability Security (Token Types)](./capability-security.md)

Checking authorization separately from actions—use capability tokens to make authorization proof part of the type system.

### [Boundary Enforcement (DTO vs. Domain)](./dto-domain-boundary.md)

Using the same class for API serialization and domain logic—separate DTOs from domain models with explicit mappers.

### [Snapshot Immutability (The "ReadOnly" Lie)](./snapshot-immutability.md)

`IReadOnlyList<T>` is not immutable—it's a view that promises *you* won't change it, but the underlying collection can still mutate. Use `System.Collections.Immutable` for true immutability.

## Creational Patterns

### [Static Factory Methods](./static-factory-methods.md)

Use static methods instead of constructors to create instances with clear, descriptive names.

### [Type-Safe Step Builder](./type-safe-builder.md)

Builder patterns that allow `.Build()` before required fields are set—use interface segregation to enforce the sequence.

## Behavioral Patterns

### [Specification Pattern (Logic as Types)](./specification-pattern.md)

Business rules buried in scattered LINQ lambdas—encapsulate rules as named, composable Specification types.

### [Idempotency Keys (Distributed Consistency)](./idempotency-keys.md)

Network retries cause duplicate operations—require an `IdempotencyKey` type to make operations safely repeatable.

### [Domain Events (Decoupling Side Effects)](./domain-events.md)

"God methods" with core action plus 10 side effects—return domain events as types to decouple and announce changes.

## Exercises

### [Subscriptions Mini-Domain](./exercise-subscriptions.md)

A hands-on exercise applying multiple type-safety patterns to refactor a subscription service.

