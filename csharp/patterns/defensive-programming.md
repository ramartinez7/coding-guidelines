# Defensive Programming

> Write code that anticipates misuse, validates inputs, and fails gracefully to prevent bugs from reaching production.

## Problem

Code that assumes perfect inputs and ideal conditions leads to runtime crashes, data corruption, and security vulnerabilities.

## Example

### ❌ Before

```csharp
public class OrderService
{
    public void ProcessOrder(Order order)
    {
        // Assumes order is not null
        var total = order.Items.Sum(i => i.Price * i.Quantity);
        order.Total = total;
        _repository.Save(order);
    }

    public User GetUser(int index)
    {
        // Assumes index is valid
        return _users[index];
    }

    public void UpdateEmail(UserId userId, string email)
    {
        // Assumes email is valid
        var user = _repository.Find(userId);
        user.Email = email;
        _repository.Update(user);
    }
}
```

### ✅ After

```csharp
public class OrderService
{
    public Result<Unit, ProcessOrderError> ProcessOrder(Order order)
    {
        ArgumentNullException.ThrowIfNull(order);

        if (order.Items.Count == 0)
        {
            return Result.Failure<Unit, ProcessOrderError>(
                new EmptyOrderError());
        }

        var total = order.Items.Sum(i => i.Price * i.Quantity);
        order.Total = total;

        try
        {
            _repository.Save(order);
            return Result.Success<Unit, ProcessOrderError>(Unit.Value);
        }
        catch (DatabaseException ex)
        {
            _logger.LogError(ex, "Failed to save order {OrderId}", order.Id);
            return Result.Failure<Unit, ProcessOrderError>(
                new DatabaseError(ex.Message));
        }
    }

    public Result<User, GetUserError> GetUser(int index)
    {
        if (index < 0 || index >= _users.Count)
        {
            return Result.Failure<User, GetUserError>(
                new IndexOutOfRangeError(index));
        }

        return Result.Success<User, GetUserError>(_users[index]);
    }

    public Result<Unit, UpdateEmailError> UpdateEmail(
        UserId userId, 
        EmailAddress email)
    {
        var userResult = _repository.Find(userId);
        if (userResult == null)
        {
            return Result.Failure<Unit, UpdateEmailError>(
                new UserNotFoundError(userId));
        }

        userResult.Email = email;

        try
        {
            _repository.Update(userResult);
            return Result.Success<Unit, UpdateEmailError>(Unit.Value);
        }
        catch (DatabaseException ex)
        {
            _logger.LogError(ex, "Failed to update email for user {UserId}", userId);
            return Result.Failure<Unit, UpdateEmailError>(
                new DatabaseError(ex.Message));
        }
    }
}
```

## Defensive Programming Techniques

### 1. Validate All Inputs

```csharp
// ❌ No validation
public void SetDiscount(decimal discount)
{
    this.discount = discount;
}

// ✅ Validate input ranges
public Result<Unit, ValidationError> SetDiscount(decimal discount)
{
    if (discount < 0 || discount > 1)
    {
        return Result.Failure<Unit, ValidationError>(
            new ValidationError("Discount must be between 0 and 1"));
    }

    this.discount = discount;
    return Result.Success<Unit, ValidationError>(Unit.Value);
}
```

### 2. Check for Null

```csharp
// ❌ Assumes not null
public void ProcessCustomer(Customer customer)
{
    var name = customer.Name.ToUpper();  // NullReferenceException!
}

// ✅ Check for null explicitly
public void ProcessCustomer(Customer customer)
{
    ArgumentNullException.ThrowIfNull(customer);
    ArgumentNullException.ThrowIfNull(customer.Name);

    var name = customer.Name.ToUpper();
}

// ✅ Use nullable reference types
public void ProcessCustomer(Customer customer)
{
    ArgumentNullException.ThrowIfNull(customer);

    // compiler enforces null checking if Name is string?
    if (customer.Name != null)
    {
        var name = customer.Name.ToUpper();
    }
}
```

### 3. Guard Against Invalid State

```csharp
// ❌ No state validation
public void Ship(Order order)
{
    order.Status = OrderStatus.Shipped;
    _shippingService.Ship(order);
}

// ✅ Validate state transitions
public Result<Unit, ShipOrderError> Ship(Order order)
{
    if (order.Status != OrderStatus.Paid)
    {
        return Result.Failure<Unit, ShipOrderError>(
            new InvalidOrderStatusError(order.Status, OrderStatus.Paid));
    }

    order.Status = OrderStatus.Shipped;
    _shippingService.Ship(order);
    return Result.Success<Unit, ShipOrderError>(Unit.Value);
}
```

### 4. Use Immutable Types

```csharp
// ❌ Mutable, can be changed unexpectedly
public class Money
{
    public decimal Amount { get; set; }
    public string Currency { get; set; }
}

// ✅ Immutable, prevents accidental modification
public sealed record Money(decimal Amount, string Currency)
{
    public Money Add(Money other)
    {
        if (Currency != other.Currency)
        {
            throw new InvalidOperationException(
                "Cannot add money with different currencies");
        }

        return new Money(Amount + other.Amount, Currency);
    }
}
```

### 5. Copy Defensive Copies

