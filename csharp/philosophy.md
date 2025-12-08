# Philosophy

> These patterns sit at the intersection of Domain-Driven Design, Functional Programming, and Type Theory.

This document explains the *why* behind our coding patterns. Each pattern in our catalog draws from one or more of these schools of thought.

---

## 1. Domain-Driven Design (Tactical Patterns)

The most immediate home for these practices is the "Tactical" side of DDD. This philosophy argues that **the code structure should match the business reality**.

### Core Concepts

**Value Objects**  
Replacing primitives with types that have identity defined by their value and enforce their own invariants.

```csharp
// Not this
decimal price = 19.99m;

// This: a type that carries meaning and rules
Money price = Money.USD(19.99m);
```

**Ubiquitous Language**  
Using types (`CustomerId` vs `int`) ensures the code speaks the same language as the domain experts. When a developer reads `ProcessOrder(CustomerId, ProductId)`, they understand the domain—not just the data types.

**Invariants**  
Encapsulating rules inside the constructor so the application can trust the data. Once you have an `Email`, it's guaranteed valid. See [Static Factory Methods](./patterns/static-factory-methods.md).

### Related Patterns

- [Primitive Obsession](./patterns/primitive-obsession.md)
- [Value Semantics](./patterns/value-semantics.md)
- [Strongly Typed IDs](./patterns/strongly-typed-ids.md)
- [Data Clumps](./patterns/data-clump.md)

---

## 2. Type-Driven Development (TyDD)

Distinct from Test-Driven Development (TDD), Type-Driven Development focuses on **modeling data structures before writing logic**. The philosophy is:

> "If it compiles, it probably works."

### The Golden Rule

**Make Illegal States Unrepresentable.**

Instead of checking for errors at runtime, you design types so that invalid states *cannot exist* in the code.

```csharp
// ❌ Illegal state is representable
public class Order
{
    public DateTime? ShippedAt { get; set; }
    public string? TrackingNumber { get; set; }  // Can have one without the other
}

// ✅ Illegal state is unrepresentable
public sealed record ShippedOrder(DateTime ShippedAt, TrackingNumber Tracking);
```

### Parse, Don't Validate

Transform data into trusted types rather than returning booleans the caller might ignore.

```csharp
// ❌ Validate: caller must remember to check
bool IsValidAge(int age);

// ✅ Parse: type carries the proof
Result<Age, string> Age.Create(int value);
```

### Related Patterns

- [Enforcing Call Order](./patterns/enforcing-call-order.md)
- [Enum State Machine](./patterns/enum-state-machine.md)
- [Ghost States](./patterns/ghost-states.md)
- [Static Factory Methods](./patterns/static-factory-methods.md)

---

## 3. Algebraic Data Types (ADTs)

