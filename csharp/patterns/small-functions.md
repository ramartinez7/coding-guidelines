# Small Functions

> Functions should do one thing, do it well, and be small enough to fit in your head.

## Problem

Large functions that do multiple things are hard to understand, test, and maintain. They violate the Single Responsibility Principle and make code reuse difficult.

## Example

### ❌ Before

```csharp
public void ProcessOrder(OrderId orderId)
{
    // Validate order exists
    var order = _repository.FindById(orderId);
    if (order == null)
    {
        _logger.LogError("Order {OrderId} not found", orderId);
        return;
    }

    // Check inventory
    foreach (var item in order.Items)
    {
        var inventory = _inventoryService.GetInventory(item.ProductId);
        if (inventory.Quantity < item.Quantity)
        {
            _logger.LogWarning("Insufficient inventory for {ProductId}", item.ProductId);
            order.Status = OrderStatus.OnHold;
            _repository.Update(order);
            _emailService.SendEmail(order.CustomerEmail, "Order on hold", "...");
            return;
        }
    }

    // Calculate total
    decimal subtotal = 0;
    foreach (var item in order.Items)
    {
        subtotal += item.Price * item.Quantity;
    }
    decimal tax = subtotal * 0.08m;
    decimal shipping = subtotal > 50 ? 0 : 5.99m;
    decimal total = subtotal + tax + shipping;

    // Process payment
    var paymentResult = _paymentGateway.Charge(order.PaymentMethod, total);
    if (!paymentResult.Success)
    {
        _logger.LogError("Payment failed for {OrderId}", orderId);
        order.Status = OrderStatus.PaymentFailed;
        _repository.Update(order);
        _emailService.SendEmail(order.CustomerEmail, "Payment failed", "...");
        return;
    }

    // Reserve inventory
    foreach (var item in order.Items)
    {
        _inventoryService.Reserve(item.ProductId, item.Quantity);
    }

    // Update order
    order.Status = OrderStatus.Confirmed;
    order.Total = total;
    _repository.Update(order);

    // Send confirmation
    _emailService.SendEmail(order.CustomerEmail, "Order confirmed", "...");
    _logger.LogInformation("Order {OrderId} processed successfully", orderId);
}
```

### ✅ After

```csharp
public Result<Unit, ProcessOrderError> ProcessOrder(OrderId orderId)
{
    return FindOrder(orderId)
        .Bind(ValidateInventory)
        .Bind(CalculateOrderTotal)
        .Bind(ProcessPayment)
        .Bind(ReserveInventory)
        .Bind(ConfirmOrder)
        .Tap(SendOrderConfirmation);
}

private Result<Order, ProcessOrderError> FindOrder(OrderId orderId)
{
    var order = _repository.FindById(orderId);
    return order != null
        ? Result.Success<Order, ProcessOrderError>(order)
        : Result.Failure<Order, ProcessOrderError>(new OrderNotFound(orderId));
}

private Result<Order, ProcessOrderError> ValidateInventory(Order order)
{
    foreach (var item in order.Items)
    {
        if (!HasSufficientInventory(item))
        {
            PlaceOrderOnHold(order);
            SendInventoryHoldEmail(order);
            return Result.Failure<Order, ProcessOrderError>(new InsufficientInventory(item.ProductId));
        }
    }
    return Result.Success<Order, ProcessOrderError>(order);
}

private bool HasSufficientInventory(OrderItem item)
{
    var inventory = _inventoryService.GetInventory(item.ProductId);
    return inventory.Quantity >= item.Quantity;
}

private Result<Order, ProcessOrderError> CalculateOrderTotal(Order order)
{
    var subtotal = CalculateSubtotal(order.Items);
    var tax = CalculateTax(subtotal);
    var shipping = CalculateShipping(subtotal);
    
    order.Total = subtotal + tax + shipping;
    return Result.Success<Order, ProcessOrderError>(order);
}

private decimal CalculateSubtotal(IEnumerable<OrderItem> items)
{
    return items.Sum(item => item.Price * item.Quantity);
}

private decimal CalculateTax(decimal subtotal)
{
    return subtotal * 0.08m;
}

private decimal CalculateShipping(decimal subtotal)
{
    return subtotal > 50 ? 0 : 5.99m;
}

private Result<Order, ProcessOrderError> ProcessPayment(Order order)
{
    var paymentResult = _paymentGateway.Charge(order.PaymentMethod, order.Total);
    
    if (!paymentResult.Success)
    {
        MarkPaymentFailed(order);
        SendPaymentFailedEmail(order);
        return Result.Failure<Order, ProcessOrderError>(new PaymentFailed(order.Id));
    }

    return Result.Success<Order, ProcessOrderError>(order);
}

private Result<Order, ProcessOrderError> ReserveInventory(Order order)
{
    foreach (var item in order.Items)
    {
        _inventoryService.Reserve(item.ProductId, item.Quantity);
    }
    return Result.Success<Order, ProcessOrderError>(order);
}

private Result<Order, ProcessOrderError> ConfirmOrder(Order order)
{
    order.Status = OrderStatus.Confirmed;
    _repository.Update(order);
    _logger.LogInformation("Order {OrderId} processed successfully", order.Id);
    return Result.Success<Order, ProcessOrderError>(order);
}

private void SendOrderConfirmation(Order order)
{
    _emailService.SendEmail(order.CustomerEmail, "Order confirmed", "...");
}
```

