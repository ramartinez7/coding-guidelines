# Immutable Collections

> Use `System.Collections.Immutable` for collections that truly cannot change—not just read-only views.

## Problem

`IReadOnlyList<T>` and `IReadOnlyCollection<T>` are not immutable—they're read-only views over potentially mutable collections. The underlying collection can change, breaking assumptions and causing bugs. True immutability requires `System.Collections.Immutable`.

## Example

### ❌ Before

```csharp
public class Order
{
    private readonly List<OrderItem> items = new();

    public IReadOnlyList<OrderItem> Items => items;

    public void AddItem(OrderItem item)
    {
        items.Add(item);
    }
}

// The "read-only" collection can still change!
var order = new Order();
var itemsSnapshot = order.Items;

Console.WriteLine(itemsSnapshot.Count);  // 0

order.AddItem(new OrderItem(...));

Console.WriteLine(itemsSnapshot.Count);  // 1 - the "snapshot" changed!

// Caller can cast back to mutable collection
var mutableItems = (List<OrderItem>)order.Items;  // Might work!
mutableItems.Add(new OrderItem(...));  // Bypassing encapsulation
```

### ✅ After

```csharp
using System.Collections.Immutable;

public class Order
{
    private ImmutableList<OrderItem> items = ImmutableList<OrderItem>.Empty;

    public ImmutableList<OrderItem> Items => items;

    public void AddItem(OrderItem item)
    {
        items = items.Add(item);  // Returns new list
    }
}

// True immutability
var order = new Order();
var itemsSnapshot = order.Items;

Console.WriteLine(itemsSnapshot.Count);  // 0

order.AddItem(new OrderItem(...));

Console.WriteLine(itemsSnapshot.Count);  // Still 0 - true snapshot!

// Cannot cast to mutable
var mutableItems = (List<OrderItem>)order.Items;  // Runtime error
```

## Why It's Better

1. **True immutability**: Collections cannot change after creation.

2. **Safe snapshots**: Captured references never change.

3. **Thread-safe**: No synchronization needed for reads.

4. **No defensive copies**: Can share references safely.

5. **Structural sharing**: Efficient memory usage through shared structure.

## Immutable Collection Types

### 1. ImmutableList\<T\>

Indexed collection with efficient random access:

```csharp
using System.Collections.Immutable;

// Create
var list1 = ImmutableList<string>.Empty;
var list2 = ImmutableList.Create("A", "B", "C");
var list3 = new[] { "A", "B", "C" }.ToImmutableList();

// Add (returns new list)
var list4 = list1.Add("X");
var list5 = list4.Add("Y");

// Remove (returns new list)
var list6 = list5.Remove("X");

// Original unchanged
Console.WriteLine(list1.Count);  // 0
Console.WriteLine(list4.Count);  // 1
```

### 2. ImmutableArray\<T\>

Contiguous memory, fastest reads, no structural sharing:

```csharp
// Best for small collections accessed frequently
var array1 = ImmutableArray<int>.Empty;
var array2 = ImmutableArray.Create(1, 2, 3, 4, 5);

// All mutations return new arrays
var array3 = array2.Add(6);
var array4 = array2.RemoveAt(0);

// Very fast iteration
foreach (var item in array2)
{
    // No overhead, direct memory access
}
```

### 3. ImmutableDictionary\<TKey, TValue\>

Key-value pairs with efficient lookups:

```csharp
var dict1 = ImmutableDictionary<UserId, User>.Empty;

var dict2 = dict1.Add(userId1, user1);
var dict3 = dict2.Add(userId2, user2);

// TryGetValue for safe lookups
if (dict3.TryGetValue(userId1, out var user))
{
    // Use user
}

// SetItem replaces if exists, adds if not
var dict4 = dict3.SetItem(userId1, updatedUser);

// Remove
var dict5 = dict4.Remove(userId1);
```

### 4. ImmutableHashSet\<T\>

Unique elements with fast membership testing:

```csharp
var set1 = ImmutableHashSet<ProductId>.Empty;

var set2 = set1.Add(productId1);
var set3 = set2.Add(productId2);
var set4 = set3.Add(productId1);  // Duplicate - no change

Console.WriteLine(set4.Count);  // 2

// Set operations
var set5 = set2.Union(set3);
var set6 = set2.Intersect(set3);
var set7 = set2.Except(set3);
```

### 5. ImmutableQueue\<T\> and ImmutableStack\<T\>

FIFO and LIFO collections:

```csharp
// Queue (FIFO)
var queue1 = ImmutableQueue<string>.Empty;
var queue2 = queue1.Enqueue("First");
var queue3 = queue2.Enqueue("Second");

var first = queue3.Peek();  // "First"
var queue4 = queue3.Dequeue(out var item);  // item = "First"

// Stack (LIFO)
var stack1 = ImmutableStack<string>.Empty;
var stack2 = stack1.Push("First");
var stack3 = stack2.Push("Second");

var top = stack3.Peek();  // "Second"
var stack4 = stack3.Pop(out var item);  // item = "Second"
```

## Performance Characteristics

### Structural Sharing

Immutable collections share structure between versions:

