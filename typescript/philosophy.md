# Philosophy

> These patterns sit at the intersection of Domain-Driven Design, Functional Programming, and Type Theory.

This document explains the *why* behind our coding patterns. Each pattern in our catalog draws from one or more of these schools of thought.

---

## 1. Domain-Driven Design (Tactical Patterns)

The most immediate home for these practices is the "Tactical" side of DDD. This philosophy argues that **the code structure should match the business reality**.

### Core Concepts

**Value Objects**  
Replacing primitives with types that have identity defined by their value and enforce their own invariants.

```typescript
// Not this
const price: number = 19.99;

// This: a type that carries meaning and rules
const price = Money.usd(19.99);
```

**Ubiquitous Language**  
Using types (`CustomerId` vs `string`) ensures the code speaks the same language as the domain experts. When a developer reads `processOrder(customerId: CustomerId, productId: ProductId)`, they understand the domain—not just the data types.

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

```typescript
// ❌ Illegal state is representable
interface Order {
  shippedAt?: Date;
  trackingNumber?: string;  // Can have one without the other
}

// ✅ Illegal state is unrepresentable
interface PendingOrder {
  readonly kind: 'pending';
}

interface ShippedOrder {
  readonly kind: 'shipped';
  readonly shippedAt: Date;
  readonly trackingNumber: TrackingNumber;
}

type Order = PendingOrder | ShippedOrder;
```

### Parse, Don't Validate

Transform data into trusted types rather than returning booleans the caller might ignore.

```typescript
// ❌ Validate: caller must remember to check
function isValidAge(age: number): boolean;

// ✅ Parse: type carries the proof
function createAge(value: number): Result<Age, string>;
```

### Related Patterns

- [Enforcing Call Order](./patterns/enforcing-call-order.md)
- [Discriminated Unions](./patterns/discriminated-unions.md)
- [Static Factory Methods](./patterns/static-factory-methods.md)

---

## 3. Algebraic Data Types (ADTs)

A concept from Type Theory (and functional languages like Haskell/F#) that classifies how types are composed.

### Product Types (AND)

Types made by combining other types. A `User` has an ID **AND** a Name **AND** an Email.

```typescript
// Product type: all fields must be present
interface User {
  readonly id: UserId;
  readonly name: Name;
  readonly email: Email;
}
```

The Data Clumps refactoring creates Product Types—grouping values that belong together.

### Sum Types (OR)

Types that can be *one of* several different things. A `Payment` is Cash **OR** Credit **OR** BankTransfer.

```typescript
// Sum type: exactly one variant at a time (discriminated union)
type Payment =
  | { readonly kind: 'cash'; readonly amount: Money }
  | { readonly kind: 'credit'; readonly amount: Money; readonly card: CardNumber }
  | { readonly kind: 'bank'; readonly amount: Money; readonly account: AccountNumber };
```

Sum types eliminate invalid combinations. A cash payment can't accidentally have a `CardNumber`.

### Related Patterns

- [Data Clumps](./patterns/data-clump.md) — creates Product Types
- [Discriminated Unions](./patterns/discriminated-unions.md) — creates Sum Types
- [Flag Arguments](./patterns/flag-arguments.md) — polymorphism over boolean flags

---

## 4. Functional Programming (FP) Concepts

TypeScript embraces functional paradigms, and these patterns leverage them to increase safety.

### Honest Functions

A function signature should tell the truth. No hidden exceptions, no surprise nulls or undefined.

```typescript
// ❌ Dishonest: might throw or return undefined
function getUser(id: number): User;

// ✅ Honest: signature reveals all outcomes
function getUser(id: UserId): Result<User, GetUserError>;
```

See [Honest Functions](./patterns/honest-functions.md) for the Result pattern implementation.

### Immutability

Once a Value Object is created, it cannot be changed—eliminating side effects and race conditions.

```typescript
class Money {
  private constructor(
    readonly amount: number,
    readonly currency: Currency
  ) {}

  add(other: Money): Money {
    return new Money(this.amount + other.amount, this.currency);
  }
}
```

See [Value Semantics](./patterns/value-semantics.md) and [Readonly Types](./patterns/readonly-types.md).

### Total Functions

A function that handles *every* possible input via exhaustive type matching—no runtime `default` bombs.

```typescript
// ✅ Compiler enforces exhaustiveness
function getFee(payment: Payment): number {
  switch (payment.kind) {
    case 'cash': return 0;
    case 'credit': return 2.5;
    case 'bank': return 1.0;
    // TypeScript error if a case is missing
  }
}
```

### Related Patterns

- [Honest Functions](./patterns/honest-functions.md)
- [Discriminated Unions](./patterns/discriminated-unions.md)
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
- [Discriminated Unions](./patterns/discriminated-unions.md)

---

## 6. Semantic Compression

By introducing a type, you **compress knowledge** into a single symbol—the numeric value, unit, conversion logic, and formatting all encapsulated.

```typescript
// Without: scattered knowledge
const distanceInMeters = 1000;
const distanceInKm = distanceInMeters / 1000;

// With: knowledge encapsulated
const distance = Distance.fromMeters(1000);
const display = distance.toString();  // "1 km"
```

Developers no longer track "is this meters or feet?" or "did I divide correctly?"—the type handles it.

### Related Patterns

- [Primitive Obsession](./patterns/primitive-obsession.md)
- [Static Factory Methods](./patterns/static-factory-methods.md)

---

## 7. Security by Construction

> How do I design code such that violating security requires conscious sabotage, not a mistake?

Traditional security relies on developers remembering to check permissions. This is fragile—one forgotten `if (isAdmin())` creates a vulnerability. Security by Construction makes unauthorized actions **impossible to express** in the type system.

### Core Principles

| Principle | Description |
|-----------|-------------|
| **Minimize the TCB** | The Trusted Computing Base is the minimal code that must be correct for security to hold. Keep it small. |
| **Private Constructors** | If anyone can create a `Capability`, it's meaningless. Restrict construction to the authorization service. |
| **Cross Boundaries Once** | Validate at the edge, then pass trusted types internally—don't repeat checks. |
| **Capabilities, Not Claims** | A claim says who you are; a capability proves what you can do *right now*. |

```typescript
// ❌ Relies on developer remembering to check
function transferFunds(from: AccountId, to: AccountId, amount: Money): void;

// ✅ Impossible to call without authorization
function transferFunds(capability: Capability<TransferFunds>, action: TransferFunds): void;
```

> Security that survives refactors is not policy-heavy.  
> It's architecture that refuses to compile lies.

For implementation details, see [Capability Security](./patterns/capability-security.md).

### Related Patterns

- [Capability Security](./patterns/capability-security.md)
- [Enforcing Call Order](./patterns/enforcing-call-order.md)

---

## 8. Zero-Trust Intra-Code

> Never trust another part of your own codebase. Make each component prove its claims.

Traditional security focuses on external threats. Zero-trust intra-code applies the same suspicion *within* your codebase—every function call is a trust boundary.

### Zero-Trust Principles

| Principle | Instead of... | Require... |
|-----------|---------------|------------|
| **Don't trust parameters** | `processOrder(order: Order)` | `processOrder(order: ValidatedOrder)` |
| **Don't trust call order** | `ship(orderId: OrderId)` | `ship(order: ReservedOrder)` |
| **Don't trust the happy path** | `getUser(id)` returning undefined | `Result<User, Error>` with explicit handling |

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

---

## Summary

These eight philosophies reinforce each other:

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

When reviewing code or designing new features, ask these questions. The patterns in this catalog are tools to answer "yes" to each one.
