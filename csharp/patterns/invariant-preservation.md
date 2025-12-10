# Invariant Preservation (Maintaining Business Rules Throughout Entity Lifecycle)

> Entities that can violate invariants after construction create bugs—preserve invariants at all times through encapsulation and immutability.

## Problem

Public setters allow invariants to be broken after entity creation, leading to invalid states.

## Example

### ❌ Before

```csharp
public class BankAccount
{
    public decimal Balance { get; set; }  // Can be set to anything!
    public decimal OverdraftLimit { get; set; }
    
    // Invariant: Balance >= -OverdraftLimit
    // But nothing enforces this!
}

var account = new BankAccount { Balance = 100, OverdraftLimit = 50 };
account.Balance = -100;  // Violates invariant!
```

### ✅ After

```csharp
public sealed class BankAccount
{
    public Money Balance { get; private set; }
    public Money OverdraftLimit { get; }
    
    private BankAccount(Money balance, Money overdraftLimit)
    {
        // Invariant enforced at construction
        if (balance.Amount < -overdraftLimit.Amount)
            throw new ArgumentException("Balance violates overdraft limit");
        
        Balance = balance;
        OverdraftLimit = overdraftLimit;
    }
    
    public static Result<BankAccount, string> Create(Money initialBalance, Money overdraftLimit)
    {
        if (initialBalance.Amount < 0)
            return Result<BankAccount, string>.Failure("Initial balance cannot be negative");
        
        if (overdraftLimit.Amount < 0)
            return Result<BankAccount, string>.Failure("Overdraft limit cannot be negative");
        
        return Result<BankAccount, string>.Success(
            new BankAccount(initialBalance, overdraftLimit));
    }
    
    public Result<Unit, string> Withdraw(Money amount)
    {
        // Invariant preserved: check before modifying
        if (Balance.Amount - amount.Amount < -OverdraftLimit.Amount)
            return Result<Unit, string>.Failure("Insufficient funds");
        
        Balance = Money.FromAmount(Balance.Amount - amount.Amount, Balance.Currency);
        return Result<Unit, string>.Success(Unit.Value);
    }
}
```

## See Also

- [Domain Invariants](./domain-invariants.md)
- [Aggregate Roots](./aggregate-roots.md)
