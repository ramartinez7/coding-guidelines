# Law of Demeter (Principle of Least Knowledge)

> Objects should only talk to their immediate friends—don't navigate through chains of relationships.

## Problem

When code navigates through multiple object relationships ("train wrecks"), it creates tight coupling to the internal structure of distant objects. Changes ripple through the system, and testing becomes difficult.

## Example

### ❌ Before

```csharp
public class OrderProcessor
{
    public void ProcessOrder(Order order)
    {
        // Train wreck: navigating through multiple relationships
        string city = order.Customer.Address.City.Name;
        
        // Another train wreck
        if (order.Payment.Card.Issuer.Country.Code == "US")
        {
            // Apply US-specific processing
        }
        
        // Yet another
        decimal shippingCost = order.ShippingDetails.Carrier.RateCalculator.Calculate(
            order.ShippingDetails.Weight);
    }
}

// Changes to any intermediate object break this code
public class Address
{
    public City City { get; set; }  // If this changes to string, ProcessOrder breaks
}
```

### ✅ After

```csharp
public class Order
{
    private readonly Customer customer;
    private readonly Payment payment;
    private readonly ShippingDetails shippingDetails;

    // Tell, don't ask: Provide methods that hide the navigation
    public string GetShippingCity()
    {
        return customer.GetShippingCity();
    }

    public bool IsPaymentFromUS()
    {
        return payment.IsFromCountry(CountryCode.US);
    }

    public Money CalculateShippingCost()
    {
        return shippingDetails.CalculateCost();
    }
}

public class OrderProcessor
{
    public void ProcessOrder(Order order)
    {
        // Talk only to the order, not its friends' friends
        string city = order.GetShippingCity();
        
        if (order.IsPaymentFromUS())
        {
            // Apply US-specific processing
        }
        
        decimal shippingCost = order.CalculateShippingCost();
    }
}
```

## Why It's a Problem

1. **Tight coupling**: Code depends on the internal structure of distant objects.

2. **Ripple effects**: Changes to intermediate objects break unrelated code.

3. **Testing difficulty**: Must mock entire object graphs to test one method.

4. **Readability**: Long navigation chains are hard to understand.

5. **Broken encapsulation**: Internal structure is exposed to callers.

## The Rule

An object should only call methods on:

1. **Itself** (`this.DoSomething()`)
2. **Objects passed as parameters** (`parameter.DoSomething()`)
3. **Objects it creates** (`var obj = new Thing(); obj.DoSomething()`)
4. **Its direct fields/properties** (`this.field.DoSomething()`)

### ❌ Violations

```csharp
// Violation: Navigating through multiple dots
customer.GetAddress().GetCity().GetName();

// Violation: Getting an object just to call a method on its property
payment.GetCard().GetIssuer().Validate();

// Violation: Chain of property accesses
order.ShippingDetails.Carrier.RateCalculator.BaseRate;
```

### ✅ Following the Law

```csharp
// Good: Tell the object what you need
string cityName = customer.GetCityName();

// Good: Ask the direct object to perform the operation
bool isValid = payment.ValidateIssuer();

// Good: Ask the order to calculate, hiding the complexity
decimal baseRate = order.GetBaseShippingRate();
```

## Refactoring Strategies

### Strategy 1: Delegate Methods

Move the navigation into the object being navigated:

```csharp
// ❌ Before
decimal price = product.Pricing.CalculatePrice(customer.Membership.Tier.DiscountRate);

// ✅ After
// Product.cs
public Money CalculatePriceFor(Customer customer)
{
    return pricing.CalculatePrice(customer.GetDiscountRate());
}

// Customer.cs
public decimal GetDiscountRate()
{
    return membership.GetDiscountRate();
}

// Usage
decimal price = product.CalculatePriceFor(customer);
```

### Strategy 2: Extract Query Methods

Create methods that encapsulate common queries:

```csharp
// ❌ Before
if (order.Customer.Address.Country.RequiresCustomsDeclaration())
{
    CreateCustomsDeclaration(order);
}

// ✅ After
// Order.cs
public bool RequiresCustomsDeclaration()
{
    return customer.RequiresCustomsDeclaration();
}

// Customer.cs  
public bool RequiresCustomsDeclaration()
{
    return address.RequiresCustomsDeclaration();
}

// Usage
if (order.RequiresCustomsDeclaration())
{
    CreateCustomsDeclaration(order);
}
```

