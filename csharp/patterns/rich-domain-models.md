# Rich Domain Models (Behavior-Rich Entities vs Anemic Models)

> Entities with only getters/setters and no behavior push business logic into services—create rich domain models that encapsulate both data and the operations that manipulate it.

## Problem

Domain entities are reduced to data containers with public setters, while all business logic lives in service classes. This violates encapsulation and scatters business rules throughout the codebase.

## Example

### ❌ Before (Anemic Domain Model)

```csharp
// Anemic entity: Just data, no behavior
public class BankAccount
{
    public Guid Id { get; set; }
    public string AccountNumber { get; set; }
    public decimal Balance { get; set; }
    public bool IsActive { get; set; }
    public DateTime OpenedAt { get; set; }
}

// All business logic in service
public class BankAccountService
{
    private readonly IBankAccountRepository _repo;
    
    public async Task<bool> Withdraw(Guid accountId, decimal amount)
    {
        var account = await _repo.GetByIdAsync(accountId);
        
        // Business rules in service, not entity
        if (!account.IsActive)
            return false;
        
        if (account.Balance < amount)
            return false;
        
        if (amount <= 0)
            return false;
        
        // Direct property manipulation
        account.Balance -= amount;
        
        await _repo.SaveAsync(account);
        return true;
    }
    
    public async Task<bool> Deposit(Guid accountId, decimal amount)
    {
        var account = await _repo.GetByIdAsync(accountId);
        
        // Business rules duplicated
        if (!account.IsActive)
            return false;
        
        if (amount <= 0)
            return false;
        
        account.Balance += amount;
        
        await _repo.SaveAsync(account);
        return true;
    }
    
    public async Task<bool> Close(Guid accountId)
    {
        var account = await _repo.GetByIdAsync(accountId);
        
        // Business rule
        if (account.Balance != 0)
            return false;
        
        account.IsActive = false;
        
        await _repo.SaveAsync(account);
        return true;
    }
}
```

**Problems:**
- Business logic scattered across services
- Entity is just a data bag
- Rules can be bypassed by setting properties directly
- Difficult to discover what operations are valid
- No encapsulation of invariants

### ✅ After (Rich Domain Model)

