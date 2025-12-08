# Comments and Documentation

> Good code explains itself—comments should explain *why*, not *what*.

## Problem

Excessive or outdated comments clutter code and often lie. Missing comments for complex logic leave future maintainers guessing. The goal is self-documenting code with strategic comments where needed.

## Example

### ❌ Before

```csharp
public class OrderProcessor
{
    // Process order
    public void Process(int id)
    {
        // Get the order
        var order = _repo.GetOrder(id);
        
        // Check if order is null
        if (order == null)
        {
            // Log error
            _logger.LogError("Order not found");
            return;
        }
        
        // Calculate total
        var total = 0m;
        // Loop through items
        foreach (var item in order.Items)
        {
            // Add item price to total
            total += item.Price;
        }
        
        // Apply discount (10% off)
        total = total * 0.9m;  // TODO: make this configurable
        
        // Save order
        order.Total = total;
        _repo.Save(order);
    }
}
```

### ✅ After

```csharp
public class OrderProcessor
{
    public Result<Unit, ProcessOrderError> ProcessOrder(OrderId orderId)
    {
        return FindOrder(orderId)
            .Map(CalculateOrderTotal)
            .Map(ApplyLoyaltyDiscount)
            .Bind(SaveOrder);
    }

    private Money CalculateOrderTotal(Order order)
    {
        return order.Items
            .Select(item => item.Price)
            .Aggregate(Money.Zero, (acc, price) => acc.Add(price));
    }

    // Discount rate based on business rule: 10% for all customers until 2024 Q2
    // when tiered loyalty program launches. See: JIRA-1234
    private Money ApplyLoyaltyDiscount(Order order)
    {
        const decimal DiscountRate = 0.10m;
        return order.Total.Multiply(1m - DiscountRate);
    }
}
```

## When to Write Comments

### ✅ DO Comment: Why, Not What

Explain the reasoning behind non-obvious decisions:

```csharp
// ❌ Comments state the obvious
// Increment counter
counter++;

// Set name to lowercase
name = name.ToLower();

// ✅ Comments explain why
// Convert to lowercase for case-insensitive comparison.
// Business rule: treat "John" and "john" as the same customer.
name = name.ToLower();

// Use UTC to avoid timezone bugs when comparing order timestamps
// across multiple data centers. See incident report: INC-2023-0456.
var orderTimestamp = DateTime.UtcNow;
```

### ✅ DO Comment: Business Rules and Constraints

Document domain knowledge and regulatory requirements:

```csharp
// Payment must be processed within 24 hours of order placement
// per PCI-DSS requirement 3.4. Expired authorizations are automatically
// voided to minimize risk of funds mismatch.
if (order.CreatedAt.AddHours(24) < DateTime.UtcNow)
{
    VoidPaymentAuthorization(order.PaymentId);
}

// Tax rate varies by jurisdiction. California Prop 30 (2012) increased
// state tax by 0.25%. See: https://taxfoundation.org/california-prop-30
const decimal CaliforniaStateTaxRate = 0.0725m;
```

### ✅ DO Comment: Workarounds and TODOs

Explain temporary solutions and technical debt:

```csharp
// WORKAROUND: Third-party API returns 500 on empty results instead of 404.
// Treat 500 as "not found" until they fix their bug. Ticket: EXT-789
try
{
    return await _externalApi.GetData(id);
}
catch (HttpException ex) when (ex.StatusCode == 500)
{
    return Result.Failure<Data>(new DataNotFound(id));
}

// TODO(username): Refactor to use async/await when upgrading to .NET 8.
// Current blocking call acceptable for <10 RPS load. Monitor: PERF-123
var result = _service.GetDataSync(id);
```

### ✅ DO Comment: Complex Algorithms

Explain non-trivial logic or algorithms:

```csharp
// Calculate order priority using weighted scoring:
// - Age: older orders score higher (prevent starvation)
// - Value: high-value orders prioritized
// - Customer tier: VIP customers get boost
// Formula: (age_hours * 2) + (value / 100) + (tier_multiplier)
private int CalculateOrderPriority(Order order)
{
    var ageHours = (DateTime.UtcNow - order.CreatedAt).TotalHours;
    var valueScore = order.Total.Amount / 100m;
    var tierMultiplier = GetTierMultiplier(order.CustomerId);
    
    return (int)((ageHours * 2) + valueScore + tierMultiplier);
}
```

### ✅ DO Comment: Public APIs

Use XML documentation comments for public APIs:

