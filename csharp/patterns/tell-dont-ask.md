# Tell, Don't Ask

> Tell objects what to do—don't ask for their state and make decisions for them.

## Problem

When code queries an object for its state and then makes decisions based on that state, it violates encapsulation. Business logic gets scattered across callers instead of living with the data, leading to duplicate code and fragile designs.

## Example

### ❌ Before

```csharp
public class BankAccount
{
    public decimal Balance { get; set; }
    public AccountStatus Status { get; set; }
    public decimal OverdraftLimit { get; set; }
}

public class TransferService
{
    public void Transfer(BankAccount from, BankAccount to, decimal amount)
    {
        // Asking for state and making decisions
        if (from.Status != AccountStatus.Active)
            throw new InvalidOperationException("Source account is not active");
        
        if (to.Status != AccountStatus.Active)
            throw new InvalidOperationException("Destination account is not active");
        
        if (from.Balance - amount < -from.OverdraftLimit)
            throw new InvalidOperationException("Insufficient funds");
        
        // Directly manipulating state
        from.Balance -= amount;
        to.Balance += amount;
    }
}
```

### ✅ After

```csharp
public sealed class BankAccount
{
    public Money Balance { get; private set; }
    public AccountStatus Status { get; private set; }
    public Money OverdraftLimit { get; }

    private BankAccount(Money balance, Money overdraftLimit, AccountStatus status)
    {
        Balance = balance;
        OverdraftLimit = overdraftLimit;
        Status = status;
    }

    public static Result<BankAccount, string> CreateAccount(Money initialBalance, Money overdraftLimit)
    {
        if (initialBalance.Amount < 0)
            return Result<BankAccount, string>.Failure("Initial balance cannot be negative");
        
        if (overdraftLimit.Amount < 0)
            return Result<BankAccount, string>.Failure("Overdraft limit cannot be negative");
        
        return Result<BankAccount, string>.Success(
            new BankAccount(initialBalance, overdraftLimit, AccountStatus.Active));
    }

    public Result<Unit, string> Withdraw(Money amount)
    {
        if (Status != AccountStatus.Active)
            return Result<Unit, string>.Failure("Account is not active");
        
        var newBalance = Balance - amount;
        if (newBalance.Amount < -OverdraftLimit.Amount)
            return Result<Unit, string>.Failure("Insufficient funds");
        
        Balance = newBalance;
        return Result<Unit, string>.Success(Unit.Value);
    }

    public Result<Unit, string> Deposit(Money amount)
    {
        if (Status != AccountStatus.Active)
            return Result<Unit, string>.Failure("Account is not active");
        
        if (amount.Amount <= 0)
            return Result<Unit, string>.Failure("Deposit amount must be positive");
        
        Balance = Balance + amount;
        return Result<Unit, string>.Success(Unit.Value);
    }

    public void Suspend()
    {
        Status = AccountStatus.Suspended;
    }

    public void Activate()
    {
        Status = AccountStatus.Active;
    }
}

public class TransferService
{
    public Result<Unit, string> Transfer(BankAccount from, BankAccount to, Money amount)
    {
        // Tell the accounts what to do
        var withdrawResult = from.Withdraw(amount);
        if (withdrawResult.IsFailure)
            return withdrawResult;
        
        var depositResult = to.Deposit(amount);
        if (depositResult.IsFailure)
        {
            // Rollback the withdrawal
            from.Deposit(amount);
            return depositResult;
        }
        
        return Result<Unit, string>.Success(Unit.Value);
    }
}
```

## Why It's a Problem

1. **Broken encapsulation**: Business logic leaks out of the object into its callers.

2. **Duplicate code**: Every caller must reimplement the same decision logic.

3. **Fragile design**: Changes to business rules require updating all callers.

4. **Poor cohesion**: Data and behavior that belong together are separated.

5. **Testing difficulty**: Must test the same logic in multiple places.

## The "Ask" Smell

Code that "asks" typically follows this pattern:

```csharp
// 1. Ask for state
if (order.Status == OrderStatus.Pending)
{
    // 2. Make a decision
    if (order.Items.Count > 0 && order.Total > 0)
    {
        // 3. Modify state
        order.Status = OrderStatus.Confirmed;
        order.ConfirmedAt = DateTime.UtcNow;
    }
}
```

This should become:

```csharp
// Tell the order what to do
var result = order.Confirm();
```

## Benefits

- **Encapsulation**: Business logic stays with the data it operates on
- **Single source of truth**: Rules exist in one place
- **Easier to maintain**: Changes happen in one location
- **Better testability**: Test the object's behavior, not its state
- **Clearer intent**: Method names express what happens, not how

## Balancing with Queries

"Tell, Don't Ask" doesn't mean you can never query an object. Read-only queries for display or reporting are fine:

```csharp
// ✅ OK: Read-only query for display
decimal balance = account.GetBalance();
Console.WriteLine($"Current balance: {balance:C}");

// ❌ Bad: Query state to make decisions
if (account.GetBalance() < 100)
{
    account.SetBalance(100);  // Deciding behavior externally
}

// ✅ Good: Tell the object what to do
var result = account.EnsureMinimumBalance(Money.USD(100));
```

## Common Scenarios

### Scenario 1: Conditional Logic

```csharp
// ❌ Ask
if (user.IsActive && user.HasPermission(Permission.Write))
{
    document.UpdateContent(newContent);
    document.SetLastModifiedBy(user.Id);
}

// ✅ Tell
var result = document.UpdateContent(user, newContent);
```

### Scenario 2: State Transitions

```csharp
// ❌ Ask
if (order.Status == OrderStatus.Pending && order.PaymentReceived)
{
    order.Status = OrderStatus.Processing;
    order.ProcessedAt = DateTime.UtcNow;
}

// ✅ Tell
var result = order.BeginProcessing();
```

### Scenario 3: Calculations

```csharp
// ❌ Ask
decimal subtotal = 0;
foreach (var item in order.Items)
{
    subtotal += item.Price * item.Quantity;
}
decimal tax = subtotal * order.TaxRate;
decimal total = subtotal + tax + order.ShippingCost;

// ✅ Tell
var total = order.CalculateTotal();
```

## When to Break the Rule

Sometimes querying is appropriate:

1. **Display/Reporting**: Reading state for UI or reports without making decisions
2. **Cross-aggregate coordination**: When orchestrating multiple aggregates in a use case
3. **Specifications/Queries**: When filtering or searching collections

```csharp
// OK: Querying for filtering
var activeUsers = users.Where(u => u.IsActive);

// OK: Reporting
var report = new SalesReport
{
    TotalSales = orders.Sum(o => o.GetTotal()),
    OrderCount = orders.Count
};
```

## See Also

- [Domain Invariants](./domain-invariants.md) — enforcing business rules at construction
- [Specification Pattern](./specification-pattern.md) — encapsulating business rules as types
- [Entities vs Value Objects](./entities-vs-value-objects.md) — where to put behavior
- [Aggregate Roots](./aggregate-roots.md) — enforcing invariants through boundaries
- [Use Cases](./use-cases.md) — orchestrating domain logic
