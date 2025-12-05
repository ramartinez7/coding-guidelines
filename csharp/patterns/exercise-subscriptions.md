# Exercise: Subscriptions Mini-Domain

> A hands-on exercise applying multiple type-safety patterns to a real-world scenario.

## The Challenge

Refactor the following code using the patterns from this catalog. Consider which patterns apply and where.

### ❌ Before

```csharp
public static class SubscriptionService
{
    public static void Activate(string subId, string email, string plan, decimal price, string currency, bool html)
    {
        Console.WriteLine($"Activate {subId} for {email} / {plan} @ {price} {currency} (html={html})");
    }

    public static void RecordPayment(string subId, decimal amount, string paymentMethod)
    {
        Console.WriteLine($"Payment for {subId}: {amount} {paymentMethod}");
    }

    public static void Cancel(string subId, string reason)
    {
        Console.WriteLine($"Canceled {subId}: {reason}");
    }
}
```

## Smells to Identify

Before looking at the solution, identify which patterns apply:

1. **Primitive Obsession**: Which `string` and `decimal` parameters represent domain concepts?
2. **Data Clumps**: Which parameters always travel together?
3. **Flag Arguments**: What does `bool html` control?
4. **Enum to Class Hierarchy**: Could `paymentMethod` benefit from being a type hierarchy?

---

## One Possible Solution

### ✅ After

```csharp
// Primitive Obsession → Domain Types
public readonly record struct SubscriptionId(string Value)
{
    public static SubscriptionId FromString(string value)
    {
        if (string.IsNullOrWhiteSpace(value))
            throw new ArgumentException("Subscription ID cannot be empty", nameof(value));
        return new SubscriptionId(value);
    }
}

public readonly record struct EmailAddress(string Value)
{
    public static EmailAddress FromString(string value)
    {
        if (string.IsNullOrWhiteSpace(value) || !value.Contains("@"))
            throw new ArgumentException("Invalid email", nameof(value));
        return new EmailAddress(value.Trim().ToLowerInvariant());
    }
}

public readonly record struct PlanName(string Value);

// Data Clump → Money type
public readonly record struct Money(decimal Amount, Currency Currency)
{
    public static Money FromDecimal(decimal amount, Currency currency)
    {
        if (amount < 0)
            throw new ArgumentException("Amount cannot be negative", nameof(amount));
        return new Money(amount, currency);
    }
}

public readonly record struct Currency(string Code)
{
    public static Currency USD => new("USD");
    public static Currency EUR => new("EUR");
    
    public static Currency FromString(string code)
    {
        if (string.IsNullOrWhiteSpace(code) || code.Length != 3)
            throw new ArgumentException("Currency must be 3-letter code", nameof(code));
        return new Currency(code.ToUpperInvariant());
    }
}

// Flag Argument → Polymorphism
public interface INotificationChannel
{
    void Send(EmailAddress to, string subject, string body);
}

public sealed class HtmlEmailChannel : INotificationChannel
{
    public void Send(EmailAddress to, string subject, string body)
    {
        Console.WriteLine($"Sending HTML email to {to.Value}: {subject}");
    }
}

public sealed class PlainTextEmailChannel : INotificationChannel
{
    public void Send(EmailAddress to, string subject, string body)
    {
        Console.WriteLine($"Sending plain text email to {to.Value}: {subject}");
    }
}

// Enum to Class Hierarchy → Payment Methods
public abstract record PaymentMethod
{
    public abstract void RecordPayment(SubscriptionId subscriptionId, Money amount);
}

public sealed record CreditCardPayment(string Last4Digits) : PaymentMethod
{
    public override void RecordPayment(SubscriptionId subscriptionId, Money amount)
    {
        Console.WriteLine($"Card ending {Last4Digits}: {amount.Amount} {amount.Currency.Code} for {subscriptionId.Value}");
    }
}

public sealed record BankTransferPayment(string AccountReference) : PaymentMethod
{
    public override void RecordPayment(SubscriptionId subscriptionId, Money amount)
    {
        Console.WriteLine($"Bank transfer from {AccountReference}: {amount.Amount} {amount.Currency.Code}");
    }
}

// Cancellation reason as a type (could be expanded)
public readonly record struct CancellationReason(string Value)
{
    public static CancellationReason UserRequested => new("User requested cancellation");
    public static CancellationReason PaymentFailed => new("Payment failed");
    public static CancellationReason Custom(string reason) => new(reason);
}

// Clean service with type-safe parameters
public sealed class SubscriptionService
{
    private readonly INotificationChannel _notificationChannel;

    public SubscriptionService(INotificationChannel notificationChannel)
    {
        _notificationChannel = notificationChannel;
    }

    public void Activate(SubscriptionId id, EmailAddress email, PlanName plan, Money price)
    {
        Console.WriteLine($"Activate {id.Value} for {email.Value} / {plan.Value} @ {price.Amount} {price.Currency.Code}");
        _notificationChannel.Send(email, "Welcome!", $"Your {plan.Value} subscription is active.");
    }

    public void RecordPayment(SubscriptionId id, Money amount, PaymentMethod method)
    {
        method.RecordPayment(id, amount);
    }

    public void Cancel(SubscriptionId id, CancellationReason reason)
    {
        Console.WriteLine($"Canceled {id.Value}: {reason.Value}");
    }
}
```

### Usage

```csharp
var service = new SubscriptionService(new HtmlEmailChannel());

var subId = SubscriptionId.FromString("SUB-12345");
var email = EmailAddress.FromString("user@example.com");
var plan = new PlanName("Pro");
var price = Money.FromDecimal(29.99m, Currency.USD);

service.Activate(subId, email, plan, price);
service.RecordPayment(subId, price, new CreditCardPayment("4242"));
service.Cancel(subId, CancellationReason.UserRequested);
```

---

## Discussion Questions

1. **Compile-time safety**: Where did the type system catch mistakes that used to be runtime-only?
   - Can't pass an email where a subscription ID is expected
   - Can't create negative money amounts
   - Can't mix up price and amount parameters

2. **Invariants in types**: Which invariants moved into types? Any left outside—why?
   - Email format validation → `EmailAddress`
   - Non-negative amounts → `Money`
   - Currency code format → `Currency`
   - Business rules (e.g., "can't cancel already-canceled subscription") might stay in service layer

3. **Clearer signatures**: Did method signatures become clearer and safer?
   - `Activate(SubscriptionId, EmailAddress, PlanName, Money)` vs `Activate(string, string, string, decimal, string, bool)`

4. **When is a new type overkill?**
   - Truly one-off values with no validation rules
   - Values that won't evolve or gain behavior
   - When the overhead outweighs the benefit (internal utilities, scripts)

5. **Performance: `readonly record struct` vs `record`?**
   - `readonly record struct`: Stack-allocated, no GC pressure, good for small value types (Money, Currency)
   - `record` (class): Heap-allocated, better for larger objects or when you need reference semantics

## Patterns Applied

- [Primitive Obsession](./primitive-obsession.md) → `SubscriptionId`, `EmailAddress`, `PlanName`, `Currency`
- [Data Clumps](./data-clump.md) → `Money` (amount + currency)
- [Flag Arguments](./flag-arguments.md) → `INotificationChannel` hierarchy
- [Enum to Class Hierarchy](./enum-to-class-hierarchy.md) → `PaymentMethod` hierarchy
- [Static Factory Methods](./static-factory-methods.md) → `FromString`, `FromDecimal` methods