```csharp
// ❌ Exposes internal collection
public class Order
{
    private List<OrderItem> items = new();

    public List<OrderItem> Items => items;  // Caller can modify!
}

// Usage
var order = new Order();
order.Items.Add(new OrderItem());  // Modifies internal state!

// ✅ Return defensive copy or read-only view
public class Order
{
    private readonly List<OrderItem> items = new();

    public IReadOnlyList<OrderItem> Items => items.AsReadOnly();
}

// ✅ Or use immutable collection
public class Order
{
    private ImmutableList<OrderItem> items = ImmutableList<OrderItem>.Empty;

    public ImmutableList<OrderItem> Items => items;
}
```

### 6. Validate at Boundaries

```csharp
// ❌ Validates deep in the call stack
public class OrderService
{
    public void ProcessOrder(OrderDto dto)
    {
        var order = MapToOrder(dto);
        ValidateOrder(order);  // Too late!
        _repository.Save(order);
    }
}

// ✅ Validate at system boundary
public class OrderController
{
    public IActionResult CreateOrder([FromBody] OrderDto dto)
    {
        var validationResult = ValidateOrderDto(dto);
        if (validationResult.IsFailure)
        {
            return BadRequest(validationResult.Error);
        }

        var order = MapToOrder(dto);
        _orderService.ProcessOrder(order);
        return Ok();
    }
}
```

### 7. Use Assertions for Invariants

```csharp
// ✅ Assert invariants in debug builds
public void ProcessPayment(Order order)
{
    Debug.Assert(order != null, "Order cannot be null");
    Debug.Assert(order.Total > 0, "Order total must be positive");
    Debug.Assert(order.Status == OrderStatus.Pending, 
        "Order must be in pending status");

    // Process payment
}
```

### 8. Fail Fast

```csharp
// ❌ Silent failure leads to corruption
public void UpdateInventory(ProductId productId, int quantity)
{
    var product = _repository.Find(productId);
    if (product != null)
    {
        product.Quantity += quantity;
        _repository.Update(product);
    }
    // Silently does nothing if product not found!
}

// ✅ Fail fast with clear error
public Result<Unit, InventoryError> UpdateInventory(
    ProductId productId, 
    int quantity)
{
    var product = _repository.Find(productId);
    if (product == null)
    {
        return Result.Failure<Unit, InventoryError>(
            new ProductNotFoundError(productId));
    }

    product.Quantity += quantity;
    _repository.Update(product);
    return Result.Success<Unit, InventoryError>(Unit.Value);
}
```

### 9. Validate Return Values

```csharp
// ❌ Assumes external service always succeeds
public User GetUser(UserId userId)
{
    var user = _externalService.GetUser(userId);
    return user;  // Could be null!
}

// ✅ Validate external results
public Result<User, GetUserError> GetUser(UserId userId)
{
    var user = _externalService.GetUser(userId);
    if (user == null)
    {
        return Result.Failure<User, GetUserError>(
            new UserNotFoundError(userId));
    }

    return Result.Success<User, GetUserError>(user);
}
```

### 10. Use Try-Parse Pattern

```csharp
// ❌ Throws exception on invalid input
public int ParseQuantity(string input)
{
    return int.Parse(input);  // Throws FormatException
}

// ✅ Use TryParse for graceful handling
public Option<int> ParseQuantity(string input)
{
    return int.TryParse(input, out var value) && value > 0
        ? Option.Some(value)
        : Option.None<int>();
}
```

### 11. Handle Resource Cleanup

```csharp
// ❌ Resource leak if exception occurs
public void ProcessFile(string path)
{
    var stream = File.OpenRead(path);
    ProcessStream(stream);
    stream.Dispose();  // May not execute!
}

// ✅ Ensure cleanup with using
public void ProcessFile(string path)
{
    using var stream = File.OpenRead(path);
    ProcessStream(stream);
}  // Guaranteed disposal
```

### 12. Validate String Inputs

```csharp
// ❌ No string validation
public void SetProductName(string name)
{
    this.name = name;  // Could be null, empty, or whitespace!
}

// ✅ Validate strings thoroughly
public Result<Unit, ValidationError> SetProductName(string name)
{
    if (string.IsNullOrWhiteSpace(name))
    {
        return Result.Failure<Unit, ValidationError>(
            new ValidationError("Product name cannot be empty"));
    }

    if (name.Length > 100)
    {
        return Result.Failure<Unit, ValidationError>(
            new ValidationError("Product name too long (max 100 characters)"));
    }

    this.name = name;
    return Result.Success<Unit, ValidationError>(Unit.Value);
}
```

## Symptoms

- NullReferenceException in production
- IndexOutOfRangeException from invalid indices
- Data corruption from invalid state
- Security vulnerabilities from unvalidated input
- Silent failures that mask bugs

## Benefits

- **Fewer runtime errors** by catching issues early
- **Clearer failure modes** with explicit error handling
- **Better security** by validating all inputs
- **Easier debugging** with fail-fast behavior
- **More maintainable code** with clear invariants

## See Also

- [Smart Constructors](./smart-constructors.md) — Enforcing invariants at construction
- [Domain Invariants](./domain-invariants.md) — Business rules at construction
- [Honest Functions](./honest-functions.md) — Explicit error cases in signatures
- [Nullable Reference Types](./nullable-reference-types.md) — Compile-time null safety
