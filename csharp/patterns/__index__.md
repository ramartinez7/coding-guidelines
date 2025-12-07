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

### [Input Sanitization (Trusted Types)](./input-sanitization.md)

Validating and sanitizing input at multiple points in the codebase—use trusted types to validate once at the boundary.

### [Secret Types (Preventing Accidental Exposure)](./secret-types.md)

Secrets logged or serialized accidentally—use dedicated types that prevent rendering in logs, exceptions, and JSON.

### [Authentication Context (Type-Safe Identity)](./authentication-context.md)

Passing user IDs as primitives and checking `HttpContext.User` in business logic—use an authenticated context type that proves identity at compile time.

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

### [Polymorphic Feature Flags (Strategy over If)](./polymorphic-feature-flags.md)

Boolean feature flags checked inside business logic—use type registration to swap implementations at startup.

## API Style Patterns

### [Versioned Endpoints (Type-Safe API Versioning)](./versioned-endpoints.md)

API versioning with string paths and runtime route matching—use types to represent versions at compile time.

### [Content Negotiation (Type-Driven Response Formatting)](./content-negotiation.md)

Checking `Accept` headers with strings and manually serializing responses—use types to represent content types and let the framework handle serialization.

### [Pagination Cursors (Opaque Token Types)](./pagination-cursors.md)

Using page numbers or offsets for pagination exposes implementation details and breaks when data changes—use opaque cursor types for stable, efficient pagination.

### [Rate Limiting (Type-Safe Token Bucket)](./rate-limiting-tokens.md)

Manual rate limit checks scattered throughout code—use typed rate limit tokens to enforce limits at compile time.

### [Correlation IDs (Type-Safe Distributed Tracing)](./correlation-ids.md)

Passing correlation IDs as strings or relying on ambient context—use typed correlation tokens to track requests across service boundaries.

### [Conditional Requests (Type-Safe ETags and Preconditions)](./conditional-requests.md)

Checking ETags and If-Match headers with strings—use typed entity tags to prevent lost updates and enable optimistic concurrency.

### [HATEOAS Links (Type-Safe Hypermedia)](./hateoas-links.md)

Building URLs with string concatenation and magic strings—use typed link relations to make APIs discoverable and self-documenting.

### [Batch Operations (Type-Safe Bulk Processing)](./batch-operations.md)

Processing multiple items with loops and mixed results in a single array—use typed batch operations with explicit success/failure tracking.

## Type-Driven Development Patterns

### [Phantom Types (Compile-Time State Tracking)](./phantom-types.md)

Using runtime checks or comments to track object state—use phantom type parameters to encode state in the type system.

### [Units of Measure (Dimensional Analysis in Types)](./units-of-measure.md)

Mixing incompatible units (meters + feet, seconds + milliseconds) causes calculation errors—encode units in the type system to prevent dimensional mistakes.

### [Branded Primitives (Nominal Typing Over Structural)](./branded-primitives.md)

Type aliases (`using`) provide no type safety—create branded types to distinguish structurally identical but semantically different values.

### [Type Witnesses (Compile-Time Proofs)](./type-witnesses.md)

Runtime checks scattered throughout code—use type witnesses to prove conditions at compile time and eliminate redundant validation.

### [Type-Safe Enumerations (Smart Enums)](./type-safe-enumerations.md)

Using primitive enums loses type safety, behavior, and flexibility—use sealed class hierarchies to create rich, type-safe enumerations.

### [Type-Level State Machines](./type-level-state-machines.md)

Using enums or booleans to track state allows invalid transitions—use sealed class hierarchies to make illegal state transitions uncompilable.

### [Indexed Types (Type-Level Indices)](./indexed-types.md)

Array indexing with integers risks out-of-bounds errors—use typed indices to make invalid indices unrepresentable at compile time.

### [Type-Safe String Interpolation (Preventing Injection Attacks)](./type-safe-string-interpolation.md)

Building SQL, HTML, or URLs from string concatenation risks injection attacks—use typed builders to make injection unrepresentable.

### [Variance Patterns (Covariance and Contravariance)](./variance-patterns.md)

Ignoring variance in generic types leads to runtime casts and lost type safety—use `in` and `out` keywords to express variance constraints at compile time.

## Domain-Driven Design Patterns

### [Aggregate Roots (Enforcing Invariants Through Boundaries)](./aggregate-roots.md)

Entities with direct public setters allow invariants to be broken—use aggregate roots to enforce consistency boundaries.

### [Repository Pattern (Domain-Centric Data Access)](./repository-pattern.md)

Domain logic coupled to database—use repositories to abstract persistence and keep domain pure.

### [Ubiquitous Language (Translating Business Terms to Types)](./ubiquitous-language.md)

Technical jargon in code disconnected from business vocabulary—use types that mirror the language domain experts use.

## Performance Patterns

### [Memory Safety (Span and Ref Structs)](./memory-safety-span.md)

String manipulation causes GC pressure—use `Span<T>` and `ref struct` for zero-allocation, memory-safe code.

### [Struct Layout (Memory and Cache Optimization)](./struct-layout.md)

Structs with poor field ordering waste memory and cause cache misses—use explicit layout and field ordering for optimal performance.

### [Allocation Budget (Zero-Allocation Patterns)](./allocation-budget.md)

High-throughput code allocates excessively, triggering frequent GC pauses—use pooling, stack allocation, and budget tracking to eliminate allocations in hot paths.

### [Lazy Initialization (Type-Safe Deferred Computation)](./lazy-initialization.md)

Expensive initialization done eagerly when it might not be needed—use lazy initialization with type safety to defer computation until actually required.

## Exercises

### [Subscriptions Mini-Domain](./exercise-subscriptions.md)

A hands-on exercise applying multiple type-safety patterns to refactor a subscription service.

