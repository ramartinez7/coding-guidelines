# First-Class Collections

> Collections wrapped in dedicated types enforce invariants, provide domain-specific operations, and make intent explicit.

## Problem

Passing raw collections (`List<T>`, `IEnumerable<T>`) throughout the codebase exposes implementation details, loses domain meaning, and allows invariants to be violated. Operations on these collections become scattered across services.

## Example

### ❌ Before

```csharp
public class OrderService
{
    public void ProcessOrder(List<OrderItem> items)
    {
        if (items == null || items.Count == 0)
            throw new ArgumentException("Order must have at least one item");
        
        if (items.Any(i => i.Quantity <= 0))
            throw new ArgumentException("All items must have positive quantity");
        
        decimal total = items.Sum(i => i.Price * i.Quantity);
        // ...
    }
}

public class InvoiceService
{
    public void CreateInvoice(List<OrderItem> items)
    {
        // Duplicate validation
        if (items == null || items.Count == 0)
            throw new ArgumentException("Invoice must have at least one item");
        
        // ...
    }
}
```

### ✅ After

```csharp
public sealed class OrderItems
{
    private readonly IReadOnlyList<OrderItem> items;

    private OrderItems(IEnumerable<OrderItem> items)
    {
        this.items = items.ToList();
    }

    public static Result<OrderItems, string> Create(IEnumerable<OrderItem> items)
    {
        if (items == null)
            return Result<OrderItems, string>.Failure("Items cannot be null");
        
        var itemList = items.ToList();
        
        if (itemList.Count == 0)
            return Result<OrderItems, string>.Failure("Order must have at least one item");
        
        if (itemList.Any(i => i.Quantity <= 0))
            return Result<OrderItems, string>.Failure("All items must have positive quantity");
        
        return Result<OrderItems, string>.Success(new OrderItems(itemList));
    }

    public Money CalculateTotal()
    {
        return items
            .Select(i => i.Price * i.Quantity)
            .Aggregate(Money.Zero, (acc, price) => acc + price);
    }

    public bool Contains(ProductId productId)
    {
        return items.Any(i => i.ProductId == productId);
    }

    public int Count => items.Count;

    public IEnumerable<OrderItem> AsEnumerable() => items;
}

public class OrderService
{
    public void ProcessOrder(OrderItems items)
    {
        // No validation needed - OrderItems guarantees validity
        var total = items.CalculateTotal();
        // ...
    }
}

public class InvoiceService
{
    public void CreateInvoice(OrderItems items)
    {
        // No validation needed - OrderItems guarantees validity
        // ...
    }
}
```

## Why It's a Problem

1. **Lost domain meaning**: `List<OrderItem>` doesn't convey the concept of "an order's items" or what makes them valid.

2. **Scattered validation**: Every consumer must remember to check for null, empty, or invalid items.

3. **Duplicate logic**: Operations like calculating totals get reimplemented in multiple places.

4. **Exposed implementation**: Callers can see (and depend on) that it's a `List` specifically.

5. **Mutable state**: Raw collections can be modified by consumers, breaking invariants.

## Benefits

- **Encapsulated invariants**: Validation happens once at construction
- **Domain operations**: Business logic lives with the data it operates on
- **Single source of truth**: No duplicate validation or calculation logic
- **Meaningful types**: `OrderItems` is more descriptive than `List<OrderItem>`
- **Immutability**: The collection cannot be modified after creation

## Advanced: Domain-Specific Operations

First-class collections can expose domain-specific operations that would be awkward as extension methods:

```csharp
public sealed class ShoppingCart
{
    private readonly Dictionary<ProductId, CartItem> items;

    private ShoppingCart(Dictionary<ProductId, CartItem> items)
    {
        this.items = items;
    }

    public static ShoppingCart Empty() => new(new Dictionary<ProductId, CartItem>());

    public ShoppingCart AddItem(ProductId productId, Quantity quantity)
    {
        var newItems = new Dictionary<ProductId, CartItem>(items);

        if (newItems.ContainsKey(productId))
        {
            var existing = newItems[productId];
            newItems[productId] = existing with { Quantity = existing.Quantity + quantity };
        }
        else
        {
            newItems[productId] = new CartItem(productId, quantity);
        }

        return new ShoppingCart(newItems);
    }

    public ShoppingCart RemoveItem(ProductId productId)
    {
        if (!items.ContainsKey(productId))
            return this;

        var newItems = new Dictionary<ProductId, CartItem>(items);
        newItems.Remove(productId);

        return new ShoppingCart(newItems);
    }

    public bool IsEmpty => items.Count == 0;

    public int TotalItems => items.Values.Sum(i => i.Quantity.Value);

    public IEnumerable<CartItem> Items => items.Values;
}
```

## When to Use

Create a first-class collection when:

- The collection has invariants (non-empty, unique items, sorted, size limits)
- Operations on the collection represent domain concepts (calculate total, find by criteria)
- The collection appears in multiple method signatures
- You need to hide the implementation detail of which collection type is used
- The collection's purpose is significant to the domain (not just "a list of things")

## When Not to Use

Avoid wrapping collections when:

- The collection is truly generic with no domain rules
- It appears in only one place
- The collection is just a return value with no operations
- You're just passing data through without transformations

## See Also

- [Primitive Collections](./primitive-collections.md) — related code smell
- [Refinement Types](./refinement-types.md) — enforcing structural constraints
- [Domain Invariants](./domain-invariants.md) — enforcing business rules at construction
- [Value Semantics](./value-semantics.md) — immutable value-based types
- [Snapshot Immutability](./snapshot-immutability.md) — true immutability vs read-only views
