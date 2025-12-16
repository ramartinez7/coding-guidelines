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

### [Tell, Don't Ask](./tell-dont-ask.md)

Tell objects what to do—don't ask for their state and make decisions for them.

### [Law of Demeter (Principle of Least Knowledge)](./law-of-demeter.md)

Objects should only talk to their immediate friends—don't navigate through chains of relationships.

### [Extension Methods vs Inheritance](./extension-methods-vs-inheritance.md)

Extend types without inheritance—extension methods add behavior without coupling, while inheritance creates tight relationships.

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

### [First-Class Collections](./first-class-collections.md)

Collections wrapped in dedicated types enforce invariants, provide domain-specific operations, and make intent explicit.

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

### [Method Chaining (Fluent Interfaces)](./method-chaining.md)

Design APIs where methods return `this` or the next builder step—enable readable, composable operation sequences.

## Behavioral Patterns

### [Null Object Pattern](./null-object-pattern.md)

Replace null references with objects that implement the expected interface but do nothing—eliminate null checks.

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

### [Principle of Least Privilege (Type-Safe Privilege Boundaries)](./principle-of-least-privilege.md)

Granting broad permissions and trusting code to self-limit—use types to enforce that code can only access the minimum privileges required through scoped repositories and narrow interfaces.

### [Secure Defaults (Fail-Safe Design)](./secure-defaults.md)

Types that allow insecure configurations through optional parameters—use secure defaults and make insecure choices explicit and difficult through scary naming and required justification.

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

### [Making Invalid States Unrepresentable](./making-invalid-states-unrepresentable.md)

The golden rule of type-driven design—if an invalid state cannot be represented in the type system, it cannot exist at runtime. Design types where invalidity is impossible.

### [When to Create Domain Types (Domain Primitive Decision Guide)](./when-to-create-domain-types.md)

Not every string, int, or decimal needs its own type—use this guide to decide when to create domain-specific types versus using primitives.

### [Smart Constructors (Parse, Don't Validate)](./smart-constructors.md)

Validation that returns boolean forces callers to check and remember—use smart constructors that transform untrusted input into trusted types.

### [Domain Invariants (Enforcing Business Rules at Construction)](./domain-invariants.md)

Business rules validated at multiple points allow invalid objects to exist—enforce invariants in constructors to make invalid states unrepresentable.

### [Result Monad (Railway-Oriented Programming)](./result-monad.md)

Using exceptions for control flow or returning null on failure—use Result types to make success and failure explicit in the type system with composable error handling.

### [Option Monad (Explicit Optionality)](./option-monad.md)

Using null to represent "no value" or optional properties—use Option types to make optionality explicit, composable, and null-safe with forced handling of absence.

### [Typed Errors (Making Failure Cases Explicit)](./typed-errors.md)

String error messages lose type information and force string parsing—use discriminated unions to represent specific error cases with type safety.

### [Discriminated Unions (Modeling Mutually Exclusive Alternatives)](./discriminated-unions.md)

Using multiple nullable fields or flags to represent alternatives—use discriminated unions (sum types) to model "one of" relationships where only one variant can exist at a time.

### [Exhaustive Pattern Matching (Compiler-Enforced Completeness)](./exhaustive-pattern-matching.md)

Using if-else chains or switch with default that hide missing cases—use sealed type hierarchies and pattern matching to make incomplete handling a compile error.

### [Newtype Pattern (Zero-Cost Wrappers)](./newtype-pattern.md)

Using primitives directly when you need nominal typing—use the newtype pattern to create compile-time distinct types with zero runtime overhead.

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

### [Record Equality (Value-Based Equality)](./record-equality.md)

Use records for types where equality should be based on value, not reference identity—compiler-generated value semantics.

### [Immutable Collections](./immutable-collections.md)

Use `System.Collections.Immutable` for collections that truly cannot change—not just read-only views.

### [Type-Safe State Transitions (Compiler-Enforced State Machines)](./type-safe-state-transitions.md)

Runtime checks for valid state transitions allow bugs to reach production—use types to make invalid transitions uncompilable.

