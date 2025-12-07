# Type-Safe Enumerations (Smart Enums)

> Using primitive enums loses type safety, behavior, and flexibility—use sealed class hierarchies to create rich, type-safe enumerations.

## Problem

C# enums are just named integers. They can be cast to any invalid value, can't carry data or behavior, don't support exhaustive matching, and break when you need more than a simple flag. Type-safe enumerations solve these problems using sealed class hierarchies.

> **Note:** Examples use `Result<T, E>` for error handling and custom domain types like `CardNumber`, `PaymentResult`, etc. These represent application-specific types that would be defined elsewhere in your domain model.

## Example

### ❌ Before (Primitive Enum)

```csharp
public enum PaymentMethod
{
    CreditCard = 1,
    DebitCard = 2,
    BankTransfer = 3,
    PayPal = 4
}

public class PaymentProcessor
{
    decimal CalculateFee(PaymentMethod method, decimal amount)
    {
        // Problems:
        // 1. Non-exhaustive: compiler won't warn about missing cases
        // 2. Can pass invalid enum: (PaymentMethod)999
        // 3. No data associated with method (like fee percentage)
        
        switch (method)
        {
            case PaymentMethod.CreditCard:
                return amount * 0.029m;
            case PaymentMethod.DebitCard:
                return amount * 0.015m;
            case PaymentMethod.BankTransfer:
                return 5.00m;
            // Forgot PayPal! Compiler doesn't warn.
            default:
                throw new ArgumentException("Unknown payment method");
        }
    }
    
    void ProcessPayment(PaymentMethod method, decimal amount, string? cardNumber)
    {
        // Problem: PayPal doesn't use card number, but it's required
        switch (method)
        {
            case PaymentMethod.CreditCard:
            case PaymentMethod.DebitCard:
                if (string.IsNullOrEmpty(cardNumber))
                    throw new ArgumentException("Card number required");
                ChargeCard(cardNumber, amount);
                break;
            case PaymentMethod.BankTransfer:
                InitiateBankTransfer(amount);
                break;
            case PaymentMethod.PayPal:
                // cardNumber parameter is meaningless here
                ChargePayPal(amount);
                break;
        }
    }
}

// Can create invalid enums
PaymentMethod invalid = (PaymentMethod)999;  // ✓ Compiles
CalculateFee(invalid, 100);  // Runtime error

// Can't tell what data each method needs
ProcessPayment(PaymentMethod.CreditCard, 100, cardNumber: null);  // Runtime error
```

### ✅ After (Type-Safe Enumeration)