```csharp
// Rich entity: Data + Behavior
public sealed class BankAccount
{
    public AccountId Id { get; }
    public AccountNumber Number { get; }
    public Money Balance { get; private set; }
    public AccountStatus Status { get; private set; }
    public DateTimeOffset OpenedAt { get; }
    private readonly List<Transaction> _transactions = new();
    public IReadOnlyList<Transaction> Transactions => _transactions.AsReadOnly();
    
    // Private constructor enforces creation through factory
    private BankAccount(
        AccountId id,
        AccountNumber number,
        Money initialBalance,
        DateTimeOffset openedAt)
    {
        Id = id;
        Number = number;
        Balance = initialBalance;
        Status = AccountStatus.Active;
        OpenedAt = openedAt;
        
        _transactions.Add(new Transaction(
            TransactionId.New(),
            TransactionType.Opening,
            initialBalance,
            openedAt));
    }
    
    public static Result<BankAccount, string> Open(
        AccountId id,
        AccountNumber number,
        Money initialBalance)
    {
        if (initialBalance.Amount < 0)
            return Result<BankAccount, string>.Failure(
                "Initial balance cannot be negative");
        
        return Result<BankAccount, string>.Success(
            new BankAccount(id, number, initialBalance, DateTimeOffset.UtcNow));
    }
    
    // Business logic encapsulated in entity
    public Result<Unit, string> Withdraw(Money amount)
    {
        if (Status != AccountStatus.Active)
            return Result<Unit, string>.Failure("Account is not active");
        
        if (amount.Amount <= 0)
            return Result<Unit, string>.Failure("Amount must be positive");
        
        if (Balance.Amount < amount.Amount)
            return Result<Unit, string>.Failure(
                $"Insufficient funds. Balance: {Balance.Amount}, Requested: {amount.Amount}");
        
        Balance = Money.FromAmount(Balance.Amount - amount.Amount, Balance.Currency);
        
        _transactions.Add(new Transaction(
            TransactionId.New(),
            TransactionType.Withdrawal,
            amount,
            DateTimeOffset.UtcNow));
        
        return Result<Unit, string>.Success(Unit.Value);
    }
    
    public Result<Unit, string> Deposit(Money amount)
    {
        if (Status != AccountStatus.Active)
            return Result<Unit, string>.Failure("Account is not active");
        
        if (amount.Amount <= 0)
            return Result<Unit, string>.Failure("Amount must be positive");
        
        Balance = Money.FromAmount(Balance.Amount + amount.Amount, Balance.Currency);
        
        _transactions.Add(new Transaction(
            TransactionId.New(),
            TransactionType.Deposit,
            amount,
            DateTimeOffset.UtcNow));
        
        return Result<Unit, string>.Success(Unit.Value);
    }
    
    public Result<Unit, string> Close()
    {
        if (Status == AccountStatus.Closed)
            return Result<Unit, string>.Failure("Account already closed");
        
        if (Balance.Amount != 0)
            return Result<Unit, string>.Failure(
                "Cannot close account with non-zero balance");
        
        Status = AccountStatus.Closed;
        
        _transactions.Add(new Transaction(
            TransactionId.New(),
            TransactionType.Closing,
            Money.Zero(Balance.Currency),
            DateTimeOffset.UtcNow));
        
        return Result<Unit, string>.Success(Unit.Value);
    }
    
    public Result<Unit, string> Suspend(string reason)
    {
        if (Status != AccountStatus.Active)
            return Result<Unit, string>.Failure("Only active accounts can be suspended");
        
        if (string.IsNullOrWhiteSpace(reason))
            return Result<Unit, string>.Failure("Suspension reason required");
        
        Status = AccountStatus.Suspended;
        return Result<Unit, string>.Success(Unit.Value);
    }
    
    public Result<Unit, string> Activate()
    {
        if (Status != AccountStatus.Suspended)
            return Result<Unit, string>.Failure("Only suspended accounts can be activated");
        
        Status = AccountStatus.Active;
        return Result<Unit, string>.Success(Unit.Value);
    }
    
    // Query methods
    public Money CalculateInterest(decimal annualRate, int days)
    {
        var dailyRate = annualRate / 365;
        var interestAmount = Balance.Amount * dailyRate * days;
        return Money.FromAmount(interestAmount, Balance.Currency);
    }
    
    public IReadOnlyList<Transaction> GetRecentTransactions(int count) =>
        _transactions.OrderByDescending(t => t.Timestamp).Take(count).ToList();
}

public sealed record Transaction(
    TransactionId Id,
    TransactionType Type,
    Money Amount,
    DateTimeOffset Timestamp);

public enum TransactionType
{
    Opening,
    Deposit,
    Withdrawal,
    Closing
}

public enum AccountStatus
{
    Active,
    Suspended,
    Closed
}

// Service is now thin—just orchestrates
public class BankAccountService
{
    private readonly IBankAccountRepository _repo;
    
    public async Task<Result<Unit, string>> Withdraw(
        AccountId accountId,
        Money amount)
    {
        var accountOpt = await _repo.GetByIdAsync(accountId);
        
        if (accountOpt.IsNone)
            return Result<Unit, string>.Failure("Account not found");
        
        var account = accountOpt.Value;
        
        // Business logic is in the entity
        var result = account.Withdraw(amount);
        
        if (result.IsSuccess)
            await _repo.SaveAsync(account);
        
        return result;
    }
}
```

## Rich Model Examples

### Example 1: Shopping Cart

