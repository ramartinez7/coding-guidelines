# Const vs ReadOnly vs Static ReadOnly

> Choose the right immutability mechanism for constants, runtime constants, and shared state.

## Problem

Using the wrong type of constant leads to versioning issues, unnecessary memory allocations, or mutable state.

## Example

### ❌ Before

```csharp
public class OrderService
{
    public const string ApiUrl = "https://api.example.com";  // Compiled into calling code!
    public static decimal TaxRate = 0.08m;  // Mutable!
    public readonly DateTime StartTime = DateTime.Now;  // Different per instance!
}
```

### ✅ After

```csharp
public class OrderService
{
    public static readonly string ApiUrl = "https://api.example.com";  // Can change without recompile
    public static readonly decimal DefaultTaxRate = 0.08m;  // Immutable
    private static readonly DateTime StartTime = DateTime.UtcNow;  // Shared across instances
    private readonly IOrderRepository repository;  // Instance-specific dependency
}
```

## Best Practices

### 1. Use const for Compile-Time Constants

```csharp
// ✅ const for true compile-time constants
public class MathConstants
{
    public const double Pi = 3.14159265359;
    public const int MaxItems = 100;
    public const string Version = "1.0.0";
}

// ⚠️ Compiled into calling code - changes require recompilation!
```

### 2. Use static readonly for Runtime Constants

```csharp
// ✅ static readonly for values known at runtime
public class Configuration
{
    public static readonly string ApplicationPath =
        Path.GetDirectoryName(Assembly.GetExecutingAssembly().Location)!;

    public static readonly Guid ApplicationId = Guid.NewGuid();

    public static readonly DateTime ApplicationStartTime = DateTime.UtcNow;
}
```

### 3. Use readonly for Instance Constants

```csharp
// ✅ readonly for instance-specific immutable values
public class OrderService
{
    private readonly IOrderRepository repository;
    private readonly ILogger<OrderService> logger;
    private readonly Guid instanceId;

    public OrderService(
        IOrderRepository repository,
        ILogger<OrderService> logger)
    {
        this.repository = repository;
        this.logger = logger;
        this.instanceId = Guid.NewGuid();  // Set once in constructor
    }
}
```

### 4. const vs static readonly for Reference Types

```csharp
// ❌ Can't use const for reference types (except string and null)
public const DateTime InvalidDate = DateTime.MinValue;  // Compiler error!

// ✅ Use static readonly for reference types
public static readonly DateTime Epoch = new DateTime(1970, 1, 1);
public static readonly TimeSpan DefaultTimeout = TimeSpan.FromSeconds(30);
```

### 5. Use static readonly for Collections

```csharp
// ✅ static readonly for immutable collections
public class OrderStatus
{
    public static readonly IReadOnlyList<string> ValidStatuses = new[]
    {
        "Pending",
        "Confirmed",
        "Shipped",
        "Delivered"
    };

    // ✅ Or immutable collection
    public static readonly ImmutableList<string> AllStatuses =
        ImmutableList.Create("Pending", "Confirmed", "Shipped", "Delivered");
}
```

### 6. Prefer static readonly Over const for Flexibility

```csharp
// ❌ const compiled into calling assemblies
// Library v1
public class Config
{
    public const int MaxRetries = 3;
}

// Client code
var retries = Config.MaxRetries;  // Value "3" compiled into client!

// Library v2 - changing const requires client recompilation
public const int MaxRetries = 5;  // Client still uses 3!

// ✅ static readonly allows changes without recompilation
public static readonly int MaxRetries = 3;  // Can update library only
```

### 7. Use const for Primitive Literal Values

```csharp
// ✅ const appropriate for these
public class Limits
{
    public const int MaxNameLength = 100;
    public const decimal MinimumOrderValue = 10.00m;
    public const bool EnableFeatureX = true;
    public const string DefaultCurrency = "USD";
}
```

### 8. Initialize readonly in Constructor

```csharp
// ✅ readonly initialized in constructor
public class OrderProcessor
{
    private readonly int maxBatchSize;
    private readonly TimeSpan timeout;

    public OrderProcessor(int maxBatchSize, TimeSpan timeout)
    {
        // Can only assign in constructor
        this.maxBatchSize = maxBatchSize;
        this.timeout = timeout;
    }

    public void Process()
    {
        // this.maxBatchSize = 20;  // Compiler error!
    }
}
```

### 9. Use init for Readonly Properties

```csharp
// ✅ init-only properties (C# 9+)
public class Configuration
{
    public string ApiUrl { get; init; } = "https://api.example.com";
    public int MaxRetries { get; init; } = 3;
    public TimeSpan Timeout { get; init; } = TimeSpan.FromSeconds(30);
}

// Can set during initialization only
var config = new Configuration
{
    ApiUrl = "https://custom.api.com",
    MaxRetries = 5
};

// config.ApiUrl = "new";  // Compiler error!
```

### 10. Static readonly for Shared Resources

```csharp
// ✅ Static readonly for expensive shared resources
public class DatabaseConfig
{
    private static readonly Lazy<DbConnection> LazyConnection =
        new Lazy<DbConnection>(() => CreateConnection());

    public static DbConnection SharedConnection => LazyConnection.Value;

    private static DbConnection CreateConnection()
    {
        // Expensive initialization
        return new DbConnection("ConnectionString");
    }
}
```

### 11. Avoid Mutable static Fields

```csharp
// ❌ Mutable static field (shared mutable state!)
public static int Counter = 0;  // Thread safety issues!

public static List<Order> AllOrders = new();  // Mutable!

// ✅ Make static fields readonly
public static readonly object SyncLock = new();
private static int counter = 0;

public static int IncrementCounter()
{
    lock (SyncLock)
    {
        return ++counter;
    }
}
```

### 12. Document const vs readonly Choice

```csharp
// ✅ Document why const vs readonly
public class ApiConfig
{
    // const: True compile-time constant, never changes
    public const string ApiVersion = "v1";

    // static readonly: May change in future versions
    // without requiring client recompilation
    public static readonly string BaseUrl = "https://api.example.com";

    // readonly: Instance-specific configuration
    private readonly IHttpClientFactory clientFactory;
}
```

## Comparison Table

| Feature | const | static readonly | readonly |
|---------|-------|----------------|----------|
| **When set** | Compile-time | Runtime (static ctor) | Runtime (instance ctor) |
| **Scope** | Type-level | Type-level | Instance-level |
| **Value types** | Yes | Yes | Yes |
| **Reference types** | string, null only | Yes | Yes |
| **Compiled into caller** | Yes | No | No |
| **Can change without recompile** | No | Yes | Yes |
| **Memory** | No memory | One copy | One per instance |

## Symptoms

- Version mismatch bugs with const changes
- Shared mutable state with static fields
- Different values per instance when shared needed
- Unnecessary memory with instance constants

## Benefits

- **Immutability** prevents accidental changes
- **Flexibility** with static readonly
- **Type safety** with compile-time checks
- **Performance** with appropriate choice
- **Clear intent** about mutability

## See Also

- [Immutable Collections](./immutable-collections.md) — Immutable data structures
- [Value Semantics](./value-semantics.md) — Immutable value types
- [Snapshot Immutability](./snapshot-immutability.md) — True vs apparent immutability