## Principles

### Single Responsibility

Each function should have **one reason to change**:

```csharp
// ❌ Multiple responsibilities
public void SaveUser(User user)
{
    ValidateUser(user);
    HashPassword(user);
    _database.Save(user);
    SendWelcomeEmail(user);
    LogUserCreation(user);
}

// ✅ Single responsibility per function
public Result<Unit, SaveUserError> SaveUser(ValidatedUser user)
{
    var hashedUser = HashPassword(user);
    _repository.Save(hashedUser);
    return Result.Success<Unit, SaveUserError>(Unit.Value);
}

public void OnUserCreated(User user)
{
    SendWelcomeEmail(user);
    LogUserCreation(user);
}
```

### One Level of Abstraction

All code in a function should be at the same level of abstraction:

```csharp
// ❌ Mixed levels of abstraction
public void ProcessReport()
{
    var data = _repository.GetData();  // High-level
    
    // Low-level details mixed in
    var formatted = new StringBuilder();
    foreach (var item in data)
    {
        formatted.AppendLine($"{item.Name}: {item.Value}");
    }
    
    SaveReport(formatted.ToString());  // High-level
}

// ✅ Consistent level of abstraction
public void ProcessReport()
{
    var data = _repository.GetData();
    var formatted = FormatReport(data);
    SaveReport(formatted);
}

private string FormatReport(IEnumerable<ReportItem> data)
{
    var formatted = new StringBuilder();
    foreach (var item in data)
    {
        formatted.AppendLine($"{item.Name}: {item.Value}");
    }
    return formatted.ToString();
}
```

### Descriptive Names

Long, descriptive function names are better than short, cryptic ones:

```csharp
// ❌ Unclear what this does
public void Process(Order order) { }
public void Handle(User user) { }

// ✅ Clear intent
public void CalculateOrderTotalWithTaxAndShipping(Order order) { }
public void ValidateUserEmailAndSendConfirmation(User user) { }

// ✅ Even better: split if doing multiple things
public Money CalculateOrderTotal(Order order) { }
public void ValidateUserEmail(Email email) { }
public void SendEmailConfirmation(User user) { }
```

### Minimize Parameters

Limit function parameters to 3 or fewer—use objects for related parameters:

```csharp
// ❌ Too many parameters
public void CreateInvoice(string customerName, string customerEmail, 
    string street, string city, string state, string zip,
    decimal subtotal, decimal tax, decimal total) { }

// ✅ Use parameter objects
public void CreateInvoice(Customer customer, Address address, OrderTotal total) { }
```

### Avoid Side Effects

Functions should not have hidden side effects:

```csharp
// ❌ Hidden side effect (modifies state)
public User GetUser(UserId id)
{
    var user = _repository.Find(id);
    _cache.Store(id, user);  // Hidden side effect!
    return user;
}

// ✅ Explicit about side effects
public User GetUser(UserId id)
{
    return _repository.Find(id);
}

public void CacheUser(UserId id, User user)
{
    _cache.Store(id, user);
}

// Or make the side effect obvious in the name
public User GetUserAndUpdateCache(UserId id)
{
    var user = _repository.Find(id);
    _cache.Store(id, user);
    return user;
}
```

### Prefer Pure Functions

When possible, write functions without side effects:

```csharp
// ❌ Impure (depends on and modifies state)
public class Calculator
{
    private decimal total = 0;
    
    public void Add(decimal value)
    {
        this.total += value;
    }
}

// ✅ Pure (no side effects)
public static class Calculator
{
    public static decimal Add(decimal a, decimal b)
    {
        return a + b;
    }
}
```

## Symptoms

- Functions longer than 20 lines
- Nested loops and conditionals more than 2-3 levels deep
- Difficulty naming the function (sign it does multiple things)
- Multiple `return` statements scattered throughout
- Comments needed to explain different sections of the function
- Hard to write unit tests (too many code paths)

## Benefits

- **Easier to understand** — each function has a single, clear purpose
- **Easier to test** — small functions have fewer code paths
- **Better reusability** — focused functions can be used in multiple contexts
- **Simplified debugging** — stack traces point to specific operations
- **Improved maintainability** — changes are localized to single-purpose functions

## See Also

- [Honest Functions](./honest-functions.md) — Functions that tell the truth about their outcomes
- [Flag Arguments](./flag-arguments.md) — Replacing boolean parameters with separate functions
- [Data Clumps](./data-clump.md) — Grouping related parameters into objects
