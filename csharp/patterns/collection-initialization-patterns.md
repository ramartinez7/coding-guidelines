# Collection Initialization Patterns

> Use modern collection initialization syntax, collection expressions, and efficient collection types for clean, performant code.

## Problem

Verbose collection initialization, inefficient collection types, and unnecessary allocations lead to code clutter and poor performance.

## Example

### ❌ Before

```csharp
public class OrderService
{
    public List<string> GetProductNames()
    {
        var names = new List<string>();
        names.Add("Product A");
        names.Add("Product B");
        names.Add("Product C");
        return names;
    }

    public Dictionary<string, int> GetInventory()
    {
        var inventory = new Dictionary<string, int>();
        inventory.Add("Apple", 10);
        inventory.Add("Banana", 15);
        inventory.Add("Orange", 8);
        return inventory;
    }

    public HashSet<int> GetValidIds()
    {
        var ids = new HashSet<int>();
        ids.Add(1);
        ids.Add(2);
        ids.Add(3);
        return ids;
    }
}
```

### ✅ After

```csharp
public class OrderService
{
    // ✅ Collection initializer syntax
    public List<string> GetProductNames()
    {
        return new List<string>
        {
            "Product A",
            "Product B",
            "Product C"
        };
    }

    // ✅ Dictionary initializer
    public Dictionary<string, int> GetInventory()
    {
        return new Dictionary<string, int>
        {
            ["Apple"] = 10,
            ["Banana"] = 15,
            ["Orange"] = 8
        };
    }

    // ✅ Collection expressions (C# 12+)
    public HashSet<int> GetValidIds()
    {
        return [1, 2, 3];
    }

    // ✅ Even better: use readonly collections
    public IReadOnlyList<string> GetProductNamesReadOnly()
    {
        return new[] { "Product A", "Product B", "Product C" };
    }
}
```

## Best Practices

### 1. Use Collection Initializers

```csharp
// ❌ Verbose
var colors = new List<string>();
colors.Add("Red");
colors.Add("Green");
colors.Add("Blue");

// ✅ Collection initializer
var colors = new List<string> { "Red", "Green", "Blue" };

// ✅ Collection expression (C# 12+)
List<string> colors = ["Red", "Green", "Blue"];
```

### 2. Use Collection Expressions (C# 12+)

```csharp
// ✅ Spread operator
int[] prefix = [1, 2, 3];
int[] suffix = [7, 8, 9];
int[] combined = [..prefix, 4, 5, 6, ..suffix];

// ✅ Conditional elements
bool includeOptional = true;
int[] numbers = [1, 2, 3, .. includeOptional ? [4, 5] : []];

// ✅ Works with any collection type
List<int> list = [1, 2, 3];
HashSet<int> set = [1, 2, 3];
ImmutableArray<int> immutable = [1, 2, 3];
```

### 3. Use Index Initializers for Dictionaries

```csharp
// ❌ Add method
var settings = new Dictionary<string, string>();
settings.Add("Timeout", "30");
settings.Add("MaxRetries", "3");

// ✅ Collection initializer (Add syntax)
var settings = new Dictionary<string, string>
{
    { "Timeout", "30" },
    { "MaxRetries", "3" }
};

// ✅ Index initializer (more readable)
var settings = new Dictionary<string, string>
{
    ["Timeout"] = "30",
    ["MaxRetries"] = "3"
};
```

### 4. Use Target-Typed New (C# 9+)

```csharp
// ❌ Redundant type
List<string> names = new List<string> { "Alice", "Bob" };

// ✅ Target-typed new
List<string> names = new() { "Alice", "Bob" };

// ✅ With readonly collections
IReadOnlyList<string> names = new List<string> { "Alice", "Bob" };
```

### 5. Choose Appropriate Collection Types

```csharp
// ❌ Using List when HashSet is better
var uniqueIds = new List<int> { 1, 2, 3, 1, 2 };  // Duplicates!

// ✅ Use HashSet for uniqueness
var uniqueIds = new HashSet<int> { 1, 2, 3 };

// ❌ Using List for lookup
var userIds = new List<int> { 1, 2, 3, 4, 5 };
if (userIds.Contains(3)) { }  // O(n) lookup

// ✅ Use HashSet for fast lookup
var userIds = new HashSet<int> { 1, 2, 3, 4, 5 };
if (userIds.Contains(3)) { }  // O(1) lookup
```

### 6. Use Array for Fixed-Size Collections

```csharp
// ❌ List for fixed data
var daysOfWeek = new List<string>
{
    "Monday", "Tuesday", "Wednesday", 
    "Thursday", "Friday", "Saturday", "Sunday"
};

// ✅ Array for fixed-size data (less memory, faster)
var daysOfWeek = new[] 
{
    "Monday", "Tuesday", "Wednesday",
    "Thursday", "Friday", "Saturday", "Sunday"
};

// ✅ Or readonly list
private static readonly IReadOnlyList<string> DaysOfWeek = new[]
{
    "Monday", "Tuesday", "Wednesday",
    "Thursday", "Friday", "Saturday", "Sunday"
};
```