A concept from Type Theory (and functional languages like Haskell/F#) that classifies how types are composed.

### Product Types (AND)

Types made by combining other types. A `User` has an ID **AND** a Name **AND** an Email.

```csharp
// Product type: all fields must be present
public sealed record User(UserId Id, Name Name, Email Email);
```

The Data Clumps refactoring creates Product Types—grouping values that belong together.

### Sum Types (OR)

Types that can be *one of* several different things. A `Payment` is Cash **OR** Credit **OR** BankTransfer.

```csharp
// Sum type: exactly one variant at a time
public abstract record Payment;
public sealed record CashPayment(Money Amount) : Payment;
public sealed record CreditPayment(Money Amount, CardNumber Card) : Payment;
public sealed record BankTransfer(Money Amount, AccountNumber Account) : Payment;
```

Sum types eliminate invalid combinations. A `CashPayment` can't accidentally have a `CardNumber`.

### Related Patterns

- [Data Clumps](./patterns/data-clump.md) — creates Product Types
- [Enum to Class Hierarchy](./patterns/enum-to-class-hierarchy.md) — creates Sum Types
- [Flag Arguments](./patterns/flag-arguments.md) — polymorphism over boolean flags

---

## 4. Functional Programming (FP) Concepts

Even in C# (an object-oriented language), these patterns borrow heavily from functional paradigms to increase safety.

### Honest Functions

A function signature should tell the truth. No hidden exceptions, no surprise nulls.

```csharp
// ❌ Dishonest: might throw or return null
User GetUser(int id);

// ✅ Honest: signature reveals all outcomes
Result<User, GetUserError> GetUser(UserId id);
```

See [Honest Functions](./patterns/honest-functions.md) for the Result pattern implementation.

### Immutability

Once a Value Object is created, it cannot be changed—eliminating side effects and race conditions.

```csharp
public readonly record struct Money(decimal Amount, Currency Currency)
{
    public Money Add(Money other) => this with { Amount = Amount + other.Amount };
}
```

See [Value Semantics](./patterns/value-semantics.md) and [Snapshot Immutability](./patterns/snapshot-immutability.md).

### Total Functions

A function that handles *every* possible input via exhaustive type matching—no runtime `default` bombs.

```csharp
// ✅ Compiler enforces exhaustiveness
decimal GetFee(Payment payment) => payment switch
{
    CashPayment => 0m,
    CreditPayment => 2.5m,
    BankTransfer => 1.0m
};
```

### Related Patterns

- [Honest Functions](./patterns/honest-functions.md)
- [Nullability vs. Optionality](./patterns/nullability-optionality.md)
- [Value Semantics](./patterns/value-semantics.md)
- [Boolean Blindness](./patterns/boolean-blindness.md)

---

## 5. Correctness by Construction

Instead of testing that code is correct, **construct it so it cannot be incorrect**. Rely on the compiler to catch logic errors before production.

| Stage | Feedback Time | Cost to Fix |
|-------|---------------|-------------|
| Production | Hours/Days | $$$$$ |
| CI Tests | Minutes | $$$ |
| **Compile Time** | **Instant** | **$** |

Every pattern in this catalog aims to shift errors left—catching them earlier.

### Related Patterns

- [Strongly Typed IDs](./patterns/strongly-typed-ids.md)
- [Enforcing Call Order](./patterns/enforcing-call-order.md)
- [Enum State Machine](./patterns/enum-state-machine.md)

---

## 6. Semantic Compression

By introducing a type, you **compress knowledge** into a single symbol—the numeric value, unit, conversion logic, and formatting all encapsulated.

```csharp
// Without: scattered knowledge
double distanceInMeters = 1000;
double distanceInKm = distanceInMeters / 1000;

// With: knowledge encapsulated
Distance distance = Distance.FromMeters(1000);
string display = distance.ToString();  // "1 km"
```

Developers no longer track "is this meters or feet?" or "did I divide correctly?"—the type handles it.

### Related Patterns

- [Primitive Obsession](./patterns/primitive-obsession.md)
- [Primitive Collections](./patterns/primitive-collections.md)
- [Static Factory Methods](./patterns/static-factory-methods.md)

---

## 7. Security by Construction

> How do I design code such that violating security requires conscious sabotage, not a mistake?

Traditional security relies on developers remembering to check permissions. This is fragile—one forgotten `if (IsAdmin())` creates a vulnerability. Security by Construction makes unauthorized actions **impossible to express** in the type system.

### Core Principles

| Principle | Description |
|-----------|-------------|
| **Minimize the TCB** | The Trusted Computing Base is the minimal code that must be correct for security to hold. Keep it small. |
| **Private Constructors** | If anyone can create a `Capability`, it's meaningless. Restrict construction to the authorization service. |
| **Cross Boundaries Once** | Validate at the edge, then pass trusted types internally—don't repeat checks. |
| **Capabilities, Not Claims** | A claim says who you are; a capability proves what you can do *right now*. |

```csharp
// ❌ Relies on developer remembering to check
public void TransferFunds(AccountId from, AccountId to, Money amount) { }

// ✅ Impossible to call without authorization
public void TransferFunds(Capability<TransferFunds> capability, TransferFunds action) { }
```

> Security that survives refactors is not policy-heavy.  
> It's architecture that refuses to compile lies.

For implementation details, see [Capability Security](./patterns/capability-security.md).

### Related Patterns

- [Capability Security](./patterns/capability-security.md)
- [Nullable Reference Types](./patterns/nullable-reference-types.md)
- [Enforcing Call Order](./patterns/enforcing-call-order.md)

---

## 8. Zero-Trust Intra-Code

> Never trust another part of your own codebase. Make each component prove its claims.

Traditional security focuses on external threats. Zero-trust intra-code applies the same suspicion *within* your codebase—every function call is a trust boundary.

### Zero-Trust Principles

| Principle | Instead of... | Require... |
|-----------|---------------|------------|
| **Don't trust parameters** | `ProcessOrder(Order order)` | `ProcessOrder(ValidatedOrder order)` |
| **Don't trust call order** | `Ship(OrderId id)` | `Ship(ReservedOrder order)` |
| **Don't trust assemblies** | Namespace-only separation | [Real assembly isolation](./patterns/assembly-isolation.md) |
| **Don't trust the happy path** | `GetUser(id)` returning null | `Result<User, Error>` with explicit handling |

### Trust Boundaries = Type Boundaries

| Layer | Accepts | Returns |
|-------|---------|--------|
| API | Raw DTOs, strings, JSON | `ValidatedRequest` or `Error` |
| Application | Validated commands, Capabilities | Domain events, Results |
| Domain | Value objects, typed state | New state, domain events |

Each boundary transforms "untrusted" into "trusted" types. Downstream layers don't re-validate.

### Related Patterns

- [Capability Security](./patterns/capability-security.md) — proof of authorization
- [Enforcing Call Order](./patterns/enforcing-call-order.md) — proof of sequence
- [Static Factory Methods](./patterns/static-factory-methods.md) — proof of validity
- [Assemblies vs. Namespaces](./patterns/assembly-isolation.md) — real isolation

---

## 9. Clean Architecture (Dependency Management)

> How do I organize code so that business logic is independent of frameworks, UI, and infrastructure?

Clean Architecture (also called Onion Architecture or Hexagonal Architecture) is an architectural pattern that complements DDD by organizing code into layers with strict dependency rules.

### The Dependency Rule

> Source code dependencies must point only inward, toward higher-level policies.

```
┌─────────────────────────────────────┐
│    Infrastructure Layer             │  ← Frameworks, DB, External APIs
│  ┌─────────────────────────────┐    │
│  │    Application Layer        │    │  ← Use Cases, Orchestration
│  │  ┌─────────────────────┐    │    │
│  │  │   Domain Layer      │    │    │  ← Business Logic & Rules
│  │  │  (Core Business)    │    │    │
│  │  └─────────────────────┘    │    │
│  └─────────────────────────────┘    │
└─────────────────────────────────────┘

Dependencies flow INWARD only →
```

### Core Principles

| Principle | Description |
|-----------|-------------|
| **Independent of frameworks** | Business logic doesn't depend on ASP.NET, EF, or any framework |
| **Testable** | Business rules can be tested without UI, database, web server |
| **Independent of UI** | Can swap Web UI for Console UI without changing business logic |
| **Independent of database** | Can swap SQL for NoSQL without changing business logic |
| **Independent of external agencies** | Business rules don't know about external services |

### Layers Explained

**Domain Layer (Center)**
- Pure business logic and rules
- Entities, value objects, domain services
- No dependencies on anything
- Example: `Order.AddItem()`, `Customer.DeactivateAccount()`

**Application Layer (Middle)**
- Use cases / application services
- Orchestrates domain objects
- Depends only on Domain
- Example: `PlaceOrderUseCase`, `RegisterUserUseCase`

**Infrastructure Layer (Outer)**
- Implementations of interfaces defined in inner layers
- Database access, external APIs, email services
- Depends on Application and Domain
- Example: `OrderRepository`, `SendGridEmailService`

**Presentation Layer (Outermost)**
- HTTP controllers, CLI handlers, gRPC services
- Translates external requests to use case commands
- Depends on Application (and Infrastructure for DI setup)
- Example: `OrdersController.Post()` → `PlaceOrderUseCase.ExecuteAsync()`

### Dependency Inversion

Inner layers define **interfaces** for what they need. Outer layers provide **implementations**.

```csharp
// Domain layer defines what it needs
namespace Domain.Interfaces
{
    public interface IOrderRepository
    {
        Task<Option<Order>> GetByIdAsync(OrderId id);
    }
}

// Infrastructure layer implements it
namespace Infrastructure.Persistence
{
    public class OrderRepository : IOrderRepository
    {
        // EF implementation - Domain doesn't know about this
    }
}
```

### Benefits

- **Framework independence**: Swap ASP.NET for gRPC, EF for Dapper—business logic unchanged
- **Testability**: Test business logic without spinning up database or web server
- **Parallel development**: Teams work on UI, domain, and infrastructure independently
- **Deferrable decisions**: Delay choosing database or framework until you have more information
- **Maintainability**: Changes to infrastructure don't affect business logic

### Related Patterns

- [Clean Architecture](./patterns/clean-architecture.md) — detailed implementation
- [Use Cases / Application Services](./patterns/use-cases.md) — application layer pattern
- [Screaming Architecture](./patterns/screaming-architecture.md) — project structure that reveals intent
- [Dependency Rule Enforcement](./patterns/dependency-rule-enforcement.md) — compile-time checks
- [Repository Pattern](./patterns/repository-pattern.md) — data access abstraction
- [Anti-Corruption Layer](./patterns/anti-corruption-layer.md) — protecting domain boundaries

---

## Summary

These nine philosophies reinforce each other:

| Philosophy | Key Question |
|------------|--------------|
| **DDD** | Does the code match the business domain? |
| **TyDD** | Can invalid states exist? |
| **ADTs** | Is this an AND or an OR? |
| **FP** | Does the signature tell the whole truth? |
| **Correctness by Construction** | Will the compiler catch this mistake? |
| **Semantic Compression** | Is knowledge scattered or encapsulated? |
| **Security by Construction** | Does violating security require sabotage, not a mistake? |
| **Zero-Trust Intra-Code** | Does this component prove its claims, or just trust? |
| **Clean Architecture** | Is business logic independent of frameworks and infrastructure? |

When reviewing code or designing new features, ask these questions. The patterns in this catalog are tools to answer "yes" to each one.
