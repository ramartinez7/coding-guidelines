# Domain Invariants (Enforcing Business Rules at Construction)

> Business rules validated at multiple points allow invalid objects to exist—enforce invariants in constructors to make invalid states unrepresentable.

## Problem

When validation is scattered throughout the codebase, objects can be created or modified into invalid states. Clients must remember to validate before every operation, leading to defensive checks everywhere and no guarantee of correctness.

## Example

### ❌ Before

```csharp
public class BankAccount
{
    public decimal Balance { get; set; }
    public AccountStatus Status { get; set; }
    public decimal OverdraftLimit { get; set; }
}

// Validation scattered throughout codebase
public class TransferService
{
    public void Transfer(BankAccount from, BankAccount to, decimal amount)
    {
        // Validation checks repeated everywhere
        if (amount <= 0)
            throw new InvalidOperationException("Amount must be positive");
        
        if (from.Status != AccountStatus.Active)
            throw new InvalidOperationException("Account not active");
        
        if (from.Balance - amount < -from.OverdraftLimit)
            throw new InvalidOperationException("Insufficient funds");
        
        // Can still create invalid states
        from.Balance -= amount;
        to.Balance += amount;
    }
}

// Easy to bypass validation
var account = new BankAccount 
{ 
    Balance = -1000000,  // Invalid! No overdraft limit check
    Status = (AccountStatus)999,  // Invalid enum value!
    OverdraftLimit = -500  // Negative limit!
};
```

**Problems:**
- Public setters allow bypassing invariants
- Validation duplicated across methods
- Invalid states are representable
- No single source of truth for business rules
- Easy to forget validation checks

### ✅ After

```csharp
/// <summary>
/// Bank account with invariants enforced at construction.
/// Once created, the object is guaranteed to be in a valid state.
/// </summary>
public sealed class BankAccount
{
    public AccountId Id { get; }
    public Money Balance { get; private set; }
    public AccountStatus Status { get; private set; }
    public Money OverdraftLimit { get; }
    
    // Private constructor—can only create through factory method
    private BankAccount(
        AccountId id,
        Money balance,
        AccountStatus status,
        Money overdraftLimit)
    {
        Id = id;
        Balance = balance;
        Status = status;
        OverdraftLimit = overdraftLimit;
    }
    
    /// <summary>
    /// Static factory method enforces invariants at creation.
    /// </summary>
    public static Result<BankAccount, string> Create(
        AccountId id,
        Money initialDeposit,
        Money overdraftLimit)
    {
        // Invariant: Initial deposit must be non-negative
        if (initialDeposit.Amount < 0)
            return Result<BankAccount, string>.Failure(
                "Initial deposit cannot be negative");
        
        // Invariant: Overdraft limit must be non-negative
        if (overdraftLimit.Amount < 0)
            return Result<BankAccount, string>.Failure(
                "Overdraft limit cannot be negative");
        
        // Invariant: New accounts start as Active
        var account = new BankAccount(
            id,
            initialDeposit,
            AccountStatus.Active,
            overdraftLimit);
        
        return Result<BankAccount, string>.Success(account);
    }
    
    /// <summary>
    /// Withdraw enforces invariant: balance cannot go below overdraft limit.
    /// </summary>
    public Result<Unit, string> Withdraw(Money amount)
    {
        // Invariant: Amount must be positive
        if (amount.Amount <= 0)
            return Result<Unit, string>.Failure("Amount must be positive");
        
        // Invariant: Account must be active
        if (Status != AccountStatus.Active)
            return Result<Unit, string>.Failure("Account is not active");
        
        // Invariant: Balance cannot go below overdraft limit
        var newBalance = Balance - amount;
        if (newBalance.Amount < -OverdraftLimit.Amount)
            return Result<Unit, string>.Failure(
                $"Insufficient funds. Available: {Balance + OverdraftLimit}");
        
        Balance = newBalance;
        return Result<Unit, string>.Success(Unit.Value);
    }
    
    /// <summary>
    /// Deposit enforces invariant: amount must be positive.
    /// </summary>
    public Result<Unit, string> Deposit(Money amount)
    {
        // Invariant: Amount must be positive
        if (amount.Amount <= 0)
            return Result<Unit, string>.Failure("Amount must be positive");
        
        // Invariant: Account must be active
        if (Status != AccountStatus.Active)
            return Result<Unit, string>.Failure("Account is not active");
        
        Balance = Balance + amount;
        return Result<Unit, string>.Success(Unit.Value);
    }
    
    /// <summary>
    /// Close account enforces state transition rules.
    /// </summary>
    public Result<Unit, string> Close()
    {
        // Invariant: Can only close active accounts
        if (Status != AccountStatus.Active)
            return Result<Unit, string>.Failure("Account already closed");
        
        // Invariant: Must have zero balance to close
        if (Balance.Amount != 0)
            return Result<Unit, string>.Failure(
                $"Cannot close account with non-zero balance: {Balance}");
        
        Status = AccountStatus.Closed;
        return Result<Unit, string>.Success(Unit.Value);
    }
}

// Transfer service now trusts the invariants
public sealed class TransferService
{
    public async Task<Result<Unit, string>> Transfer(
        BankAccount from,
        BankAccount to,
        Money amount)
    {
        // No validation needed—accounts enforce their own invariants
        var withdrawResult = from.Withdraw(amount);
        if (!withdrawResult.IsSuccess)
            return withdrawResult;
        
        var depositResult = to.Deposit(amount);
        if (!depositResult.IsSuccess)
        {
            // Compensate: return the money
            from.Deposit(amount);
            return depositResult;
        }
        
        return Result<Unit, string>.Success(Unit.Value);
    }
}
```

