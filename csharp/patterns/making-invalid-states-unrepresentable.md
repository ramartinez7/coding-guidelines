# Making Invalid States Unrepresentable

> The golden rule of type-driven design: If an invalid state cannot be represented in the type system, it cannot exist at runtime.

## Philosophy

Traditional programming defensively validates at runtime. Type-driven design shifts validation left to compile time by designing types where invalid states literally cannot exist.

> **"Well-typed programs cannot go wrong"** — Robin Milner

This pattern is the foundation of all type safety patterns in this catalog. Every pattern here applies this principle in a specific context.

## Core Principle

Instead of:
1. Create object
2. Validate it
3. Hope everyone remembers to validate

Do this:
1. Design types where invalidity is impossible
2. Enforce invariants at construction
3. Trust the type system

## Example: Order with Shipping

### ❌ Invalid States Are Representable

```csharp
public class Order
{
    public OrderId Id { get; set; }
    public List<OrderItem> Items { get; set; }
    public OrderStatus Status { get; set; }
    
    // Optional fields—can be null even when Status = Shipped
    public DateTime? ShippedAt { get; set; }
    public string? TrackingNumber { get; set; }
    public ShippingAddress? ShippingAddress { get; set; }
}

// Invalid states that compile but are wrong:
var order1 = new Order
{
    Status = OrderStatus.Shipped,
    ShippedAt = null,  // Shipped but no ship date?
    TrackingNumber = null  // Shipped but no tracking?
};

var order2 = new Order
{
    Status = OrderStatus.Pending,
    ShippedAt = DateTime.Now,  // Not shipped but has ship date?
    TrackingNumber = "ABC123"  // Not shipped but has tracking?
};

// Must validate everywhere:
void ProcessShippedOrder(Order order)
{
    if (order.Status != OrderStatus.Shipped)
        throw new InvalidOperationException("Order not shipped");
    
    if (order.ShippedAt is null)
        throw new InvalidOperationException("Shipped order missing ship date");
    
    if (order.TrackingNumber is null)
        throw new InvalidOperationException("Shipped order missing tracking");
    
    // Finally, do work...
}
```

### ✅ Invalid States Are Unrepresentable

```csharp
/// <summary>
/// Order states as sealed hierarchy—each state has exactly the data it needs.
/// </summary>
public abstract record OrderState
{
    private OrderState() { }
    
    /// <summary>
    /// Pending order—awaiting payment.
    /// </summary>
    public sealed record Pending : OrderState;
    
    /// <summary>
    /// Paid order—awaiting shipment.
    /// </summary>
    public sealed record Paid(PaymentId PaymentId, DateTime PaidAt) : OrderState;
    
    /// <summary>
    /// Shipped order—MUST have tracking and ship date.
    /// Impossible to create without these fields.
    /// </summary>
    public sealed record Shipped(
        PaymentId PaymentId,
        DateTime PaidAt,
        DateTime ShippedAt,
        TrackingNumber TrackingNumber) : OrderState;
    
    /// <summary>
    /// Delivered order—MUST have delivery confirmation.
    /// </summary>
    public sealed record Delivered(
        PaymentId PaymentId,
        DateTime PaidAt,
        DateTime ShippedAt,
        TrackingNumber TrackingNumber,
        DateTime DeliveredAt,
        string SignedBy) : OrderState;
}

public sealed record Order(
    OrderId Id,
    CustomerId CustomerId,
    IReadOnlyList<OrderItem> Items,
    ShippingAddress ShippingAddress,
    OrderState State)
{
    public static Order CreateNew(
        CustomerId customerId,
        IReadOnlyList<OrderItem> items,
        ShippingAddress shippingAddress)
    {
        return new Order(
            OrderId.NewId(),
            customerId,
            items,
            shippingAddress,
            new OrderState.Pending());
    }
}

// Invalid states CANNOT be created:
// ❌ Won't compile: Shipped requires DateTime and TrackingNumber
// var invalid = new OrderState.Shipped();

// ✅ Must provide all required data:
var validShipped = new OrderState.Shipped(
    paymentId,
    paidAt,
    DateTime.UtcNow,
    TrackingNumber.From("ABC123"));

// No validation needed—type guarantees correctness:
void ProcessShippedOrder(Order order)
{
    // Pattern match extracts data—compiler proves it exists
    if (order.State is not OrderState.Shipped shipped)
        throw new InvalidOperationException("Order not shipped");
    
    // These are guaranteed non-null by the type system:
    Console.WriteLine($"Shipped at {shipped.ShippedAt}");
    Console.WriteLine($"Tracking: {shipped.TrackingNumber}");
    
    // No null checks, no defensive coding
}
```

## Example: User Registration

### ❌ Invalid States Are Representable

