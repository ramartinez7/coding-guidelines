# Polymorphic Feature Flags (Strategy over If)

> Boolean feature flags checked inside business logic—use type registration to swap implementations at startup.

## Problem

Feature flags are often implemented as booleans checked inside business logic. This leads to spaghetti code where old and new logic are intermingled in the same method. When the feature is fully rolled out, you must surgically remove the `else` block—a process that's error-prone and leaves debris.

## Example

### ❌ Before

```csharp
public class CheckoutService
{
    private readonly IFeatureFlagService _flags;
    private readonly IPaymentGateway _paymentGateway;
    private readonly INewPaymentGateway _newPaymentGateway;
    private readonly IInventoryService _inventoryService;
    private readonly INewInventoryService _newInventoryService;

    public async Task<CheckoutResult> ProcessCheckoutAsync(Cart cart)
    {
        // Flag check #1
        decimal total;
        if (_flags.IsEnabled("new-pricing-engine"))
        {
            total = CalculateNewPricing(cart);
        }
        else
        {
            total = CalculateLegacyPricing(cart);
        }

        // Flag check #2
        PaymentResult payment;
        if (_flags.IsEnabled("stripe-v2"))
        {
            payment = await _newPaymentGateway.ChargeAsync(cart.CustomerId, total);
        }
        else
        {
            payment = await _paymentGateway.ChargeAsync(cart.CustomerId, total);
        }

        // Flag check #3
        if (_flags.IsEnabled("real-time-inventory"))
        {
            await _newInventoryService.ReserveAsync(cart.Items);
        }
        else
        {
            await _inventoryService.ReserveAsync(cart.Items);
        }

        // Flag check #4 - nested complexity
        if (_flags.IsEnabled("new-confirmation-email"))
        {
            if (_flags.IsEnabled("email-templates-v2"))
            {
                await SendNewTemplateEmailAsync(cart);
            }
            else
            {
                await SendNewEmailAsync(cart);
            }
        }
        else
        {
            await SendLegacyEmailAsync(cart);
        }

        return new CheckoutResult(payment.TransactionId);
    }
}

// Testing requires mocking the flag service
[Fact]
public async Task ProcessCheckout_WithNewPricing_CalculatesCorrectly()
{
    var flags = new Mock<IFeatureFlagService>();
    flags.Setup(f => f.IsEnabled("new-pricing-engine")).Returns(true);
    flags.Setup(f => f.IsEnabled("stripe-v2")).Returns(false);
    flags.Setup(f => f.IsEnabled("real-time-inventory")).Returns(false);
    flags.Setup(f => f.IsEnabled("new-confirmation-email")).Returns(false);
    // ... setup all the other mocks

    var service = new CheckoutService(flags.Object, /* 4 more dependencies */);
    // Test is coupled to flag names and internal branching logic
}
```

**Problems:**
- Business logic polluted with flag checks
- Old and new code intermingled in same method
- Testing requires mocking flag service + understanding internal branches
- Cleanup requires surgical removal of `else` blocks
- Nested flags create combinatorial complexity

### ✅ After

```csharp
// Define the contract
public interface IPricingEngine
{
    decimal Calculate(Cart cart);
}

public interface IPaymentGateway
{
    Task<PaymentResult> ChargeAsync(CustomerId customerId, decimal amount);
}

public interface IInventoryService
{
    Task ReserveAsync(IReadOnlyList<CartItem> items);
}

// Implement each variant as a separate class
public sealed class LegacyPricingEngine : IPricingEngine
{
    public decimal Calculate(Cart cart)
    {
        // Legacy pricing logic—clean, isolated
        return cart.Items.Sum(i => i.Price * i.Quantity);
    }
}

public sealed class NewPricingEngine : IPricingEngine
{
    private readonly IDiscountService _discounts;

    public NewPricingEngine(IDiscountService discounts)
        => _discounts = discounts;

    public decimal Calculate(Cart cart)
    {
        // New pricing logic—clean, isolated
        var subtotal = cart.Items.Sum(i => i.Price * i.Quantity);
        var discount = _discounts.CalculateDiscount(cart);
        return subtotal - discount;
    }
}

// CheckoutService is clean—no flag checks
public class CheckoutService
{
    private readonly IPricingEngine _pricingEngine;
    private readonly IPaymentGateway _paymentGateway;
    private readonly IInventoryService _inventoryService;

    public CheckoutService(
        IPricingEngine pricingEngine,
        IPaymentGateway paymentGateway,
        IInventoryService inventoryService)
    {
        _pricingEngine = pricingEngine;
        _paymentGateway = paymentGateway;
        _inventoryService = inventoryService;
    }

    public async Task<CheckoutResult> ProcessCheckoutAsync(Cart cart)
    {
        // No flags—just use the injected implementation
        var total = _pricingEngine.Calculate(cart);
        var payment = await _paymentGateway.ChargeAsync(cart.CustomerId, total);
        await _inventoryService.ReserveAsync(cart.Items);

        return new CheckoutResult(payment.TransactionId);
    }
}
```