```csharp
/// <summary>
/// Base class for payment method enumeration.
/// Sealed to prevent external extension.
/// </summary>
public abstract record PaymentMethod(string Name)
{
    // All cases defined as static readonly instances
    public static readonly CreditCardPayment CreditCard = new();
    public static readonly DebitCardPayment DebitCard = new();
    public static readonly BankTransferPayment BankTransfer = new();
    public static readonly PayPalPayment PayPal = new();
    
    // All possible values
    public static IReadOnlyList<PaymentMethod> All { get; } = new[]
    {
        CreditCard,
        DebitCard,
        BankTransfer,
        PayPal
    };
    
    // Parse from string
    public static Result<PaymentMethod, string> FromString(string name)
    {
        var method = All.FirstOrDefault(m => 
            m.Name.Equals(name, StringComparison.OrdinalIgnoreCase));
        
        return method is not null
            ? Result<PaymentMethod, string>.Success(method)
            : Result<PaymentMethod, string>.Failure($"Unknown payment method: {name}");
    }
    
    // Abstract behavior that each case must implement
    public abstract decimal CalculateFee(decimal amount);
}

/// <summary>
/// Credit card payment with card-specific logic.
/// </summary>
public sealed record CreditCardPayment() : PaymentMethod("CreditCard")
{
    public override decimal CalculateFee(decimal amount) => amount * 0.029m;
    
    public PaymentResult Process(decimal amount, CardNumber cardNumber, Cvv cvv)
    {
        // Card-specific validation
        if (amount <= 0)
            return PaymentResult.Failed("Invalid amount");
        
        // Process credit card
        return ChargeCard(cardNumber, cvv, amount);
    }
}

/// <summary>
/// Debit card payment with lower fees.
/// </summary>
public sealed record DebitCardPayment() : PaymentMethod("DebitCard")
{
    public override decimal CalculateFee(decimal amount) => amount * 0.015m;
    
    public PaymentResult Process(decimal amount, CardNumber cardNumber, Pin pin)
    {
        // Debit cards use PIN, not CVV
        return ChargeDebitCard(cardNumber, pin, amount);
    }
}

/// <summary>
/// Bank transfer with flat fee.
/// </summary>
public sealed record BankTransferPayment() : PaymentMethod("BankTransfer")
{
    public override decimal CalculateFee(decimal amount) => 5.00m;
    
    public PaymentResult Process(decimal amount, AccountNumber account, RoutingNumber routing)
    {
        // Bank-specific parameters
        return InitiateBankTransfer(account, routing, amount);
    }
}

/// <summary>
/// PayPal payment with OAuth token.
/// </summary>
public sealed record PayPalPayment() : PaymentMethod("PayPal")
{
    public override decimal CalculateFee(decimal amount) => amount * 0.032m + 0.30m;
    
    public PaymentResult Process(decimal amount, PayPalToken token)
    {
        // PayPal-specific authentication
        return ChargePayPal(token, amount);
    }
}

// Usage: Type system enforces correct parameters
public class PaymentProcessor
{
    // Each payment type has its own strongly-typed method
    public PaymentResult ProcessCreditCard(decimal amount, CardNumber card, Cvv cvv)
    {
        var fee = PaymentMethod.CreditCard.CalculateFee(amount);
        var total = amount + fee;
        
        return PaymentMethod.CreditCard.Process(total, card, cvv);
    }
    
    public PaymentResult ProcessBankTransfer(decimal amount, AccountNumber account, RoutingNumber routing)
    {
        var fee = PaymentMethod.BankTransfer.CalculateFee(amount);
        var total = amount + fee;
        
        return PaymentMethod.BankTransfer.Process(total, account, routing);
    }
    
    // Pattern matching is exhaustive and type-safe
    string GetDisplayName(PaymentMethod method) => method switch
    {
        CreditCardPayment => "Credit Card (2.9% fee)",
        DebitCardPayment => "Debit Card (1.5% fee)",
        BankTransferPayment => "Bank Transfer ($5 flat fee)",
        PayPalPayment => "PayPal (3.2% + $0.30 fee)",
        _ => throw new UnreachableException() // Compiler warning if case missed
    };
}

// Cannot create invalid payment methods
// PaymentMethod invalid = new PaymentMethod("Invalid");  // ❌ Won't compile: abstract

// Type-safe parsing
var result = PaymentMethod.FromString("CreditCard");
if (result.IsSuccess)
{
    var fee = result.Value.CalculateFee(100);  // Type-safe
}
```

## Enumeration with Associated Data

```csharp
/// <summary>
/// HTTP method with associated metadata.
/// </summary>
public abstract record HttpMethod(string Name, bool IsIdempotent, bool IsSafe)
{
    public static readonly GetMethod Get = new();
    public static readonly PostMethod Post = new();
    public static readonly PutMethod Put = new();
    public static readonly DeleteMethod Delete = new();
    public static readonly PatchMethod Patch = new();
    
    public static IReadOnlyList<HttpMethod> All { get; } = new HttpMethod[]
    {
        Get, Post, Put, Delete, Patch
    };
}

public sealed record GetMethod() 
    : HttpMethod("GET", IsIdempotent: true, IsSafe: true);

public sealed record PostMethod() 
    : HttpMethod("POST", IsIdempotent: false, IsSafe: false);

public sealed record PutMethod() 
    : HttpMethod("PUT", IsIdempotent: true, IsSafe: false);

public sealed record DeleteMethod() 
    : HttpMethod("DELETE", IsIdempotent: true, IsSafe: false);

public sealed record PatchMethod() 
    : HttpMethod("PATCH", IsIdempotent: false, IsSafe: false);

// Usage: Data is carried with each variant
void CacheResponse(HttpMethod method, Response response)
{
    // Can cache safe methods
    if (method.IsSafe)
    {
        cache.Store(response);
    }
}

bool CanRetry(HttpMethod method)
{
    // Idempotent methods can be safely retried
    return method.IsIdempotent;
}
```

## Enumeration with Behavior

