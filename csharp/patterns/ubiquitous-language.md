# Ubiquitous Language (Translating Business Terms to Types)

> Technical jargon in code disconnected from business vocabulary—use types that mirror the language domain experts use.

## Problem

Code that uses technical terms (`DTO`, `Manager`, `Handler`, `Processor`) instead of business domain concepts creates a translation barrier between developers and domain experts.

## Example

### ❌ Before

```csharp
public class DataProcessor
{
    public ProcessResult ProcessData(DataTransferObject dto)
    {
        var entity = MapToEntity(dto);
        var validator = new DataValidator();
        
        if (!validator.Validate(entity))
            return ProcessResult.Failure();
        
        _repository.Save(entity);
        return ProcessResult.Success();
    }
}
```

**Problems:**
- Generic names (`DataProcessor`, `DataTransferObject`)
- No domain concepts visible
- Hard for domain experts to understand
- Vocabulary mismatch with business

### ✅ After

```csharp
/// <summary>
/// A customer request to purchase products.
/// Language from domain experts: "place an order"
/// </summary>
public sealed class PlaceOrderCommand
{
    public required CustomerId CustomerId { get; init; }
    public required ShippingAddress DeliveryAddress { get; init; }
    public required List<OrderLine> Items { get; init; }
}

/// <summary>
/// Represents a single line item in an order.
/// Domain experts say "order line" not "order item DTO"
/// </summary>
public sealed record OrderLine(
    ProductId Product,
    Quantity Amount,
    Money UnitPrice);

/// <summary>
/// The service that accepts orders from customers.
/// Domain experts: "order fulfillment"
/// </summary>
public class OrderFulfillmentService
{
    public async Task<OrderResult> AcceptOrder(PlaceOrderCommand command)
    {
        // Validate order can be fulfilled
        var availability = await _warehouse.CheckAvailability(command.Items);
        
        if (!availability.AllItemsInStock)
            return OrderResult.OutOfStock(availability.UnavailableItems);
        
        // Reserve inventory
        var reservation = await _warehouse.ReserveItems(command.Items);
        
        // Create order
        var order = Order.Place(
            customerId: command.CustomerId,
            items: command.Items,
            deliveryAddress: command.DeliveryAddress);
        
        await _orderRepository.SaveAsync(order);
        
        return OrderResult.Accepted(order.Id);
    }
}
```

## Business Concepts as Types

```csharp
// Domain expert: "A customer can have multiple shipping addresses"
public sealed class Customer
{
    private readonly List<ShippingAddress> _addresses = new();
    
    public IReadOnlyList<ShippingAddress> ShippingAddresses => _addresses.AsReadOnly();
    
    public void AddShippingAddress(ShippingAddress address)
    {
        if (_addresses.Any(a => a.Label == address.Label))
            throw new InvalidOperationException($"Address with label '{address.Label}' already exists");
        
        _addresses.Add(address);
    }
}

// Domain expert: "Products can be on promotion with a discount"
public sealed class Product
{
    public Money RegularPrice { get; }
    public Option<Promotion> ActivePromotion { get; private set; }
    
    public Money CurrentPrice => ActivePromotion.Match(
        onSome: promo => promo.ApplyDiscount(RegularPrice),
        onNone: () => RegularPrice);
    
    public void ApplyPromotion(Promotion promotion)
    {
        ActivePromotion = Option<Promotion>.Some(promotion);
    }
}

// Domain expert: "Promotions have a percentage discount and validity period"
public sealed record Promotion(
    DiscountPercentage Discount,
    DateRange ValidityPeriod)
{
    public Money ApplyDiscount(Money originalPrice)
    {
        if (!ValidityPeriod.IsCurrentlyValid())
            return originalPrice;
        
        return originalPrice * (1 - Discount.Value / 100);
    }
}
```

## Workflow Steps as Types

```csharp
// Domain expert: "Orders go through: placed → confirmed → shipped → delivered"
public abstract record OrderState
{
    private OrderState() { }
    
    public sealed record Placed(
        DateTime PlacedAt,
        CustomerId Customer) : OrderState;
    
    public sealed record Confirmed(
        DateTime PlacedAt,
        DateTime ConfirmedAt,
        CustomerId Customer,
        PaymentId Payment) : OrderState;
    
    public sealed record Shipped(
        DateTime PlacedAt,
        DateTime ConfirmedAt,
        DateTime ShippedAt,
        CustomerId Customer,
        PaymentId Payment,
        TrackingNumber Tracking) : OrderState;
    
    public sealed record Delivered(
        DateTime PlacedAt,
        DateTime ConfirmedAt,
        DateTime ShippedAt,
        DateTime DeliveredAt,
        CustomerId Customer,
        PaymentId Payment,
        SignatureProof Signature) : OrderState;
}
```

## Business Rules as Named Types

```csharp
// Domain expert: "Customers with good payment history qualify for premium shipping"
public sealed class PremiumShippingEligibility
{
    public bool IsEligible(Customer customer)
    {
        return customer.PaymentHistory.HasNoLatePayments() &&
               customer.TotalOrderValue > Money.USD(1000) &&
               customer.AccountAge > TimeSpan.FromDays(90);
    }
}

// Domain expert: "Orders over $100 get free shipping"
public sealed class FreeShippingThreshold
{
    private static readonly Money Threshold = Money.USD(100);
    
    public bool QualifiesForFreeShipping(Order order)
    {
        return order.Total >= Threshold;
    }
}

// Domain expert: "We can't ship perishable items internationally"
public sealed class InternationalShippingRestrictions
{
    public bool CanShip(Order order, Country destination)
    {
        if (!destination.IsInternational())
            return true;
        
        return !order.Items.Any(item => item.Product.IsPerishable);
    }
}
```

## Naming Conventions

| Instead of | Use |
|------------|-----|
| `DataProcessor` | `OrderFulfillmentService` |
| `EntityManager` | `CustomerRepository` |
| `DataValidator` | `ShippingAddressValidator` |
| `DTO` | `PlaceOrderCommand` |
| `Status` enum | Explicit states: `Placed`, `Shipped`, `Delivered` |
| `ProcessData()` | `FulfillOrder()`, `ShipPackage()` |

## Why It's a Problem

1. **Communication gap**: Technical terms don't match business vocabulary
2. **Lost context**: Generic names hide domain concepts
3. **Harder onboarding**: New developers must learn two vocabularies
4. **Requirements drift**: Code diverges from business understanding

## Symptoms

- Names ending in `-Manager`, `-Handler`, `-Processor`
- Generic types like `Entity`, `Data`, `Item`
- Glossary needed to map code to business terms
- Business experts can't read code

## Benefits

- **Shared vocabulary**: Code matches business language
- **Better communication**: Domain experts understand code
- **Self-documenting**: Names reveal business concepts
- **Reduced errors**: Less translation between domains

## See Also

- [Aggregate Roots](./aggregate-roots.md) — business boundaries
- [Domain Events](./domain-events.md) — business occurrences
- [Specification Pattern](./specification-pattern.md) — business rules
- [Value Semantics](./value-semantics.md) — business concepts
