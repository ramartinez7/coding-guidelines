# C# Patterns

A catalog of common code smells and refactoring patterns.

## Clean Code Fundamentals

### [Meaningful Names](./meaningful-names.md)

Names should reveal intent—no comments needed to explain what a variable, method, or class does.

### [Small Functions](./small-functions.md)

Functions should do one thing, do it well, and be small enough to fit in your head.

### [Comments and Documentation](./comments-documentation.md)

Good code explains itself—comments should explain *why*, not *what*.

### [Error Handling](./error-handling.md)

Use exceptions for exceptional circumstances; use Result types for expected failures.

### [SOLID Principles](./solid-principles.md)

Five principles for creating maintainable, flexible object-oriented designs in C#.

### [Dependency Injection](./dependency-injection.md)

Inject dependencies through constructors rather than creating them internally or using service locators.

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

## Security Patterns

### [Common Security Attacks and Prevention Strategies](./security-attacks-overview.md)

Comprehensive overview of OWASP Top 10 security risks and how to prevent them using type-safe patterns—covers injection attacks, XSS, CSRF, deserialization, XXE, path traversal, open redirects, cryptographic failures, and SSRF with defense-in-depth strategies.

## API Security Patterns

### [CSRF Protection (Anti-Forgery Tokens)](./csrf-protection.md)

Using string-based tokens or forgetting CSRF checks—use typed anti-forgery tokens to prevent cross-site request forgery at compile time.

### [API Key Rotation (Time-Limited Credentials)](./api-key-rotation.md)

Using permanent API keys as strings—use time-limited, versioned API keys with automatic rotation to minimize exposure windows.

### [Request Signing (Message Authentication)](./request-signing.md)

Trusting API requests at face value—use HMAC-based request signatures to verify authenticity and prevent tampering.

### [Security Headers (Type-Safe HTTP Security)](./security-headers.md)

Setting security headers with string literals—use typed security headers to enforce security policies at compile time.

### [CORS Configuration (Type-Safe Origin Validation)](./cors-configuration.md)

Using string-based CORS policies—use typed origin validation to prevent unauthorized cross-origin access at compile time.

## Application Security Patterns

### [SQL Injection Prevention (Type-Safe Queries)](./sql-injection-prevention.md)

Building SQL with string concatenation or interpolation—use parameterized queries and typed SQL builders to make injection attacks unrepresentable.

### [Path Traversal Prevention (Safe File Paths)](./path-traversal-prevention.md)

Using unsanitized user input in file paths—use validated path types to prevent directory traversal attacks.

### [Command Injection Prevention (Safe Process Execution)](./command-injection-prevention.md)

Executing shell commands with untrusted input—use validated command types to prevent command injection attacks.

### [XXE Prevention (Safe XML Parsing)](./xxe-prevention.md)

Parsing XML with default settings enables XML External Entity (XXE) attacks—use secure XML parsing configuration to prevent entity expansion and external resource access.

### [Deserialization Prevention (Safe Object Deserialization)](./deserialization-prevention.md)

Deserializing untrusted data with `BinaryFormatter` or unrestricted type resolution—use safe serializers and type allowlists to prevent remote code execution.

### [Open Redirect Prevention (Safe URL Redirects)](./open-redirect-prevention.md)

Redirecting to URLs from untrusted input—use validated URL types to prevent open redirect attacks that enable phishing.

### [Cryptographic Best Practices (Type-Safe Cryptography)](./cryptographic-practices.md)

Using weak algorithms, hardcoded keys, or manual crypto implementation—use modern cryptographic types and key management to prevent security vulnerabilities.

## Type-Driven Development Patterns

### [When to Create Domain Types (Domain Primitive Decision Guide)](./when-to-create-domain-types.md)

Not every string, int, or decimal needs its own type—use this guide to decide when to create domain-specific types versus using primitives.

### [Smart Constructors (Parse, Don't Validate)](./smart-constructors.md)

Validation that returns boolean forces callers to check and remember—use smart constructors that transform untrusted input into trusted types.

### [Domain Invariants (Enforcing Business Rules at Construction)](./domain-invariants.md)

Business rules validated at multiple points allow invalid objects to exist—enforce invariants in constructors to make invalid states unrepresentable.

### [Typed Errors (Making Failure Cases Explicit)](./typed-errors.md)

String error messages lose type information and force string parsing—use discriminated unions to represent specific error cases with type safety.

### [Type-Safe Workflow Modeling (Business Processes as Types)](./type-safe-workflow-modeling.md)

Business workflows with runtime state checks allow invalid transitions—model each workflow step as a distinct type to make illegal transitions unrepresentable.

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

## Clean Architecture Patterns

### [Clean Architecture (Onion Architecture / Hexagonal Architecture)](./clean-architecture.md)

Large codebases with tangled dependencies make testing and change difficult—organize code in layers where dependencies point inward toward the domain.

### [Use Cases / Application Services (Orchestrating Domain Logic)](./use-cases.md)

Business workflows scattered across controllers and services make the system's capabilities unclear—encapsulate each use case as a distinct class that orchestrates domain logic.

### [Screaming Architecture (Project Structure Reveals Intent)](./screaming-architecture.md)

Generic folder structures like "Controllers," "Services," "Models" hide what the system does—organize by features so the architecture screams the business domain.

### [Dependency Rule Enforcement (Architectural Constraints)](./dependency-rule-enforcement.md)

Dependencies between layers can accidentally point the wrong direction—use compile-time checks to enforce that dependencies flow inward toward the domain.

## Domain-Driven Design Patterns

### [Entities vs Value Objects (Identity vs Equality)](./entities-vs-value-objects.md)

Objects with mutable state and identity tracked over time—distinguish entities from value objects to model domain accurately.

### [Aggregate Roots (Enforcing Invariants Through Boundaries)](./aggregate-roots.md)

Entities with direct public setters allow invariants to be broken—use aggregate roots to enforce consistency boundaries.

### [Repository Pattern (Domain-Centric Data Access)](./repository-pattern.md)

Domain logic coupled to database—use repositories to abstract persistence and keep domain pure.

### [Domain Services (Stateless Domain Operations)](./domain-services.md)

Business logic that doesn't naturally belong to any single entity or value object—use domain services to coordinate operations across multiple aggregates.

### [Domain Factories (Complex Object Creation)](./domain-factories.md)

Complex object creation with invariants and multiple dependencies—use factories to encapsulate construction logic and ensure valid aggregates.

### [Bounded Contexts (Strategic Domain Boundaries)](./bounded-contexts.md)

Large models with conflicting meanings for the same term—divide the system into bounded contexts where each term has one precise definition.

### [Anti-Corruption Layer (Protecting Domain Boundaries)](./anti-corruption-layer.md)

External systems with incompatible models pollute your domain—use an anti-corruption layer to translate and isolate foreign concepts.

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

