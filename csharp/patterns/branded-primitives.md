# Branded Primitives (Nominal Typing Over Structural)

> Type aliases (`using`) provide no type safety—create branded types to distinguish structurally identical but semantically different values.

## Problem

C# `using` aliases are structural: `using CustomerId = int` and `using ProductId = int` are interchangeable at compile time. You can pass a `ProductId` where `CustomerId` expected without errors.

## Example

### ❌ Before

```csharp
// Type aliases don't create new types
using CustomerId = System.Int32;
using ProductId = System.Int32;
using OrderId = System.Int32;

public class OrderService
{
    public void PlaceOrder(CustomerId customerId, ProductId productId, OrderId orderId)
    {
        // All three parameters are just 'int' to the compiler
    }
}

// Compiles successfully but logically wrong
int customerId = 123;
int productId = 456;
int orderId = 789;

// Parameters can be swapped—compiler doesn't notice
svc.PlaceOrder(productId, orderId, customerId);  // ✓ Compiles! But wrong.
```

### ✅ After

```csharp
/// <summary>
/// Brand that makes a primitive type nominally distinct.
/// The brand exists only at compile time—zero runtime overhead.
/// </summary>
public interface IBrand<out TBrand> { }

/// <summary>
/// Branded primitive type.
/// Structurally identical to T, but nominally distinct by brand.
/// </summary>
public readonly record struct Branded<T, TBrand>(T Value) 
    : IBrand<TBrand>, IComparable<Branded<T, TBrand>>
    where T : IComparable<T>
{
    public int CompareTo(Branded<T, TBrand> other) => Value.CompareTo(other.Value);
    
    public override string ToString() => Value.ToString() ?? string.Empty;
    
    // Explicit conversion only
    public static explicit operator T(Branded<T, TBrand> branded) => branded.Value;
}

// Brand types (empty interfaces, exist only at compile time)
public interface ICustomerIdBrand { }
public interface IProductIdBrand { }
public interface IOrderIdBrand { }

// Branded types
public using CustomerId = Branded<int, ICustomerIdBrand>;
public using ProductId = Branded<int, IProductIdBrand>;
public using OrderId = Branded<int, IOrderIdBrand>;

public class OrderService
{
    public void PlaceOrder(CustomerId customerId, ProductId productId, OrderId orderId)
    {
        // Types are now distinct
    }
}

// Usage
var customerId = new CustomerId(123);
var productId = new ProductId(456);
var orderId = new OrderId(789);

svc.PlaceOrder(customerId, productId, orderId);  // ✓ Correct

// Won't compile: type mismatch
// svc.PlaceOrder(productId, orderId, customerId);  // ❌ Error!
```

## Branded Strings

```csharp
public interface IEmailBrand { }
public interface IUsernameBrand { }
public interface IUrlBrand { }

public using Email = Branded<string, IEmailBrand>;
public using Username = Branded<string, IUsernameBrand>;
public using Url = Branded<string, IUrlBrand>;

public class UserService
{
    public void SendWelcomeEmail(Email email, Username username)
    {
        // Cannot accidentally swap email and username
    }
}

// Factory methods with validation
public static class EmailExtensions
{
    public static Result<Email, string> CreateEmail(string value)
    {
        if (!value.Contains('@'))
            return Result<Email, string>.Failure("Invalid email");
        
        return Result<Email, string>.Success(new Email(value));
    }
}
```

## Branded Numerics with Constraints

```csharp
public interface IPositiveInt { }
public interface INonNegativeInt { }
public interface IPercentage { }

public using PositiveInt = Branded<int, IPositiveInt>;
public using NonNegativeInt = Branded<int, INonNegativeInt>;
public using Percentage = Branded<double, IPercentage>;

public static class PositiveIntExtensions
{
    public static Result<PositiveInt, string> Create(int value)
    {
        if (value <= 0)
            return Result<PositiveInt, string>.Failure("Value must be positive");
        
        return Result<PositiveInt, string>.Success(new PositiveInt(value));
    }
}

public static class PercentageExtensions
{
    public static Result<Percentage, string> Create(double value)
    {
        if (value < 0 || value > 100)
            return Result<Percentage, string>.Failure("Percentage must be 0-100");
        
        return Result<Percentage, string>.Success(new Percentage(value));
    }
}
```

## Why It's a Problem

1. **Type aliases are structural**: No compile-time distinction
2. **Easy to swap**: Parameters of same underlying type can be swapped
3. **No semantic meaning**: `int` doesn't convey domain concept

## Benefits

- **Nominal typing**: Types distinct even with same structure
- **Compile-time safety**: Cannot swap branded types
- **Zero overhead**: Brand exists only at compile time
- **Self-documenting**: Type name conveys meaning

## See Also

- [Strongly Typed IDs](./strongly-typed-ids.md) — ID wrapping
- [Primitive Obsession](./primitive-obsession.md) — avoiding primitives
- [Units of Measure](./units-of-measure.md) — physical units