## Types of Invariants

### 1. Single-Field Invariants

```csharp
// Invariant: Email must be valid format
public sealed record EmailAddress
{
    private const string EmailPattern = @"^[^@\s]+@[^@\s]+\.[^@\s]+$";
    
    public string Value { get; }
    
    private EmailAddress(string value) => Value = value;
    
    public static Result<EmailAddress, string> Create(string value)
    {
        if (string.IsNullOrWhiteSpace(value))
            return Result<EmailAddress, string>.Failure("Email cannot be empty");
        
        if (!Regex.IsMatch(value, EmailPattern))
            return Result<EmailAddress, string>.Failure("Email format invalid");
        
        return Result<EmailAddress, string>.Success(new EmailAddress(value));
    }
}

// Invariant: Age must be between 0 and 150
public readonly record struct Age
{
    public int Value { get; }
    
    private Age(int value) => Value = value;
    
    public static Result<Age, string> Create(int value)
    {
        if (value < 0)
            return Result<Age, string>.Failure("Age cannot be negative");
        
        if (value > 150)
            return Result<Age, string>.Failure("Age cannot exceed 150");
        
        return Result<Age, string>.Success(new Age(value));
    }
}
```

### 2. Multi-Field Invariants

```csharp
// Invariant: End date must be after start date
public sealed record DateRange
{
    public DateOnly Start { get; }
    public DateOnly End { get; }
    
    private DateRange(DateOnly start, DateOnly end)
    {
        Start = start;
        End = end;
    }
    
    public static Result<DateRange, string> Create(DateOnly start, DateOnly end)
    {
        if (end < start)
            return Result<DateRange, string>.Failure(
                "End date must be on or after start date");
        
        return Result<DateRange, string>.Success(new DateRange(start, end));
    }
    
    public int Days => End.DayNumber - Start.DayNumber + 1;
    public bool Contains(DateOnly date) => date >= Start && date <= End;
}

// Invariant: Price must be positive, quantity must be positive, total must match
public sealed record OrderLine
{
    public ProductId ProductId { get; }
    public int Quantity { get; }
    public Money UnitPrice { get; }
    public Money LineTotal { get; }
    
    private OrderLine(ProductId productId, int quantity, Money unitPrice, Money lineTotal)
    {
        ProductId = productId;
        Quantity = quantity;
        UnitPrice = unitPrice;
        LineTotal = lineTotal;
    }
    
    public static Result<OrderLine, string> Create(
        ProductId productId,
        int quantity,
        Money unitPrice)
    {
        if (quantity <= 0)
            return Result<OrderLine, string>.Failure("Quantity must be positive");
        
        if (unitPrice.Amount <= 0)
            return Result<OrderLine, string>.Failure("Unit price must be positive");
        
        var lineTotal = new Money(unitPrice.Amount * quantity, unitPrice.Currency);
        
        return Result<OrderLine, string>.Success(
            new OrderLine(productId, quantity, unitPrice, lineTotal));
    }
}
```