```csharp
public class User
{
    public UserId Id { get; set; }
    public string Username { get; set; }
    public string Email { get; set; }
    public bool IsVerified { get; set; }
    
    // Optional—but when is it set? Type doesn't say.
    public string? VerificationToken { get; set; }
    public DateTime? VerifiedAt { get; set; }
}

// Invalid states:
var user1 = new User
{
    IsVerified = true,
    VerificationToken = "abc123",  // Verified but still has token?
    VerifiedAt = null  // Verified but no verification date?
};

var user2 = new User
{
    IsVerified = false,
    VerificationToken = null,  // Unverified but no token to send?
    VerifiedAt = DateTime.Now  // Unverified but has verification date?
};
```

### ✅ Invalid States Are Unrepresentable

```csharp
/// <summary>
/// User verification state—exactly one state at a time.
/// </summary>
public abstract record UserVerificationState
{
    private UserVerificationState() { }
    
    /// <summary>
    /// Unverified user with pending token.
    /// </summary>
    public sealed record Unverified(
        VerificationToken Token,
        DateTime TokenCreatedAt,
        DateTime TokenExpiresAt) : UserVerificationState;
    
    /// <summary>
    /// Verified user with proof of verification.
    /// </summary>
    public sealed record Verified(
        DateTime VerifiedAt) : UserVerificationState;
}

public sealed record User(
    UserId Id,
    Username Username,
    Email Email,
    UserVerificationState VerificationState)
{
    public bool IsVerified => VerificationState is UserVerificationState.Verified;
    
    public static User CreateUnverified(Username username, Email email)
    {
        var token = VerificationToken.Generate();
        var state = new UserVerificationState.Unverified(
            token,
            DateTime.UtcNow,
            DateTime.UtcNow.AddDays(3));
        
        return new User(UserId.NewId(), username, email, state);
    }
    
    public User MarkVerified()
    {
        // Can only verify unverified users
        if (VerificationState is not UserVerificationState.Unverified)
            throw new InvalidOperationException("User already verified");
        
        return this with
        {
            VerificationState = new UserVerificationState.Verified(DateTime.UtcNow)
        };
    }
}

// Impossible to create invalid states:
// ❌ Won't compile: Verified requires DateTime
// var invalid = new UserVerificationState.Verified();

// ✅ Must provide verification date:
var verified = new UserVerificationState.Verified(DateTime.UtcNow);
```

## Example: Required vs Optional

### ❌ Nullable Hides Intent

```csharp
public class Product
{
    // Is description optional or required? Type doesn't say.
    public string? Description { get; set; }
    
    // Is discount optional? Can it be zero? Type doesn't say.
    public decimal? Discount { get; set; }
    
    // Is category required? When is it null? Type doesn't say.
    public CategoryId? CategoryId { get; set; }
}

// Must validate everywhere:
void DisplayProduct(Product product)
{
    if (product.Description is null)
        Console.WriteLine("No description");
    else
        Console.WriteLine(product.Description);
    
    if (product.Discount.HasValue)
        Console.WriteLine($"Discount: {product.Discount.Value:P}");
}
```

### ✅ Types Express Requirements

```csharp
/// <summary>
/// Product with required and optional fields made explicit.
/// </summary>
public sealed record Product(
    ProductId Id,
    ProductName Name,
    Money Price,
    CategoryId CategoryId,  // Required—not nullable
    ProductDescription Description,  // Required—validated type
    Option<Discount> Discount)  // Explicitly optional
{
    public static Result<Product, ValidationError> Create(
        ProductName name,
        Money price,
        CategoryId categoryId,
        string descriptionText,
        Option<decimal> discountPercent)
    {
        var descResult = ProductDescription.Create(descriptionText);
        if (!descResult.IsSuccess)
            return Result<Product, ValidationError>.Failure(descResult.Error);
        
        var discount = discountPercent.Match(
            onSome: pct => Discount.Create(pct).ToOption(),
            onNone: () => Option<Discount>.None);
        
        return Result<Product, ValidationError>.Success(
            new Product(
                ProductId.NewId(),
                name,
                price,
                categoryId,
                descResult.Value,
                discount));
    }
}

/// <summary>
/// Description is always valid—validated at construction.
/// </summary>
public readonly record struct ProductDescription
{
    readonly string value;
    
    ProductDescription(string value)
    {
        this.value = value;
    }
    
    public static Result<ProductDescription, ValidationError> Create(string input)
    {
        if (string.IsNullOrWhiteSpace(input))
            return Result<ProductDescription, ValidationError>.Failure(
                new ValidationError("Description required"));
        
        if (input.Length > 5000)
            return Result<ProductDescription, ValidationError>.Failure(
                new ValidationError("Description too long"));
        
        return Result<ProductDescription, ValidationError>.Success(
            new ProductDescription(input));
    }
    
    public override string ToString() => value;
}

/// <summary>
/// Discount is always valid—validated at construction.
/// </summary>
public readonly record struct Discount
{
    readonly decimal percentage;
    
    Discount(decimal percentage)
    {
        this.percentage = percentage;
    }
    
    public static Result<Discount, ValidationError> Create(decimal input)
    {
        if (input <= 0 || input >= 1)
            return Result<Discount, ValidationError>.Failure(
                new ValidationError("Discount must be between 0 and 100%"));
        
        return Result<Discount, ValidationError>.Success(new Discount(input));
    }
    
    public decimal Percentage => percentage;
}

// No validation needed—types guarantee correctness:
void DisplayProduct(Product product)
{
    // Description guaranteed non-empty
    Console.WriteLine(product.Description.ToString());
    
    // Category guaranteed valid
    Console.WriteLine($"Category: {product.CategoryId}");
    
    // Discount explicitly optional—use pattern matching
    product.Discount.Match(
        onSome: discount => Console.WriteLine($"Discount: {discount.Percentage:P}"),
        onNone: () => Console.WriteLine("No discount"));
}
```

