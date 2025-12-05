# Enum to Class Hierarchy

> Enums that determine which parameters are meaningful—model variants as distinct types instead.

## Problem

Enums combined with conditional logic create invalid states that compile but fail at runtime. Parameters become meaningless for certain enum values, and the compiler can't enforce which combinations are valid.

## Example

### ❌ Before

```csharp
public enum PaymentMethod
{
    CreditCard,
    BankTransfer,
    Cash
}

public class PaymentProcessor
{
    public void Process(decimal amount, PaymentMethod method, string? cardNumber, string? bankAccount)
    {
        if (method == PaymentMethod.CreditCard)
            Console.WriteLine($"Charging card {cardNumber} for ${amount}");
        else if (method == PaymentMethod.BankTransfer)
            Console.WriteLine($"Transferring ${amount} from {bankAccount}");
        else if (method == PaymentMethod.Cash)
            Console.WriteLine($"Accepting ${amount} in cash");
    }
}

// These compile but are wrong:
processor.Process(100m, PaymentMethod.CreditCard, null, null);           // Missing card!
processor.Process(100m, PaymentMethod.BankTransfer, "4111...", null);    // Wrong parameter used
processor.Process(10m, PaymentMethod.Cash, "4111...", "ACCT-123");       // Meaningless params
```

### ✅ After

```csharp
public abstract record Payment(decimal Amount)
{
    public abstract void Process();
}

public sealed record CreditCardPayment(decimal Amount, string CardNumber) : Payment(Amount)
{
    public override void Process() => Console.WriteLine($"Charging card {CardNumber} for ${Amount}");
}

public sealed record BankTransferPayment(decimal Amount, string BankAccount) : Payment(Amount)
{
    public override void Process() => Console.WriteLine($"Transferring ${Amount} from {BankAccount}");
}

public sealed record CashPayment(decimal Amount) : Payment(Amount)
{
    public override void Process() => Console.WriteLine($"Accepting ${Amount} in cash");
}

// Invalid states are now unrepresentable:
var card = new CreditCardPayment(100m, "4111-1111-1111-1111");  // Card number required
var transfer = new BankTransferPayment(75m, "ACCT-12345");       // Bank account required
var cash = new CashPayment(10m);                                  // No extra params needed

card.Process();
transfer.Process();
cash.Process();
```

## Why It's a Problem

1. **Invalid states compile**: Nothing prevents passing `null` for required data or providing irrelevant parameters.

2. **Meaningless parameters**: For `Cash`, both `cardNumber` and `bankAccount` are noise—confusing readers and inviting bugs.

3. **Scattered validation**: Each method using the enum must validate the correct parameters are present.

## Symptoms

- Parameters that are only meaningful with certain enum values
- `null` passed for parameters that "don't apply" to a particular case
- Static methods analyzing enum values for business rules (e.g., `bool IsBusinessDay(DayOfWeek day)`)
- Extension methods defined on enum types to add behavior
- Switch statements or if-else chains on enum values scattered throughout the codebase

## Benefits

- **Invalid states are unrepresentable**: The compiler enforces required data per variant
- **Self-documenting**: Each type clearly shows what data it needs
- **Behavior with data**: Each variant can have its own methods and logic
- **Exhaustiveness checking**: Pattern matching warns if you miss a case (with sealed hierarchy)

## Variation: Simpler Interface Approach

```csharp
public interface IPayment
{
    decimal Amount { get; }
    void Process();
}

public sealed class CreditCardPayment : IPayment { /* ... */ }
public sealed class BankTransferPayment : IPayment { /* ... */ }
public sealed class CashPayment : IPayment { /* ... */ }
```

## See Also

- [Flag Arguments](./flag-arguments.md)
- [Primitive Obsession](./primitive-obsession.md)