```csharp
// ❌ Anemic: Cart is just a bag of items
public class ShoppingCart
{
    public List<CartItem> Items { get; set; }
    public decimal Total { get; set; }
}

public class ShoppingCartService
{
    public void AddItem(ShoppingCart cart, string productId, int quantity)
    {
        var item = cart.Items.FirstOrDefault(i => i.ProductId == productId);
        if (item != null)
        {
            item.Quantity += quantity;
        }
        else
        {
            cart.Items.Add(new CartItem { ProductId = productId, Quantity = quantity });
        }
        
        cart.Total = cart.Items.Sum(i => i.Price * i.Quantity);
    }
}

// ✅ Rich: Cart encapsulates behavior
public sealed class ShoppingCart
{
    private readonly List<CartItem> _items = new();
    
    public CartId Id { get; }
    public CustomerId CustomerId { get; }
    public IReadOnlyList<CartItem> Items => _items.AsReadOnly();
    public Money Total => CalculateTotal();
    public int ItemCount => _items.Sum(i => i.Quantity);
    
    private ShoppingCart(CartId id, CustomerId customerId)
    {
        Id = id;
        CustomerId = customerId;
    }
    
    public static ShoppingCart Create(CartId id, CustomerId customerId) =>
        new ShoppingCart(id, customerId);
    
    public Result<Unit, string> AddItem(ProductId productId, int quantity, Money unitPrice)
    {
        if (quantity <= 0)
            return Result<Unit, string>.Failure("Quantity must be positive");
        
        if (unitPrice.Amount <= 0)
            return Result<Unit, string>.Failure("Price must be positive");
        
        var existing = _items.FirstOrDefault(i => i.ProductId == productId);
        
        if (existing != null)
        {
            var newQuantity = existing.Quantity + quantity;
            if (newQuantity > 100)
                return Result<Unit, string>.Failure("Maximum quantity per item is 100");
            
            _items.Remove(existing);
            _items.Add(existing with { Quantity = newQuantity });
        }
        else
        {
            if (_items.Count >= 50)
                return Result<Unit, string>.Failure("Maximum 50 different items in cart");
            
            _items.Add(new CartItem(productId, quantity, unitPrice));
        }
        
        return Result<Unit, string>.Success(Unit.Value);
    }
    
    public Result<Unit, string> RemoveItem(ProductId productId)
    {
        var item = _items.FirstOrDefault(i => i.ProductId == productId);
        if (item == null)
            return Result<Unit, string>.Failure("Item not in cart");
        
        _items.Remove(item);
        return Result<Unit, string>.Success(Unit.Value);
    }
    
    public Result<Unit, string> UpdateQuantity(ProductId productId, int newQuantity)
    {
        if (newQuantity <= 0)
            return Result<Unit, string>.Failure("Quantity must be positive");
        
        if (newQuantity > 100)
            return Result<Unit, string>.Failure("Maximum quantity is 100");
        
        var item = _items.FirstOrDefault(i => i.ProductId == productId);
        if (item == null)
            return Result<Unit, string>.Failure("Item not in cart");
        
        _items.Remove(item);
        _items.Add(item with { Quantity = newQuantity });
        
        return Result<Unit, string>.Success(Unit.Value);
    }
    
    public void Clear() => _items.Clear();
    
    public Result<CartCheckout, string> PrepareCheckout()
    {
        if (_items.Count == 0)
            return Result<CartCheckout, string>.Failure("Cannot checkout empty cart");
        
        return Result<CartCheckout, string>.Success(
            new CartCheckout(Id, CustomerId, Items.ToList(), Total));
    }
    
    private Money CalculateTotal() => _items
        .Select(i => i.SubTotal)
        .Aggregate(Money.Zero, (sum, price) => sum + price);
}

public sealed record CartItem(ProductId ProductId, int Quantity, Money UnitPrice)
{
    public Money SubTotal => UnitPrice * Quantity;
}
```

### Example 2: Reservation System