### 3. Collection Invariants

```csharp
// Invariant: Order must have at least one item
public sealed class Order
{
    private readonly List<OrderLine> _lines = new();
    
    public OrderId Id { get; }
    public CustomerId CustomerId { get; }
    public IReadOnlyList<OrderLine> Lines => _lines.AsReadOnly();
    
    private Order(OrderId id, CustomerId customerId, IEnumerable<OrderLine> lines)
    {
        Id = id;
        CustomerId = customerId;
        _lines.AddRange(lines);
    }
    
    public static Result<Order, string> Create(
        OrderId id,
        CustomerId customerId,
        IEnumerable<OrderLine> lines)
    {
        var lineList = lines.ToList();
        
        // Invariant: Must have at least one line
        if (lineList.Count == 0)
            return Result<Order, string>.Failure("Order must have at least one item");
        
        // Invariant: All lines must be for different products
        var uniqueProducts = lineList.Select(l => l.ProductId).Distinct().Count();
        if (uniqueProducts != lineList.Count)
            return Result<Order, string>.Failure("Duplicate products in order");
        
        return Result<Order, string>.Success(new Order(id, customerId, lineList));
    }
    
    public Money Total
    {
        get
        {
            if (_lines.Count == 0)
                return new Money(0, Currency.USD);  // Default currency
            
            return _lines
                .Select(line => line.LineTotal)
                .Aggregate((sum, total) => sum + total);
        }
    }
}

// Invariant: Non-empty list
public sealed record NonEmptyList<T>
{
    private readonly List<T> _items;
    
    public T First { get; }
    public IReadOnlyList<T> Rest { get; }
    public IReadOnlyList<T> All => _items.AsReadOnly();
    public int Count => _items.Count;
    
    private NonEmptyList(T first, IEnumerable<T> rest)
    {
        First = first;
        var restList = rest.ToList();
        Rest = restList.AsReadOnly();
        _items = new List<T>(restList.Count + 1) { first };
        _items.AddRange(restList);
    }
    
    public static NonEmptyList<T> Create(T first, params T[] rest)
    {
        return new NonEmptyList<T>(first, rest);
    }
    
    public static Result<NonEmptyList<T>, string> FromList(IEnumerable<T> items)
    {
        var list = items.ToList();
        if (list.Count == 0)
            return Result<NonEmptyList<T>, string>.Failure("List cannot be empty");
        
        return Result<NonEmptyList<T>, string>.Success(
            new NonEmptyList<T>(list[0], list.Skip(1)));
    }
}
```

### 4. State-Dependent Invariants

```csharp
// Invariant: Shipped orders must have tracking number and ship date
public abstract record OrderState;

public sealed record DraftOrder(
    OrderId Id,
    CustomerId CustomerId,
    List<OrderLine> Lines) : OrderState
{
    public Result<SubmittedOrder, string> Submit()
    {
        if (Lines.Count == 0)
            return Result<SubmittedOrder, string>.Failure(
                "Cannot submit empty order");
        
        return Result<SubmittedOrder, string>.Success(
            new SubmittedOrder(Id, CustomerId, Lines.AsReadOnly()));
    }
}

public sealed record SubmittedOrder(
    OrderId Id,
    CustomerId CustomerId,
    IReadOnlyList<OrderLine> Lines) : OrderState
{
    public Result<ShippedOrder, string> Ship(
        TrackingNumber trackingNumber,
        DateTimeOffset shippedAt)
    {
        return Result<ShippedOrder, string>.Success(
            new ShippedOrder(Id, CustomerId, Lines, trackingNumber, shippedAt));
    }
}

public sealed record ShippedOrder(
    OrderId Id,
    CustomerId CustomerId,
    IReadOnlyList<OrderLine> Lines,
    TrackingNumber TrackingNumber,  // Always present when shipped
    DateTimeOffset ShippedAt) : OrderState;  // Always present when shipped
```