### [Compile-Time Validation (Catching Errors Before Runtime)](./compile-time-validation.md)

Runtime validation catches errors in production—move validation to compile time using types to catch bugs during development.

### [Type-Level Constraints (Encoding Constraints in the Type System)](./type-level-constraints.md)

Runtime validation catches errors late—encode constraints in types to catch violations at compile time.

### [Type-Safe Functional Core (Functional Core, Imperative Shell)](./type-safe-functional-core.md)

Business logic mixed with I/O and side effects makes testing hard and reasoning difficult—separate pure functional core from imperative shell using types to enforce the boundary.

### [Type-Safe CQRS (Command Query Responsibility Segregation with Types)](./type-safe-cqrs.md)

Mixing reads and writes in the same model creates complexity—separate commands and queries using types to enforce the distinction at compile time.

### [Type-Safe Event Sourcing (Events as Source of Truth)](./type-safe-event-sourcing.md)

Storing only current state loses history—use event sourcing with typed events to maintain complete audit trail and enable time travel.

### [Type-Safe Messaging (Compile-Time Message Contract Safety)](./type-safe-messaging.md)

Message contracts as strings or dynamic objects fail at runtime—use typed messages with serialization contracts to catch integration errors early.

### [Type-Safe Configuration (Compile-Time Configuration Validation)](./type-safe-configuration.md)

Configuration loaded as strings and dictionaries fails at runtime—use strongly-typed configuration with validation to catch errors at startup.

### [Dependent Types Emulation (Types That Depend on Values)](./dependent-types-emulation.md)

Types that can't express relationships between values lead to runtime checks—emulate dependent types to encode value relationships in the type system.

### [Algebraic Effects Emulation (Effect Handlers in C#)](./algebraic-effects-emulation.md)

Side effects hidden in methods make code unpredictable—make effects explicit in types to enable effect handling and interpretation.

### [Type-Safe Query Building (LINQ Safety)](./type-safe-query-building.md)

Building LINQ queries with string column names risks runtime errors—use expression trees and typed query builders to make invalid queries uncompilable.

### [Type-Safe Dependency Injection (Compile-Time DI Validation)](./type-safe-dependency-injection.md)

Runtime service resolution failures from missing registrations or circular dependencies—use compile-time DI validation and strongly-typed service descriptors to catch configuration errors early.

### [Type-Safe Serialization Contracts (Preventing Serialization Errors)](./type-safe-serialization-contracts.md)

Serialization that fails at runtime from missing properties, type mismatches, or versioning issues—use typed serialization contracts to catch errors at compile time.

### [Type-Safe Resource Lifetime (RAII in C#)](./type-safe-resource-lifetime.md)

Resources cleaned up with `try-finally` or forgotten `Dispose()` calls leak—use the type system to enforce Resource Acquisition Is Initialization (RAII) and make resource leaks impossible.

### [Type-Safe Pipeline Builder (Data Transformation Pipelines)](./type-safe-pipeline-builder.md)

Chaining transformations with generic `Func<>` delegates loses type information and allows incompatible steps—use typed pipeline builders to make invalid pipelines uncompilable.

### [Type-Safe Retry Policies (Exponential Backoff Encoding)](./type-safe-retry-policies.md)

Retry logic with magic numbers and scattered configuration fails silently—encode retry policies in types to make retry behavior explicit and composable.

### [Type-Safe Feature Toggles (Compile-Time Feature Flags)](./type-safe-feature-toggles.md)

Boolean feature flags checked at runtime allow dead code and untested paths—use types to make features explicit and eliminate runtime checks.

### [Type-Safe Multitenancy (Tenant Isolation)](./type-safe-multitenancy.md)

Tenant IDs passed as primitives allow cross-tenant data leaks—use typed tenant contexts to enforce tenant isolation at compile time.

### [Type-Safe Localization (i18n Keys)](./type-safe-localization.md)

String-based localization keys fail silently when keys are misspelled or missing—use typed resource keys to catch localization errors at compile time.

### [Type-Safe Caching Strategies (Cache Key Safety)](./type-safe-caching-strategies.md)