### Strategy 3: Pass Specific Data

Instead of passing complex objects, pass only what's needed:

```csharp
// ❌ Before
void ProcessShipment(Order order)
{
    var weight = order.ShippingDetails.Package.Weight;
    var destination = order.Customer.Address.ZipCode;
    // ...
}

// ✅ After
void ProcessShipment(Weight weight, ZipCode destination)
{
    // ...
}

// Caller
ProcessShipment(order.GetShipmentWeight(), order.GetDestinationZipCode());
```

## When to Break the Rule

### Fluent APIs and Builders

Fluent interfaces intentionally chain calls:

```csharp
// OK: Fluent builder pattern
var query = new QueryBuilder()
    .Select("Name", "Email")
    .From("Users")
    .Where("Active", true)
    .OrderBy("Name")
    .Build();
```

### Data Transfer Objects (DTOs)

DTOs are data structures, not objects with behavior:

```csharp
// OK: DTOs are designed for navigation
var json = JsonSerializer.Serialize(new
{
    CustomerName = order.Customer.Name,
    City = order.Customer.Address.City,
    Total = order.Total
});
```

### LINQ and Query Expressions

LINQ chains are transformations, not navigation:

```csharp
// OK: LINQ pipelines
var result = orders
    .Where(o => o.Status == OrderStatus.Completed)
    .Select(o => o.Total)
    .Sum();
```

### Value Objects and Immutable Types

Navigation into value objects is generally safe:

```csharp
// OK: Value objects expose their structure
var latitude = location.Coordinates.Latitude;
var day = dateTime.Date.Day;
```

## Benefits

- **Loose coupling**: Changes to internal structures don't propagate
- **Easier testing**: Mock only direct dependencies
- **Better encapsulation**: Internal structure stays hidden
- **Clearer intent**: Method names express what you need, not how to get it
- **Flexibility**: Internal implementation can change without breaking callers

## Real-World Example

### ❌ Before

```csharp
public class InvoiceGenerator
{
    public void GenerateInvoice(Order order)
    {
        var invoice = new Invoice();
        
        // Violating Law of Demeter repeatedly
        invoice.CustomerName = order.Customer.Profile.FullName;
        invoice.CustomerEmail = order.Customer.ContactInfo.Email.Value;
        invoice.BillingAddress = order.Customer.BillingAddress.ToInvoiceFormat();
        invoice.ShippingAddress = order.ShippingDetails.Address.ToInvoiceFormat();
        
        foreach (var item in order.Items)
        {
            invoice.AddLine(
                item.Product.Name,
                item.Product.Sku.Value,
                item.Quantity.Value,
                item.Product.Price.Amount);
        }
        
        invoice.Subtotal = order.Items.Sum(i => i.Product.Price.Amount * i.Quantity.Value);
        invoice.Tax = invoice.Subtotal * order.Customer.Address.TaxJurisdiction.Rate;
        invoice.Total = invoice.Subtotal + invoice.Tax;
    }
}
```

### ✅ After

```csharp
public class Order
{
    public InvoiceData GetInvoiceData()
    {
        return new InvoiceData(
            CustomerName: customer.GetFullName(),
            CustomerEmail: customer.GetEmail(),
            BillingAddress: customer.GetBillingAddressForInvoice(),
            ShippingAddress: shippingDetails.GetAddressForInvoice(),
            LineItems: GetInvoiceLineItems(),
            Subtotal: CalculateSubtotal(),
            Tax: CalculateTax(),
            Total: CalculateTotal());
    }

    private IReadOnlyList<InvoiceLineItem> GetInvoiceLineItems()
    {
        return items.Select(i => i.ToInvoiceLineItem()).ToList();
    }
}

public class InvoiceGenerator
{
    public void GenerateInvoice(Order order)
    {
        // Simple and loosely coupled
        var invoiceData = order.GetInvoiceData();
        var invoice = Invoice.FromData(invoiceData);
    }
}
```

## See Also

- [Tell, Don't Ask](./tell-dont-ask.md) — related principle about encapsulation
- [Aggregate Roots](./aggregate-roots.md) — defining consistency boundaries
- [Anti-Corruption Layer](./anti-corruption-layer.md) — isolating external models
- [DTO vs Domain Boundary](./dto-domain-boundary.md) — separating data structures from domain objects