```csharp
/// <summary>
/// Order status with state-specific behavior.
/// </summary>
public abstract record OrderStatus(string Name)
{
    public static readonly PendingStatus Pending = new();
    public static readonly ConfirmedStatus Confirmed = new();
    public static readonly ShippedStatus Shipped = new();
    public static readonly DeliveredStatus Delivered = new();
    public static readonly CancelledStatus Cancelled = new();
    
    // Abstract behavior each status must implement
    public abstract bool CanBeCancelled();
    public abstract bool CanBeShipped();
    public abstract string GetCustomerMessage();
}

public sealed record PendingStatus() : OrderStatus("Pending")
{
    public override bool CanBeCancelled() => true;
    public override bool CanBeShipped() => false;
    public override string GetCustomerMessage() => "Your order is being processed.";
}

public sealed record ConfirmedStatus() : OrderStatus("Confirmed")
{
    public override bool CanBeCancelled() => true;
    public override bool CanBeShipped() => true;
    public override string GetCustomerMessage() => "Your order is confirmed and will ship soon.";
}

public sealed record ShippedStatus() : OrderStatus("Shipped")
{
    public override bool CanBeCancelled() => false;
    public override bool CanBeShipped() => false;
    public override string GetCustomerMessage() => "Your order is on the way!";
}

public sealed record DeliveredStatus() : OrderStatus("Delivered")
{
    public override bool CanBeCancelled() => false;
    public override bool CanBeShipped() => false;
    public override string GetCustomerMessage() => "Your order has been delivered.";
}

public sealed record CancelledStatus() : OrderStatus("Cancelled")
{
    public override bool CanBeCancelled() => false;
    public override bool CanBeShipped() => false;
    public override string GetCustomerMessage() => "Your order was cancelled.";
}
```

## Comparison with Primitive Enums

| Feature | Primitive Enum | Type-Safe Enum |
|---------|----------------|----------------|
| Can cast to invalid value | ✅ Yes: `(Status)999` | ❌ No: sealed hierarchy |
| Associated data | ❌ No | ✅ Yes: each case can have fields |
| Associated behavior | ❌ No | ✅ Yes: each case can have methods |
| Exhaustive matching | ❌ No: `default` case needed | ✅ Yes: compiler warns |
| Parse from string | Manual with switch | Built-in with `FromString` |
| Iterate all values | `Enum.GetValues()` | `All` property |
| Type-safe variants | ❌ No | ✅ Yes: each case is distinct type |
| Serialization | Built-in | Custom converters |

## Serialization Support

```csharp
// JSON converter for type-safe enums
public class PaymentMethodConverter : JsonConverter<PaymentMethod>
{
    public override PaymentMethod Read(
        ref Utf8JsonReader reader,
        Type typeToConvert,
        JsonSerializerOptions options)
    {
        var name = reader.GetString();
        var result = PaymentMethod.FromString(name ?? "");
        
        return result.IsSuccess
            ? result.Value
            : throw new JsonException($"Unknown payment method: {name}");
    }
    
    public override void Write(
        Utf8JsonWriter writer,
        PaymentMethod value,
        JsonSerializerOptions options)
    {
        writer.WriteStringValue(value.Name);
    }
}

// Register converter
var options = new JsonSerializerOptions
{
    Converters = { new PaymentMethodConverter() }
};
```

## When to Use Each

### Use Primitive Enum When:
- Simple flags with no behavior
- Need [Flags] attribute for bitwise operations
- Performance-critical code (enums are integers)
- Interop with external APIs expecting integers

### Use Type-Safe Enum When:
- Each variant has different data or behavior
- Need exhaustive pattern matching
- Want to prevent invalid values
- Need rich domain modeling

## Benefits

- **Type safety**: Cannot create invalid instances
- **Rich behavior**: Each variant can have unique methods and data
- **Exhaustive matching**: Compiler warns about missing cases
- **Domain modeling**: Models complex business concepts accurately
- **Extensibility**: Easy to add new variants with breaking changes

## See Also

- [Enum to Class Hierarchy](./enum-to-class-hierarchy.md) — refactoring enums to classes
- [Boolean Blindness](./boolean-blindness.md) — avoiding boolean flags
- [Flag Arguments](./flag-arguments.md) — polymorphism over flags
- [Honest Functions](./honest-functions.md) — explicit return types