**Registration based on flags:**

```csharp
// Program.cs / Startup.cs
builder.Services.AddScoped<CheckoutService>();

// Pricing engine based on flag
if (featureFlags.IsEnabled("new-pricing-engine"))
{
    builder.Services.AddScoped<IPricingEngine, NewPricingEngine>();
}
else
{
    builder.Services.AddScoped<IPricingEngine, LegacyPricingEngine>();
}

// Payment gateway based on flag
if (featureFlags.IsEnabled("stripe-v2"))
{
    builder.Services.AddScoped<IPaymentGateway, StripeV2Gateway>();
}
else
{
    builder.Services.AddScoped<IPaymentGateway, StripeV1Gateway>();
}
```

**Testing is trivial:**

```csharp
[Fact]
public async Task ProcessCheckout_CalculatesTotalAndCharges()
{
    // Inject the specific implementation you want to test
    var pricingEngine = new NewPricingEngine(new FakeDiscountService());
    var paymentGateway = new FakePaymentGateway();
    var inventoryService = new FakeInventoryService();

    var service = new CheckoutService(pricingEngine, paymentGateway, inventoryService);

    var result = await service.ProcessCheckoutAsync(cart);

    Assert.Equal(expectedTotal, paymentGateway.LastChargedAmount);
}

// Test each implementation in isolation
[Fact]
public void NewPricingEngine_AppliesDiscounts()
{
    var discounts = new FakeDiscountService(discount: 10m);
    var engine = new NewPricingEngine(discounts);

    var total = engine.Calculate(cartWith100Total);

    Assert.Equal(90m, total);
}
```

## Dynamic Flag Resolution

For flags that change at runtime (gradual rollouts, A/B testing):

```csharp
// Factory that resolves at request time
public interface IPricingEngineFactory
{
    IPricingEngine Create();
}

public sealed class FeatureFlaggedPricingEngineFactory : IPricingEngineFactory
{
    private readonly IFeatureFlagService _flags;
    private readonly IServiceProvider _services;

    public FeatureFlaggedPricingEngineFactory(
        IFeatureFlagService flags,
        IServiceProvider services)
    {
        _flags = flags;
        _services = services;
    }

    public IPricingEngine Create()
    {
        return _flags.IsEnabled("new-pricing-engine")
            ? _services.GetRequiredService<NewPricingEngine>()
            : _services.GetRequiredService<LegacyPricingEngine>();
    }
}

// Registration
builder.Services.AddScoped<NewPricingEngine>();
builder.Services.AddScoped<LegacyPricingEngine>();
builder.Services.AddScoped<IPricingEngineFactory, FeatureFlaggedPricingEngineFactory>();

// Usage
public class CheckoutService
{
    private readonly IPricingEngineFactory _pricingEngineFactory;

    public async Task<CheckoutResult> ProcessCheckoutAsync(Cart cart)
    {
        var pricingEngine = _pricingEngineFactory.Create();
        var total = pricingEngine.Calculate(cart);
        // ...
    }
}
```

## Percentage Rollouts