```csharp
/// <summary>
/// Processes a customer refund and returns the transaction result.
/// </summary>
/// <param name="refundRequest">The validated refund request containing order ID and amount.</param>
/// <returns>
/// A result containing the refund transaction on success, or a specific error explaining
/// why the refund could not be processed (invalid order, already refunded, payment gateway error).
/// </returns>
/// <exception cref="ArgumentNullException">Thrown when refundRequest is null.</exception>
public async Task<Result<RefundTransaction, RefundError>> ProcessRefund(
    ValidatedRefundRequest refundRequest)
{
    // Implementation...
}
```

## When NOT to Write Comments

### ❌ DON'T Comment: Obvious Code

Let the code speak for itself:

```csharp
// ❌ Useless comments
// Create a new user
var user = new User();

// Set the name
user.Name = name;

// Add user to list
users.Add(user);

// ✅ No comments needed
var user = new User { Name = name };
users.Add(user);
```

### ❌ DON'T Comment: Commented-Out Code

Use version control instead:

```csharp
// ❌ Dead code creates confusion
public void ProcessOrder(Order order)
{
    // var tax = order.Total * 0.08m;
    // var shipping = CalculateShipping(order);
    var total = order.Total;  // Used to include tax and shipping
    // total = total + tax + shipping;
    
    _repository.Save(order);
}

// ✅ Remove dead code, use git history if needed
public void ProcessOrder(Order order)
{
    _repository.Save(order);
}
```

### ❌ DON'T Comment: To Excuse Bad Code

Refactor instead:

```csharp
// ❌ Comment apologizing for messy code
// This is a bit hacky but it works
var result = data.Split(',')[0].Trim().ToLower().Replace(" ", "");

// ✅ Extract to well-named method
var result = NormalizeCustomerName(data);

private string NormalizeCustomerName(string rawData)
{
    return rawData
        .Split(',')[0]
        .Trim()
        .ToLower()
        .Replace(" ", "");
}
```

### ❌ DON'T Comment: Variable/Function Purpose

Use better names instead:

```csharp
// ❌ Name doesn't reveal intent
// Days until order expires
var d = 7;

// ✅ Name reveals intent, no comment needed
var daysUntilOrderExpires = 7;
```

### ❌ DON'T Comment: Redundant Information

Avoid repeating what the code already says:

```csharp
// ❌ Comment adds no value
/// <summary>
/// Gets the user.
/// </summary>
/// <param name="id">The user id.</param>
/// <returns>The user.</returns>
public User GetUser(int id)
{
    return _repository.Find(id);
}

// ✅ Add value or remove comment
/// <summary>
/// Retrieves user from cache if available, otherwise queries database.
/// Returns null if user is not found or has been deleted.
/// </summary>
public User? GetUser(UserId id)
{
    return _cache.TryGetValue(id, out var user) 
        ? user 
        : _repository.Find(id);
}
```

## Self-Documenting Code Techniques

### Use Descriptive Names

```csharp
// ❌ Needs comments to explain
var d = 86400;  // Seconds in a day
var t = DateTime.Now.AddSeconds(d);  // Tomorrow

// ✅ Self-explanatory
var secondsPerDay = 86400;
var tomorrow = DateTime.Now.AddSeconds(secondsPerDay);
```

### Extract Methods

```csharp
// ❌ Complex condition needs comment
// Check if user is eligible for free shipping (over $50 and prime member)
if (order.Total > 50 && user.IsPrimeMember)
{
    ApplyFreeShipping(order);
}

// ✅ Method name explains the logic
if (IsEligibleForFreeShipping(order, user))
{
    ApplyFreeShipping(order);
}

private bool IsEligibleForFreeShipping(Order order, User user)
{
    return order.Total > 50 && user.IsPrimeMember;
}
```

### Use Value Objects

```csharp
// ❌ Magic numbers need comments
// Apply 25% discount for VIP customers
var discountedPrice = price * 0.75m;

// ✅ Value object encapsulates meaning
var vipDiscount = Discount.VIP;  // 25%
var discountedPrice = price.Apply(vipDiscount);
```

## Documentation Smells

Signs your comments are problematic:

- **Obvious comments** that restate the code
- **Outdated comments** that contradict the code
- **Too many comments** suggesting unclear code
- **Apologetic comments** ("this is messy", "hack")
- **Attribution comments** (use version control)
- **Closing brace comments** (functions too long)

## Benefits

- **Maintainable codebase** where code and comments stay in sync
- **Faster onboarding** with self-documenting code
- **Less cognitive overhead** reading code without noise
- **Preserved intent** for complex business rules and algorithms
- **Historical context** for future refactoring decisions

## See Also

- [Meaningful Names](./meaningful-names.md) — Names that eliminate the need for comments
- [Small Functions](./small-functions.md) — Functions simple enough to not need comments
- [Ubiquitous Language](./ubiquitous-language.md) — Using domain terms that explain themselves