```csharp
var list1 = ImmutableList.Create(1, 2, 3, 4, 5);  // 5 nodes

var list2 = list1.Add(6);  // Shares first 5 nodes, adds 1 new node
var list3 = list1.Add(7);  // Shares first 5 nodes, adds different node

// Memory-efficient: 7 nodes total, not 17
```

### When to Use Each Type

| Type | Use When | Complexity |
|------|----------|------------|
| `ImmutableArray<T>` | Small, read-heavy, rarely modified | O(n) modifications |
| `ImmutableList<T>` | General purpose, balanced operations | O(log n) access/modify |
| `ImmutableDictionary<TK,TV>` | Key-value lookups, frequent changes | O(log n) operations |
| `ImmutableHashSet<T>` | Unique items, membership testing | O(log n) operations |
| `ImmutableQueue<T>` | FIFO operations | O(1) enqueue/dequeue |
| `ImmutableStack<T>` | LIFO operations | O(1) push/pop |

## Builder Pattern for Efficiency

When making many changes, use builders to avoid intermediate allocations:

```csharp
// ❌ Inefficient - creates intermediate lists
var list = ImmutableList<int>.Empty;
foreach (var i in Enumerable.Range(1, 1000))
{
    list = list.Add(i);  // 1000 intermediate lists!
}

// ✅ Efficient - uses builder
var builder = ImmutableList.CreateBuilder<int>();
foreach (var i in Enumerable.Range(1, 1000))
{
    builder.Add(i);  // Mutable during build
}
var list = builder.ToImmutable();  // One final immutable list
```

## Domain Modeling with Immutable Collections

### Value Objects

```csharp
public sealed record OrderItems
{
    private readonly ImmutableList<OrderItem> items;

    private OrderItems(ImmutableList<OrderItem> items)
    {
        this.items = items;
    }

    public static Result<OrderItems, string> Create(IEnumerable<OrderItem> items)
    {
        var itemList = items.ToImmutableList();
        
        if (itemList.IsEmpty)
            return Result<OrderItems, string>.Failure("Order must have at least one item");
        
        if (itemList.Any(i => i.Quantity <= 0))
            return Result<OrderItems, string>.Failure("All items must have positive quantity");
        
        return Result<OrderItems, string>.Success(new OrderItems(itemList));
    }

    public OrderItems AddItem(OrderItem item)
    {
        return new OrderItems(items.Add(item));
    }

    public int Count => items.Count;

    public ImmutableList<OrderItem> ToImmutableList() => items;
}
```

### Immutable Entities (Event Sourcing)

```csharp
public sealed record ShoppingCart
{
    public CartId Id { get; init; }
    public CustomerId CustomerId { get; init; }
    public ImmutableList<CartItem> Items { get; init; } = ImmutableList<CartItem>.Empty;

    public ShoppingCart AddItem(ProductId productId, Quantity quantity)
    {
        var newItem = new CartItem(productId, quantity);
        return this with { Items = Items.Add(newItem) };
    }

    public ShoppingCart RemoveItem(ProductId productId)
    {
        var updatedItems = Items.RemoveAll(i => i.ProductId == productId);
        return this with { Items = updatedItems };
    }

    public ShoppingCart UpdateQuantity(ProductId productId, Quantity newQuantity)
    {
        var itemIndex = Items.FindIndex(i => i.ProductId == productId);
        if (itemIndex == -1)
            return this;
        
        var updatedItems = Items.SetItem(itemIndex, Items[itemIndex] with { Quantity = newQuantity });
        return this with { Items = updatedItems };
    }
}
```

## Comparison with Alternatives

### IReadOnlyList\<T\> vs ImmutableList\<T\>

```csharp
// IReadOnlyList<T> - just a view
List<int> mutable = new() { 1, 2, 3 };
IReadOnlyList<int> readOnly = mutable;

mutable.Add(4);
Console.WriteLine(readOnly.Count);  // 4 - changed!

// ImmutableList<T> - truly immutable
ImmutableList<int> immutable = ImmutableList.Create(1, 2, 3);
ImmutableList<int> snapshot = immutable;

immutable = immutable.Add(4);
Console.WriteLine(snapshot.Count);  // 3 - unchanged!
```

### ToArray() vs ToImmutableArray()

```csharp
// ToArray() - defensive copy, wasteful
public IEnumerable<OrderItem> GetItems()
{
    return items.ToArray();  // New array every call
}

// ToImmutableArray() - can return same instance
private ImmutableArray<OrderItem> items;

public ImmutableArray<OrderItem> GetItems()
{
    return items;  // Safe to return directly
}
```

## When to Use

Use immutable collections when:

✅ Collections are shared across threads  
✅ You need true snapshots  
✅ Collections represent value objects  
✅ Event sourcing or functional architectures  
✅ Preventing accidental mutations is critical  

Don't use when:

❌ Building up large collections incrementally (use builder)  
❌ Hot paths where allocations matter (profile first)  
❌ Working with external APIs expecting mutable collections  
❌ Simple internal state that's never shared  

## See Also

- [Snapshot Immutability](./snapshot-immutability.md) — IReadOnlyList\<T\> is not immutable
- [Value Semantics](./value-semantics.md) — immutability in value objects
- [First-Class Collections](./first-class-collections.md) — wrapping collections in domain types
- [Record Equality](./record-equality.md) — immutable records