## Benefits

- **Compile-time guarantees**: Invalid states are unrepresentable
- **Single source of truth**: Business rules in one place
- **Self-documenting**: Constructor parameters reveal requirements
- **Fail fast**: Invalid data rejected at boundary
- **No defensive checks**: Internal code trusts invariants
- **Easier testing**: Test construction, not scattered validation
- **Refactoring safety**: Changes to invariants cause compile errors

## Invariant Enforcement Strategies

### 1. Private Constructor + Static Factory

```csharp
public sealed class Product
{
    public ProductId Id { get; }
    public string Name { get; }
    public Money Price { get; }
    
    private Product(ProductId id, string name, Money price)
    {
        Id = id;
        Name = name;
        Price = price;
    }
    
    public static Result<Product, string> Create(
        ProductId id,
        string name,
        Money price)
    {
        if (string.IsNullOrWhiteSpace(name))
            return Result<Product, string>.Failure("Name is required");
        
        if (price.Amount <= 0)
            return Result<Product, string>.Failure("Price must be positive");
        
        return Result<Product, string>.Success(new Product(id, name, price));
    }
}
```

### 2. Record with Validation in Constructor

```csharp
public sealed record PostalCode
{
    private static readonly Regex UsZipCodePattern = new(@"^\d{5}(-\d{4})?$");
    
    public string Value { get; }
    
    public PostalCode(string value)
    {
        if (string.IsNullOrWhiteSpace(value))
            throw new ArgumentException("Postal code cannot be empty");
        
        if (!UsZipCodePattern.IsMatch(value))
            throw new ArgumentException("Invalid US postal code format");
        
        Value = value;
    }
}

// Usage: exceptions at construction
try
{
    var zip = new PostalCode("12345");  // ✓ Valid
    var bad = new PostalCode("ABCDE");  // ✗ Throws
}
catch (ArgumentException ex)
{
    // Handle validation error
}
```

### 3. Required Properties with Init-Only Setters

```csharp
public sealed record Customer
{
    // Required properties enforce invariant: customer must have these
    public required CustomerId Id { get; init; }
    public required string Name { get; init; }
    public required EmailAddress Email { get; init; }
    
    // Cannot create without all required properties
}

// ✓ Valid
var customer = new Customer
{
    Id = CustomerId.New(),
    Name = "Alice",
    Email = EmailAddress.Parse("alice@example.com").Value
};

// ✗ Compile error: missing required properties
var invalid = new Customer { Name = "Bob" };
```

## Why It's a Problem

1. **Invalid states possible**: Objects can exist in states that violate business rules
2. **Scattered validation**: Same checks duplicated throughout codebase
3. **Defensive programming**: Every method must validate inputs
4. **Runtime errors**: Invalid data discovered late, in production
5. **Maintenance burden**: Changing rules requires updating multiple locations

## Symptoms

- Validation checks repeated in multiple methods
- Comments like "must not be null" or "must be positive"
- Public setters allowing arbitrary state changes
- Try-catch blocks catching invalid state exceptions
- Unit tests for validation logic scattered everywhere

## See Also

- [Static Factory Methods](./static-factory-methods.md) — creating instances with validation
- [Honest Functions](./honest-functions.md) — using Result types for validation
- [Value Semantics](./value-semantics.md) — immutable value objects with invariants
- [Aggregate Roots](./aggregate-roots.md) — enforcing aggregate-level invariants
- [Enforcing Call Order](./enforcing-call-order.md) — state transition invariants
