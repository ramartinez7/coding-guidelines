# Enum Best Practices

> Use enums for fixed sets of named constants, but consider alternatives for complex scenarios.

## Problem

Misusing enums leads to magic numbers, string comparisons, and inability to add behavior or data.

## Example

### ❌ Before

```csharp
public const int STATUS_PENDING = 0;
public const int STATUS_CONFIRMED = 1;
public const int STATUS_SHIPPED = 2;

public class Order
{
    public int Status { get; set; }  // Magic numbers!

    public void ProcessOrder()
    {
        if (Status == 1)  // What is 1?
        {
            // Process
        }
    }
}
```

### ✅ After

```csharp
public enum OrderStatus
{
    Pending = 0,
    Confirmed = 1,
    Shipped = 2,
    Delivered = 3
}

public class Order
{
    public OrderStatus Status { get; set; }

    public void ProcessOrder()
    {
        if (Status == OrderStatus.Confirmed)
        {
            // Process
        }
    }
}
```

## Best Practices

### 1. Use Enums for Fixed Constants

```csharp
// ✅ Enum for fixed set of values
public enum DayOfWeek
{
    Sunday,
    Monday,
    Tuesday,
    Wednesday,
    Thursday,
    Friday,
    Saturday
}

public enum OrderStatus
{
    Pending,
    Confirmed,
    Shipped,
    Delivered,
    Cancelled
}
```

### 2. Explicitly Set Enum Values

```csharp
// ❌ Implicit values (fragile)
public enum Priority
{
    Low,      // 0
    Medium,   // 1
    High      // 2
}

// ✅ Explicit values (stable for persistence)
public enum Priority
{
    Low = 1,
    Medium = 2,
    High = 3
}
```

### 3. Use [Flags] for Bit Fields

```csharp
// ✅ Flags enum for combinations
[Flags]
public enum Permissions
{
    None = 0,
    Read = 1,
    Write = 2,
    Delete = 4,
    Admin = 8,

    // Combinations
    ReadWrite = Read | Write,
    All = Read | Write | Delete | Admin
}

// Usage
var permissions = Permissions.Read | Permissions.Write;
if (permissions.HasFlag(Permissions.Write))
{
    // Can write
}
```

### 4. Don't Use Enum.Parse with Strings

```csharp
// ❌ Unsafe string parsing
var status = (OrderStatus)Enum.Parse(typeof(OrderStatus), "Pending");

// ✅ Use TryParse with error handling
if (Enum.TryParse<OrderStatus>("Pending", out var status))
{
    ProcessOrder(status);
}
else
{
    // Handle invalid value
}

// ✅ Or use smart enum pattern
public abstract class OrderStatus
{
    public static readonly OrderStatus Pending = new PendingStatus();
    public static readonly OrderStatus Confirmed = new ConfirmedStatus();

    private sealed class PendingStatus : OrderStatus { }
    private sealed class ConfirmedStatus : OrderStatus { }
}
```

### 5. Use Switch Expression with Enums

```csharp
// ✅ Switch expression for enum
public string GetStatusMessage(OrderStatus status) => status switch
{
    OrderStatus.Pending => "Order is pending",
    OrderStatus.Confirmed => "Order confirmed",
    OrderStatus.Shipped => "Order has shipped",
    OrderStatus.Delivered => "Order delivered",
    OrderStatus.Cancelled => "Order cancelled",
    _ => throw new ArgumentException($"Unknown status: {status}")
};
```

### 6. Don't Add Behavior to Enums (Use Smart Enums)

```csharp
// ❌ Can't add methods to enum
public enum PaymentMethod
{
    CreditCard,
    PayPal,
    BankTransfer
}

// ✅ Smart enum class for behavior
public abstract class PaymentMethod
{
    public static readonly PaymentMethod CreditCard = new CreditCardPayment();
    public static readonly PaymentMethod PayPal = new PayPalPayment();
    public static readonly PaymentMethod BankTransfer = new BankTransferPayment();

    public abstract decimal CalculateFee(decimal amount);

    private sealed class CreditCardPayment : PaymentMethod
    {
        public override decimal CalculateFee(decimal amount) => amount * 0.029m;
    }

    private sealed class PayPalPayment : PaymentMethod
    {
        public override decimal CalculateFee(decimal amount) => amount * 0.034m;
    }

    private sealed class BankTransferPayment : PaymentMethod
    {
        public override decimal CalculateFee(decimal amount) => 0m;
    }
}
```

### 7. Use Descriptive Names

```csharp
// ❌ Unclear names
public enum Status
{
    A,
    B,
    C
}

// ✅ Descriptive names
public enum SubscriptionStatus
{
    Active,
    Paused,
    Cancelled,
    Expired
}
```

### 8. Avoid Enum as Method Parameters (Use Types)

```csharp
// ❌ Enum parameter
public void SendEmail(string to, EmailType type)
{
    switch (type)
    {
        case EmailType.Welcome:
            SendWelcomeEmail(to);
            break;
        case EmailType.PasswordReset:
            SendPasswordResetEmail(to);
            break;
    }
}

// ✅ Specific types
public void SendWelcomeEmail(EmailAddress to) { }
public void SendPasswordResetEmail(EmailAddress to) { }
```

### 9. Use Enum.GetValues for Iteration

```csharp
// ✅ Iterate all enum values
foreach (var status in Enum.GetValues<OrderStatus>())
{
    Console.WriteLine($"{status}: {(int)status}");
}

// ✅ Get all names
var statusNames = Enum.GetNames<OrderStatus>();
```

### 10. Validate Enum Values

```csharp
// ✅ Validate enum input
public void SetStatus(OrderStatus status)
{
    if (!Enum.IsDefined(typeof(OrderStatus), status))
    {
        throw new ArgumentException($"Invalid status: {status}");
    }

    this.status = status;
}
```

### 11. Use EnumMember for Serialization

```csharp
// ✅ Control serialization with attributes
using System.Runtime.Serialization;

public enum OrderStatus
{
    [EnumMember(Value = "pending")]
    Pending,

    [EnumMember(Value = "confirmed")]
    Confirmed,

    [EnumMember(Value = "shipped")]
    Shipped
}
```

### 12. Consider Discriminated Unions Instead

```csharp
// ❌ Enum with related data in separate fields
public class Order
{
    public OrderStatus Status { get; set; }
    public DateTime? ShippedDate { get; set; }  // Only for Shipped
    public string? TrackingNumber { get; set; }  // Only for Shipped
    public string? CancellationReason { get; set; }  // Only for Cancelled
}

// ✅ Discriminated union (sealed hierarchy)
public abstract record OrderStatus;

public sealed record Pending : OrderStatus;

public sealed record Confirmed : OrderStatus;

public sealed record Shipped(
    DateTime ShippedDate,
    string TrackingNumber) : OrderStatus;

public sealed record Cancelled(
    string Reason) : OrderStatus;

public class Order
{
    public OrderStatus Status { get; set; }
    // Related data in the status itself!
}
```

## Symptoms

- Magic numbers instead of named constants
- String comparisons for status values
- Unable to add behavior to enum
- Fields that are only valid for certain enum values
- Difficulty extending enum without breaking changes

## Benefits

- **Type safety** over magic numbers
- **Readability** with named constants
- **IntelliSense support** for valid values
- **Compile-time checking** of values
- **Clear intent** in code

## See Also

- [Type-Safe Enumerations](./type-safe-enumerations.md) — Smart enum pattern
- [Discriminated Unions](./discriminated-unions.md) — Sum types for alternatives
- [Enum to Class Hierarchy](./enum-to-class-hierarchy.md) — Refactoring pattern