String-based cache keys collide, expire silently, and can't be invalidated reliably—use typed cache keys to prevent cache pollution and ensure type-safe cache operations.

### [Type-Safe Health Checks (Dependency Health Encoding)](./type-safe-health-checks.md)

Health check endpoints with string-based status and no type safety—use typed health descriptors to make health check contracts explicit and composable.

### [Type-Safe Circuit Breaker States](./type-safe-circuit-breaker-states.md)

Circuit breaker state machines with enums and runtime checks allow invalid transitions—use sealed class hierarchies to make illegal state transitions uncompilable.

### [Type-Safe Permission Composition (Combining Capabilities)](./type-safe-permission-composition.md)

Permissions checked with boolean AND/OR operations allow privilege escalation—use typed permission composition to enforce least privilege through the type system.

### [Type-Safe Audit Trail (Event Type Safety)](./type-safe-audit-trail.md)

Audit logs with string messages and untyped data lose type information—use typed audit events to make audit trails queryable and type-safe.

### [Type-Safe Protocol Versioning (API Protocol Evolution)](./type-safe-protocol-versioning.md)

API protocol versions tracked with integers or strings allow version mismatches—use typed protocol versions to enforce compatibility at compile time.

### [Type-Safe Null Object with Phantom Types](./type-safe-null-object-phantom.md)

Null object pattern implemented with single class lacks type information—use phantom types to distinguish between null objects and real instances at compile time.

### [Type-Safe Data Migration (Schema Versioning)](./type-safe-data-migration.md)

Database migrations with version numbers and SQL scripts fail silently—use typed migrations to enforce schema evolution at compile time.

### [Type-Safe Cross-Cutting Concerns (Logging and Metrics)](./type-safe-cross-cutting-concerns.md)

Logging and metrics with string keys and untyped values lose type safety—use typed telemetry to make cross-cutting concerns compile-time safe.

### [Type-Safe Saga Orchestration (Workflow Compensation)](./type-safe-saga-orchestration.md)

Saga patterns with string-based step names and runtime compensation logic—use typed saga steps to enforce compensation at compile time.

### [Type-Safe Aspect-Oriented Composition](./type-safe-aspect-oriented-composition.md)

Cross-cutting concerns applied with reflection or dynamic proxies fail silently—use typed decorators and composition to make aspects compile-time safe.

### [Type-Safe Rate Limiting Buckets](./type-safe-rate-limiting-buckets.md)

Rate limiting with counters and timestamps scattered in code—use typed rate limit buckets to enforce rate limits through the type system.

### [Type-Safe Background Job Scheduling](./type-safe-background-job-scheduling.md)

Background jobs scheduled with string job names and untyped parameters fail silently—use typed job descriptors to make job scheduling compile-time safe.

### [Type-Safe Graph Traversal](./type-safe-graph-traversal.md)

Graph traversals with generic node types and unsafe casting—use typed graph structures to make traversal paths compile-time safe.

### [Type-Safe Time-Based Access Control](./type-safe-time-based-access-control.md)

Access control based on time windows checked at runtime—use typed temporal permissions to enforce time-based access at compile time.

### [Type-Safe Command Pattern with Validation](./type-safe-command-pattern-validation.md)

Commands executed without validation or with runtime type checking—use typed commands with compile-time validation to ensure command integrity.

## Clean Architecture Patterns

### [Clean Architecture (Onion Architecture / Hexagonal Architecture)](./clean-architecture.md)

Large codebases with tangled dependencies make testing and change difficult—organize code in layers where dependencies point inward toward the domain.

### [Use Cases / Application Services (Orchestrating Domain Logic)](./use-cases.md)

Business workflows scattered across controllers and services make the system's capabilities unclear—encapsulate each use case as a distinct class that orchestrates domain logic.

### [Screaming Architecture (Project Structure Reveals Intent)](./screaming-architecture.md)

Generic folder structures like "Controllers," "Services," "Models" hide what the system does—organize by features so the architecture screams the business domain.

### [Dependency Rule Enforcement (Architectural Constraints)](./dependency-rule-enforcement.md)