```csharp
public sealed class GradualRolloutPricingEngineFactory : IPricingEngineFactory
{
    private readonly IFeatureFlagService _flags;
    private readonly IServiceProvider _services;
    private readonly IHttpContextAccessor _httpContext;

    public IPricingEngine Create()
    {
        // Consistent assignment per user
        var userId = _httpContext.HttpContext?.User.GetUserId();
        var bucket = userId?.GetHashCode() % 100 ?? 0;
        var rolloutPercentage = _flags.GetValue<int>("new-pricing-rollout-percent");

        return bucket < rolloutPercentage
            ? _services.GetRequiredService<NewPricingEngine>()
            : _services.GetRequiredService<LegacyPricingEngine>();
    }
}
```

## Why It's a Problem

1. **Spaghetti code**: Old and new logic intermingled in same methods.

2. **Testing complexity**: Must mock flag service and understand internal branches.

3. **Cleanup debt**: Removing a flag requires editing business logic methods.

4. **Combinatorial explosion**: N flags = 2^N possible code paths.

5. **Hidden dependencies**: Flag names are magic strings scattered in code.

## Symptoms

- Methods with multiple `if (flags.IsEnabled(...))` checks
- Nested flag conditions creating complex branching
- Tests that mock `IFeatureFlagService` extensively
- "Flag cleanup" stories that linger in the backlog
- Bugs from incorrect flag combinations

## Cleanup Is Delete, Not Edit

When the new pricing engine is fully rolled out:

```csharp
// Before cleanup (flag-based):
// - Find all IsEnabled("new-pricing-engine") calls
// - Understand which branch to keep
// - Surgically edit each method
// - Hope you didn't break anything

// After cleanup (polymorphic):
// 1. Delete LegacyPricingEngine.cs
// 2. Change registration:
builder.Services.AddScoped<IPricingEngine, NewPricingEngine>();
// Done. Business logic unchanged.
```

## Composite Strategies

When flags affect multiple behaviors together:

```csharp
public interface ICheckoutStrategy
{
    IPricingEngine PricingEngine { get; }
    IPaymentGateway PaymentGateway { get; }
    IInventoryService InventoryService { get; }
}

public sealed class LegacyCheckoutStrategy : ICheckoutStrategy
{
    public IPricingEngine PricingEngine { get; }
    public IPaymentGateway PaymentGateway { get; }
    public IInventoryService InventoryService { get; }

    public LegacyCheckoutStrategy(/* legacy dependencies */) { }
}

public sealed class NewCheckoutStrategy : ICheckoutStrategy
{
    public IPricingEngine PricingEngine { get; }
    public IPaymentGateway PaymentGateway { get; }
    public IInventoryService InventoryService { get; }

    public NewCheckoutStrategy(/* new dependencies */) { }
}

// One flag controls the entire strategy
if (featureFlags.IsEnabled("checkout-v2"))
{
    builder.Services.AddScoped<ICheckoutStrategy, NewCheckoutStrategy>();
}
else
{
    builder.Services.AddScoped<ICheckoutStrategy, LegacyCheckoutStrategy>();
}
```

## Benefits

- **Clean business logic**: No flag checks polluting code
- **Isolated implementations**: Each variant is a separate, testable class
- **Easy cleanup**: Delete old class, change registration
- **Type safety**: Compiler ensures interface contracts are met
- **Single Responsibility**: Each class does one thing one way

## When Flags in Code Are OK

Not every flag needs polymorphism:

| Scenario | Use Polymorphism? | Reason |
|----------|-------------------|--------|
| Different algorithms/strategies | ✅ Yes | Clean separation |
| Simple UI toggle | ❌ No | `if (showBanner)` is fine |
| Kill switch for external service | ❌ No | Simple circuit breaker |
| Complex branching logic | ✅ Yes | Avoid spaghetti |
| A/B test with metrics | ✅ Yes | Each variant testable |

## See Also

- [Flag Arguments](./flag-arguments.md)
- [Enum to Class Hierarchy](./enum-to-class-hierarchy.md)
- [Specification Pattern (Logic as Types)](./specification-pattern.md)