## Patterns That Apply This Principle

Every pattern in this catalog implements "making invalid states unrepresentable" in a specific context:

### Identity and Values
- **[Strongly Typed IDs](./strongly-typed-ids.md)** — Cannot swap different ID types
- **[Branded Primitives](./branded-primitives.md)** — Cannot mix structurally identical values
- **[Newtype Pattern](./newtype-pattern.md)** — Zero-cost distinct types

### State Machines
- **[Phantom Types](./phantom-types.md)** — Invalid state transitions don't compile
- **[Type-Level State Machines](./type-level-state-machines.md)** — State as type parameter
- **[Enum State Machine](./enum-state-machine.md)** — Each state is a distinct type

### Validation
- **[Smart Constructors](./smart-constructors.md)** — Cannot create invalid instances
- **[Refinement Types](./refinement-types.md)** — Structural constraints enforced
- **[Domain Invariants](./domain-invariants.md)** — Business rules enforced at construction

### Workflows
- **[Enforcing Call Order](./enforcing-call-order.md)** — Invalid sequences don't compile
- **[Type-Safe Workflow Modeling](./type-safe-workflow-modeling.md)** — Process steps as types
- **[Type-Safe Builder](./type-safe-builder.md)** — Required fields enforced

### Security
- **[Capability Security](./capability-security.md)** — Cannot call without authorization
- **[Authentication Context](./authentication-context.md)** — Cannot access user without auth
- **[Secret Types](./secret-types.md)** — Cannot accidentally log secrets

### Collections and Optionality
- **[Refinement Types](./refinement-types.md)** — Non-empty collections
- **[Nullability vs. Optionality](./nullability-optionality.md)** — Explicit optional values
- **[Ghost States](./ghost-states.md)** — Context-specific types

## The Three Questions

When designing a type, ask:

1. **What states are valid?**
   - Design types that can only represent those states

2. **What transitions are allowed?**
   - Design methods that only allow those transitions

3. **What invariants must hold?**
   - Enforce invariants at construction, not at usage

## Why It's Revolutionary

Traditional programming:
```
Create → Validate → Hope everyone validates → Runtime errors
```

Type-driven design:
```
Design types where invalid = impossible → Compile errors → Trust
```

The compiler becomes your QA team. Invalid states trigger compile errors, not production bugs.

## Core Techniques

### 1. Use Sealed Hierarchies for Variants

```csharp
// Exactly one variant, compiler enforces exhaustiveness
public abstract record State
{
    private State() { }
    public sealed record A : State;
    public sealed record B : State;
}
```

### 2. Private Constructors + Smart Factories

```csharp
// Cannot create invalid instances
public readonly record struct Email
{
    Email(string value) { /* ... */ }
    public static Result<Email, Error> Create(string input) { /* validate */ }
}
```

### 3. Phantom Types for State

```csharp
// State encoded in type parameter
public class Resource<TState> where TState : IState
{
    // Methods only available in specific states
}
```

### 4. Required Data in Constructor

```csharp
// Cannot forget required data
public sealed record Shipped(
    DateTime ShippedAt,  // Not nullable—MUST provide
    TrackingNumber Tracking);  // Not nullable—MUST provide
```

## Benefits

- **Compile-time safety**: Invalid states don't compile
- **Refactoring confidence**: Compiler finds all affected code
- **Documentation**: Types explain themselves
- **Testing reduction**: No need to test impossible states
- **Maintainability**: Changes break compilation, not production

## Trade-offs

- **Initial complexity**: More types, more upfront design
- **Learning curve**: Team must understand type-driven design
- **Verbosity**: More code than "just use strings"
- **Library limitations**: Some libraries expect primitives

## The Payoff

The cost is upfront design and learning. The benefit is:

1. Bugs caught at compile time instead of production
2. Refactoring without fear
3. Code that's impossible to misuse
4. Type system as documentation
5. Reduced testing burden

**The goal: Make wrong code look wrong by making it not compile.**

## See Also

- [Smart Constructors](./smart-constructors.md) — parse, don't validate
- [Phantom Types](./phantom-types.md) — state in type parameters
- [Type-Safe Workflow Modeling](./type-safe-workflow-modeling.md) — workflows as types
- [Domain Invariants](./domain-invariants.md) — business rules in types