Dependencies between layers can accidentally point the wrong direction—use compile-time checks to enforce that dependencies flow inward toward the domain.

### [Port-Adapter Separation (Hexagonal Core)](./port-adapter-separation.md)

Core business logic defines abstract ports (interfaces), while adapters implement concrete connections to external systems—enabling the domain to remain independent of technical details.

### [Hexagonal Ports (Input/Output Boundaries)](./hexagonal-ports.md)

Define clear boundaries between the application core and external concerns through ports (interfaces)—primary ports for use cases driven by external actors, secondary ports for infrastructure services needed by the core.

### [Primary vs Secondary Adapters](./primary-secondary-adapters.md)

Primary adapters drive the application (HTTP controllers, CLI, message consumers), while secondary adapters are driven by the application (database repositories, email services, external APIs)—understanding the direction of dependency flow is key to hexagonal architecture.

### [Onion Layer Dependencies](./onion-layer-dependencies.md)

Organize code in concentric layers where dependencies point inward toward the domain—outer layers depend on inner layers, never the reverse, ensuring the domain remains pure and infrastructure swappable.

### [Application Service Layer Isolation](./application-service-isolation.md)

Application services orchestrate domain logic without containing business rules themselves—they coordinate use cases, manage transactions, and translate between external concerns and the domain.

### [Domain Service vs Application Service](./domain-service-vs-application-service.md)

Domain services encapsulate business logic that doesn't belong to a single entity, while application services orchestrate use cases and coordinate between domain and infrastructure—understanding the difference prevents logic from landing in the wrong layer.

### [Read Model vs Write Model (CQRS Application Layer)](./read-write-model-separation.md)

Separate read operations (queries) from write operations (commands) to optimize each for their specific purpose—commands validate and modify state, queries return pre-computed views without business logic.

### [Query Objects (Read-Side Abstraction)](./query-objects.md)

Encapsulate database queries in dedicated query objects to separate data retrieval concerns from controllers and keep read-side logic organized and testable.

### [Command Handlers (Write-Side Orchestration)](./command-handlers.md)

Encapsulate write operations in dedicated command handlers that validate input, execute business logic, and coordinate domain operations—separating concerns and enabling decorator-based cross-cutting concerns.

### [Domain Event Handlers](./domain-event-handlers.md)

Domain events decouple side effects from core business logic—publish events when something significant happens in the domain, then handle them with separate event handlers to maintain clean boundaries.

### [Cross-Cutting Concerns via Decorators](./cross-cutting-concerns-decorators.md)

Apply cross-cutting concerns (logging, validation, authorization, transactions) through the Decorator pattern rather than scattering them throughout code—keeping business logic clean and concerns composable.

### [Infrastructure Service Adapters](./infrastructure-service-adapters.md)

Implement infrastructure concerns (email, file storage, external APIs) as adapters that implement domain or application interfaces—keeping infrastructure swappable and domain pure.

### [Persistence Ignorance](./persistence-ignorance.md)

Domain entities should have no knowledge of how they're persisted—no ORM attributes, no database concerns, keeping the domain pure and persistence swappable.

### [Mapping Strategies (Domain-DTO Translation)](./mapping-strategies.md)

Translate between domain models and DTOs using explicit mapping strategies—keeping domain pure while providing API-friendly representations.

### [Application Layer Validation vs Domain Validation](./application-validation-vs-domain-validation.md)

Application layer validates input and authorization, while domain layer enforces business invariants—separating concerns and preventing invalid domain objects from existing.

### [Use Case Transactions and Unit of Work](./use-case-transactions-unit-of-work.md)

Manage database transactions at the application layer using the Unit of Work pattern—ensuring atomic operations and keeping the domain transaction-agnostic.

### [Layered Testing Strategy](./layered-testing-strategy.md)

Test each architectural layer with appropriate strategies—unit tests for domain logic, integration tests for infrastructure, and end-to-end tests for critical paths—maximizing confidence while minimizing test maintenance.

### [Infrastructure Layer Organization](./infrastructure-layer-organization.md)

Organize infrastructure code by technology (Persistence, Email, ExternalApis) or by feature—keeping adapters cohesive and making it easy to locate and swap implementations.