```csharp
// ✅ Rich: Reservation encapsulates business rules
public sealed class Reservation
{
    public ReservationId Id { get; }
    public CustomerId CustomerId { get; }
    public ResourceId ResourceId { get; }
    public TimeSlot TimeSlot { get; }
    public ReservationStatus Status { get; private set; }
    public Option<string> CancellationReason { get; private set; }
    
    private Reservation(
        ReservationId id,
        CustomerId customerId,
        ResourceId resourceId,
        TimeSlot timeSlot)
    {
        Id = id;
        CustomerId = customerId;
        ResourceId = resourceId;
        TimeSlot = timeSlot;
        Status = ReservationStatus.Pending;
        CancellationReason = Option<string>.None();
    }
    
    public static Result<Reservation, string> Create(
        ReservationId id,
        CustomerId customerId,
        ResourceId resourceId,
        TimeSlot timeSlot)
    {
        if (timeSlot.Start < DateTimeOffset.UtcNow.AddHours(1))
            return Result<Reservation, string>.Failure(
                "Reservations must be at least 1 hour in advance");
        
        if (timeSlot.Duration < TimeSpan.FromMinutes(30))
            return Result<Reservation, string>.Failure(
                "Minimum reservation duration is 30 minutes");
        
        if (timeSlot.Duration > TimeSpan.FromHours(8))
            return Result<Reservation, string>.Failure(
                "Maximum reservation duration is 8 hours");
        
        return Result<Reservation, string>.Success(
            new Reservation(id, customerId, resourceId, timeSlot));
    }
    
    public Result<Unit, string> Confirm()
    {
        if (Status != ReservationStatus.Pending)
            return Result<Unit, string>.Failure("Only pending reservations can be confirmed");
        
        Status = ReservationStatus.Confirmed;
        return Result<Unit, string>.Success(Unit.Value);
    }
    
    public Result<Unit, string> Cancel(string reason)
    {
        if (Status == ReservationStatus.Cancelled)
            return Result<Unit, string>.Failure("Reservation already cancelled");
        
        if (Status == ReservationStatus.Completed)
            return Result<Unit, string>.Failure("Cannot cancel completed reservation");
        
        if (string.IsNullOrWhiteSpace(reason))
            return Result<Unit, string>.Failure("Cancellation reason required");
        
        var now = DateTimeOffset.UtcNow;
        var hoursUntilStart = (TimeSlot.Start - now).TotalHours;
        
        if (hoursUntilStart < 24 && Status == ReservationStatus.Confirmed)
            return Result<Unit, string>.Failure(
                "Cannot cancel confirmed reservation less than 24 hours before start");
        
        Status = ReservationStatus.Cancelled;
        CancellationReason = Option<string>.Some(reason);
        
        return Result<Unit, string>.Success(Unit.Value);
    }
    
    public Result<Unit, string> MarkAsCompleted()
    {
        if (Status != ReservationStatus.Confirmed)
            return Result<Unit, string>.Failure("Only confirmed reservations can be completed");
        
        if (TimeSlot.End > DateTimeOffset.UtcNow)
            return Result<Unit, string>.Failure("Reservation time slot not yet ended");
        
        Status = ReservationStatus.Completed;
        return Result<Unit, string>.Success(Unit.Value);
    }
    
    public bool IsActive() =>
        Status == ReservationStatus.Confirmed &&
        TimeSlot.Start <= DateTimeOffset.UtcNow &&
        TimeSlot.End > DateTimeOffset.UtcNow;
    
    public bool OverlapsWith(TimeSlot other) =>
        TimeSlot.OverlapsWith(other);
}

public sealed record TimeSlot(DateTimeOffset Start, DateTimeOffset End)
{
    public TimeSpan Duration => End - Start;
    
    public bool OverlapsWith(TimeSlot other) =>
        Start < other.End && other.Start < End;
}

public enum ReservationStatus
{
    Pending,
    Confirmed,
    Cancelled,
    Completed
}
```

## Why It's a Problem

1. **Scattered logic**: Business rules in services, not entities
2. **Poor encapsulation**: No protection of invariants
3. **Duplication**: Same rules repeated in multiple services
4. **Discoverability**: Hard to find what operations are valid
5. **Testing**: Must test logic in services, not domain

## Symptoms

- Entities with only getters/setters
- All business logic in service classes
- Services named `*Manager`, `*Service` doing domain work
- Public setters on domain entities
- Validation scattered across multiple services

## Benefits

- **Encapsulation**: Business rules protected by entity
- **Cohesion**: Data and behavior together
- **Discoverability**: Methods reveal valid operations
- **Testability**: Test business logic on entity directly
- **Maintainability**: Rules in one place

## See Also

- [Aggregate Roots](./aggregate-roots.md) — enforcing invariants
- [Domain Invariants](./domain-invariants.md) — business rules at construction
- [Smart Constructors](./smart-constructors.md) — parse, don't validate
- [Value Semantics](./value-semantics.md) — immutable value objects
- [Tell, Don't Ask](./tell-dont-ask.md) — command, don't query
