# Snapshot Immutability (The "ReadOnly" Lie)

> `IReadOnlyList<T>` is not immutable—it's a view that promises *you* won't change it, but the underlying collection can still mutate. Use `System.Collections.Immutable` for true immutability.

## Problem

C#'s `IReadOnly*` interfaces (`IReadOnlyList<T>`, `IReadOnlyCollection<T>`, `IReadOnlyDictionary<K,V>`) are often mistaken for immutable collections. They are not. These interfaces only hide mutation methods from the consumer—the underlying collection can still change via another reference, another thread, or the original owner.

## The Hidden Danger

```csharp
public class OrderService
{
    private readonly List<Order> _orders = new();

    // Looks safe—returning a "readonly" view
    public IReadOnlyList<Order> GetOrders() => _orders;
}

// Caller thinks they have a stable snapshot...
var orders = orderService.GetOrders();
Console.WriteLine(orders.Count);  // 3

// Meanwhile, another thread or method adds an order
orderService.AddOrder(new Order());

Console.WriteLine(orders.Count);  // 4 — it changed!

// Even worse: the caller can cast and mutate
((List<Order>)orders).Clear();  // Boom — original list is now empty
```

## Example

### ❌ Before

```csharp
public class ShoppingCart
{
    private readonly List<CartItem> _items = new();

    public IReadOnlyList<CartItem> Items => _items;  // "ReadOnly" lie

    public void AddItem(CartItem item) => _items.Add(item);
    
    public void Clear() => _items.Clear();
}

// Consumer code
var cart = new ShoppingCart();
cart.AddItem(new CartItem("Widget", 2));

var items = cart.Items;  // Grab "readonly" reference
var count1 = items.Count;  // 1

cart.AddItem(new CartItem("Gadget", 1));  // Mutate underlying list

var count2 = items.Count;  // 2 — our "snapshot" changed!

// Thread safety disaster
Task.Run(() => 
{
    foreach (var item in items)  // Enumerating...
    {
        Process(item);
    }
});

cart.Clear();  // InvalidOperationException: Collection was modified!
```

### ✅ After

```csharp
using System.Collections.Immutable;

public class ShoppingCart
{
    private ImmutableList<CartItem> _items = ImmutableList<CartItem>.Empty;

    public ImmutableList<CartItem> Items => _items;  // True immutability

    public void AddItem(CartItem item) => 
        _items = _items.Add(item);  // Returns new list
    
    public void Clear() => 
        _items = ImmutableList<CartItem>.Empty;
}

// Consumer code
var cart = new ShoppingCart();
cart.AddItem(new CartItem("Widget", 2));

var snapshot = cart.Items;  // True snapshot
var count1 = snapshot.Count;  // 1

cart.AddItem(new CartItem("Gadget", 1));  // Creates new internal list

var count2 = snapshot.Count;  // Still 1 — snapshot is stable!

// Thread-safe iteration
Task.Run(() => 
{
    foreach (var item in snapshot)  // Safe—snapshot never changes
    {
        Process(item);
    }
});

cart.Clear();  // No problem—snapshot is independent
```

## Why It's a Problem

1. **False sense of security**: Developers assume "readonly" means "won't change" when it only means "you can't change it through this reference".

2. **Thread safety bugs**: Iterating over an `IReadOnlyList<T>` while another thread modifies the underlying collection throws `InvalidOperationException`.

3. **Broken snapshots**: Code that caches a "readonly" collection for later use sees unexpected changes.

4. **Casting exploits**: Callers can cast `IReadOnlyList<T>` back to `List<T>` and mutate the original.

5. **API trust issues**: Returning `IReadOnlyList<T>` from public APIs doesn't protect internal state.

## Symptoms

- `InvalidOperationException: Collection was modified during enumeration`
- Defensive `.ToList()` calls everywhere to create "safe copies"
- Race conditions in multi-threaded code with shared collections
- Comments like "don't modify this" or "make a copy first"
- Unit tests that pass individually but fail when run together

## The IReadOnly Interfaces Explained

| Interface | What it means | What it doesn't mean |
|-----------|---------------|----------------------|
| `IReadOnlyList<T>` | You can't add/remove via this reference | Collection won't change |
| `IReadOnlyCollection<T>` | You can't modify via this reference | Collection is thread-safe |
| `IReadOnlyDictionary<K,V>` | You can't write via this reference | Contents are stable |

## Immutable Collections from System.Collections.Immutable

| Type | Description |
|------|-------------|
| `ImmutableList<T>` | Immutable list with O(log n) access |
| `ImmutableArray<T>` | Immutable array with O(1) access, best for fixed data |
| `ImmutableDictionary<K,V>` | Immutable dictionary |
| `ImmutableHashSet<T>` | Immutable hash set |
| `ImmutableStack<T>` | Immutable LIFO stack |
| `ImmutableQueue<T>` | Immutable FIFO queue |

## Choosing the Right Collection

```csharp
// Use ImmutableArray<T> for small, fixed collections
public static readonly ImmutableArray<string> ValidStatuses = 
    ImmutableArray.Create("Pending", "Active", "Completed");

// Use ImmutableList<T> for collections that grow/shrink
private ImmutableList<Order> _orders = ImmutableList<Order>.Empty;

// Use ImmutableDictionary<K,V> for lookup tables
private ImmutableDictionary<CustomerId, Customer> _cache = 
    ImmutableDictionary<CustomerId, Customer>.Empty;
```

## Builder Pattern for Efficient Construction

Building immutable collections item-by-item creates many intermediate instances. Use builders for efficient batch construction:

```csharp
// ❌ Inefficient: creates 1000 intermediate ImmutableList instances
var list = ImmutableList<int>.Empty;
for (int i = 0; i < 1000; i++)
{
    list = list.Add(i);  // Each Add creates a new list
}

// ✅ Efficient: single allocation
var builder = ImmutableList.CreateBuilder<int>();
for (int i = 0; i < 1000; i++)
{
    builder.Add(i);  // Mutates the builder
}
var list = builder.ToImmutable();  // Single immutable list created
```

## Benefits

- **True snapshots**: Captured references never change
- **Thread-safe by design**: No locking needed for reads
- **No defensive copying**: Pass immutable collections freely
- **Structural sharing**: Modifications share structure with original (memory-efficient)
- **API safety**: Callers cannot mutate your internal state

## When IReadOnly Is Acceptable

`IReadOnlyList<T>` is fine when:

- The collection is created once and never modified
- You're exposing a local variable that goes out of scope immediately
- Performance is critical and you understand the risks
- You're using it as a parameter type (caller owns the collection)

```csharp
// OK: parameter type—caller is responsible for the collection
public decimal CalculateTotal(IReadOnlyList<CartItem> items)
{
    return items.Sum(i => i.Price * i.Quantity);
}

// NOT OK: return type exposing mutable internal state
public IReadOnlyList<CartItem> GetItems() => _items;  // Use ImmutableList
```

## See Also

- [Value Semantics](./value-semantics.md)
- [Primitive Collections](./primitive-collections.md)
- [Refinement Types](./refinement-types.md)