### [Composition Root (DI Container Configuration)](./composition-root.md)

Configure dependency injection in a single location (composition root) at the application's entry point—wiring together interfaces and implementations while keeping the application core ignorant of DI frameworks.

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

### [Value Object Composition (Building Complex Values from Simple Ones)](./value-object-composition.md)

Primitive values scattered throughout code—compose value objects from other value objects to model rich domain concepts with guaranteed consistency.

### [Aggregate Consistency Boundaries (Transactional Consistency Through Design)](./aggregate-consistency-boundaries.md)

Entities modified in multiple transactions lead to inconsistent state—define aggregate boundaries to ensure atomic, consistent updates across related objects.

### [Domain Model Isolation (Pure Domain Logic Without Infrastructure Concerns)](./domain-model-isolation.md)

Domain logic mixed with database, HTTP, or framework code makes testing hard and couples business rules to infrastructure—isolate pure domain models from all external concerns.

### [Rich Domain Models (Behavior-Rich Entities vs Anemic Models)](./rich-domain-models.md)

Entities with only getters/setters and no behavior push business logic into services—create rich domain models that encapsulate both data and the operations that manipulate it.

### [Entity Identity Strategies (Choosing the Right Identity Pattern)](./entity-identity-strategies.md)

Using auto-increment IDs or random GUIDs without considering domain needs leads to confusion and performance issues—choose identity strategies that match business requirements and technical constraints.

### [Lifecycle Management (Managing Entity State Transitions Over Time)](./lifecycle-management.md)

Entities that can be modified at any time without lifecycle constraints lead to invalid states—model entity lifecycles explicitly to control valid transitions.

### [Invariant Preservation (Maintaining Business Rules Throughout Entity Lifecycle)](./invariant-preservation.md)

Entities that can violate invariants after construction create bugs—preserve invariants at all times through encapsulation and immutability.

### [Domain Model Anemia Prevention (Avoiding Anemic Domain Models)](./domain-model-anemia-prevention.md)

Domain models with only getters/setters and no behavior result in business logic scattered across service classes—add behavior to entities to prevent anemic domain models.

### [Context Mapping (Strategic Domain Boundaries and Integration)](./context-mapping.md)

Shared models between domains create coupling and confusion—use context maps to define bounded context relationships and translation strategies.

### [Context Boundaries (Explicit Domain Context Separation)](./context-boundaries.md)

Mixing concepts from different domains creates confusion—establish explicit context boundaries with translation layers.

### [Saga Pattern (Long-Running Distributed Transactions)](./saga-pattern.md)

Distributed operations fail midway leaving inconsistent state—use sagas to coordinate multi-step workflows with compensation logic encoded in types.

## Performance Patterns

### [Memory Safety (Span and Ref Structs)](./memory-safety-span.md)

String manipulation causes GC pressure—use `Span<T>` and `ref struct` for zero-allocation, memory-safe code.

### [Struct Layout (Memory and Cache Optimization)](./struct-layout.md)

Structs with poor field ordering waste memory and cause cache misses—use explicit layout and field ordering for optimal performance.

### [Allocation Budget (Zero-Allocation Patterns)](./allocation-budget.md)

High-throughput code allocates excessively, triggering frequent GC pauses—use pooling, stack allocation, and budget tracking to eliminate allocations in hot paths.

### [Lazy Initialization (Type-Safe Deferred Computation)](./lazy-initialization.md)

Expensive initialization done eagerly when it might not be needed—use lazy initialization with type safety to defer computation until actually required.

## Async and Concurrency Patterns

### [Async Patterns (Task vs ValueTask)](./async-patterns.md)

Choose the right async return type—`Task<T>` for most cases, `ValueTask<T>` for hot paths with synchronous completions.

### [CancellationToken Handling](./cancellation-token-handling.md)

Support cancellation in async operations—respect timeouts, user cancellations, and resource cleanup.

## Exercises

### [Subscriptions Mini-Domain](./exercise-subscriptions.md)

A hands-on exercise applying multiple type-safety patterns to refactor a subscription service.