### 7. Use Immutable Collections for Thread Safety

```csharp
// ❌ Mutable shared state
public class ConfigService
{
    public List<string> AllowedOrigins { get; } = new List<string>
    {
        "https://example.com",
        "https://app.example.com"
    };
}

// ✅ Immutable collection
public class ConfigService
{
    public ImmutableList<string> AllowedOrigins { get; } = ImmutableList.Create(
        "https://example.com",
        "https://app.example.com"
    );
}

// ✅ Or with initializer
public ImmutableList<string> AllowedOrigins { get; } =
    ImmutableList.CreateRange(new[]
    {
        "https://example.com",
        "https://app.example.com"
    });
```

### 8. Preallocate Capacity for Known Sizes

```csharp
// ❌ Default capacity (resize overhead)
var items = new List<Item>();
for (int i = 0; i < 1000; i++)
{
    items.Add(CreateItem(i));  // May resize multiple times
}

// ✅ Preallocate capacity
var items = new List<Item>(1000);  // No resizing needed
for (int i = 0; i < 1000; i++)
{
    items.Add(CreateItem(i));
}

// ✅ Or use collection initializer if values known
var items = Enumerable.Range(0, 1000)
    .Select(i => CreateItem(i))
    .ToList();
```

### 9. Use Span for Stack-Allocated Collections

```csharp
// ❌ Heap allocation for small, temporary array
public void ProcessSmallBatch()
{
    var items = new[] { 1, 2, 3, 4, 5 };
    ProcessItems(items);
}

// ✅ Stack allocation (no GC pressure)
public void ProcessSmallBatch()
{
    Span<int> items = stackalloc int[] { 1, 2, 3, 4, 5 };
    ProcessItems(items);
}
```

### 10. Use ReadOnlySpan for Constants

```csharp
// ❌ Static array (heap allocated)
private static readonly string[] ValidStatuses = 
    new[] { "Pending", "Approved", "Rejected" };

// ✅ ReadOnlySpan (no allocation)
private static ReadOnlySpan<string> ValidStatuses =>
    new[] { "Pending", "Approved", "Rejected" };
```

### 11. Use Collection Builders for Complex Init

```csharp
// ❌ Mixing initialization and logic
public List<Product> GetProducts()
{
    var products = new List<Product>
    {
        new Product { Id = 1, Name = "A" },
        new Product { Id = 2, Name = "B" }
    };

    if (_includeSpecial)
    {
        products.Add(new Product { Id = 3, Name = "Special" });
    }

    return products;
}

// ✅ Use LINQ or builder pattern
public List<Product> GetProducts()
{
    var baseProducts = new[]
    {
        new Product { Id = 1, Name = "A" },
        new Product { Id = 2, Name = "B" }
    };

    var specialProducts = _includeSpecial
        ? new[] { new Product { Id = 3, Name = "Special" } }
        : Array.Empty<Product>();

    return baseProducts.Concat(specialProducts).ToList();
}

// ✅ Or collection expressions with spread (C# 12+)
public List<Product> GetProducts()
{
    Product[] baseProducts = 
    [
        new Product { Id = 1, Name = "A" },
        new Product { Id = 2, Name = "B" }
    ];

    return 
    [
        ..baseProducts,
        .. _includeSpecial ? [new Product { Id = 3, Name = "Special" }] : []
    ];
}
```

### 12. Avoid Repeated .ToList()

```csharp
// ❌ Multiple enumerations and allocations
var products = _repository.GetProducts().ToList();
var activeProducts = products.Where(p => p.IsActive).ToList();
var sortedProducts = activeProducts.OrderBy(p => p.Name).ToList();

// ✅ Single materialization at the end
var products = _repository
    .GetProducts()
    .Where(p => p.IsActive)
    .OrderBy(p => p.Name)
    .ToList();
```

## Symptoms

- Verbose collection initialization code
- Unnecessary heap allocations
- Collection resizing overhead
- Thread safety issues with shared collections
- Poor performance from wrong collection type

## Benefits

- **Cleaner code** with concise initialization syntax
- **Better performance** with appropriate collection types
- **Type safety** with strongly-typed collections
- **Thread safety** with immutable collections
- **Reduced allocations** with stack-allocated spans

## See Also

- [First-Class Collections](./first-class-collections.md) — Wrapping collections in domain types
- [Immutable Collections](./immutable-collections.md) — True immutability
- [Primitive Collections](./primitive-collections.md) — Avoiding primitive collection types
- [Memory Safety](./memory-safety-span.md) — Span<T> and stack allocation
