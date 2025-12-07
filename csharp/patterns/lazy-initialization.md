# Lazy Initialization (Type-Safe Deferred Computation)

> Expensive initialization done eagerly when it might not be needed—use lazy initialization with type safety to defer computation until actually required.

## Problem

Creating objects that are expensive to initialize (database connections, large data structures, computed values) but might not be used wastes resources. Traditional lazy initialization patterns are error-prone and lack type safety.

## Example

### ❌ Before

```csharp
public class ReportGenerator
{
    // Computed eagerly, even if never used
    private readonly Dictionary<string, List<DataPoint>> _cache;
    
    public ReportGenerator()
    {
        // Expensive: loads and processes all historical data
        _cache = LoadHistoricalData();  // Takes 5 seconds, always runs
    }
    
    public Report GenerateReport(string category)
    {
        // Only 10% of instances actually use the cache
        if (_cache.ContainsKey(category))
            return CreateFromCache(_cache[category]);
        
        return CreateFreshReport(category);
    }
    
    private Dictionary<string, List<DataPoint>> LoadHistoricalData()
    {
        // Expensive I/O and computation
        Thread.Sleep(5000);  // Simulating slow operation
        return new Dictionary<string, List<DataPoint>>();
    }
}

// Problem: Every instance pays the 5-second cost, even if never calling GenerateReport
var generator = new ReportGenerator();  // 5 seconds wasted if we don't use it
```

**Problems:**
- Initialization always happens, even if not needed
- No thread safety for lazy initialization
- Nullable pattern is error-prone
- Initialization code mixed with usage code

### ✅ After

```csharp
public class ReportGenerator
{
    // Thread-safe lazy initialization
    private readonly Lazy<Dictionary<string, List<DataPoint>>> _cache;
    
    public ReportGenerator()
    {
        // Defers computation until first access
        _cache = new Lazy<Dictionary<string, List<DataPoint>>>(
            LoadHistoricalData,
            LazyThreadSafetyMode.ExecutionAndPublication);
        
        // Constructor returns instantly
    }
    
    public Report GenerateReport(string category)
    {
        // Computed only on first access, then cached
        var cache = _cache.Value;
        
        if (cache.ContainsKey(category))
            return CreateFromCache(cache[category]);
        
        return CreateFreshReport(category);
    }
    
    private Dictionary<string, List<DataPoint>> LoadHistoricalData()
    {
        // Expensive I/O and computation
        Thread.Sleep(5000);
        return new Dictionary<string, List<DataPoint>>();
    }
}

// Instance created instantly; expensive work only happens when needed
var generator = new ReportGenerator();  // Instant
var report = generator.GenerateReport("sales");  // First call pays the cost
```

## Lazy with Custom Factory

```csharp
public class DatabaseConnection
{
    // Lazy connection with dependency injection
    private readonly Lazy<IDbConnection> _connection;
    
    public DatabaseConnection(string connectionString)
    {
        _connection = new Lazy<IDbConnection>(() =>
        {
            var connection = new SqlConnection(connectionString);
            connection.Open();  // Expensive operation deferred
            return connection;
        });
    }
    
    public IDbConnection Connection => _connection.Value;
    
    // Check if initialized without triggering initialization
    public bool IsConnected => _connection.IsValueCreated && _connection.Value.State == ConnectionState.Open;
}
```

## Lazy with Validation

```csharp
/// <summary>
/// Lazy value that validates on initialization.
/// </summary>
public class ValidatedLazy<T>
{
    private readonly Lazy<Result<T, string>> _lazy;
    
    public ValidatedLazy(Func<Result<T, string>> factory)
    {
        _lazy = new Lazy<Result<T, string>>(factory);
    }
    
    public Result<T, string> Value => _lazy.Value;
    
    public bool IsValueCreated => _lazy.IsValueCreated;
}

public class ConfigurationService
{
    private readonly ValidatedLazy<AppConfig> _config;
    
    public ConfigurationService(IConfiguration configuration)
    {
        _config = new ValidatedLazy<AppConfig>(() =>
        {
            var config = new AppConfig();
            configuration.Bind(config);
            
            // Validate
            if (string.IsNullOrEmpty(config.ApiKey))
                return Result<AppConfig, string>.Failure("ApiKey is required");
            
            if (config.MaxConnections <= 0)
                return Result<AppConfig, string>.Failure("MaxConnections must be positive");
            
            return Result<AppConfig, string>.Success(config);
        });
    }
    
    public Result<AppConfig, string> Config => _config.Value;
}
```

## Async Lazy

For asynchronous initialization:

```csharp
public class AsyncLazy<T>
{
    private readonly Lazy<Task<T>> _lazy;
    
    public AsyncLazy(Func<Task<T>> factory)
    {
        _lazy = new Lazy<Task<T>>(() => Task.Run(factory));
    }
    
    public Task<T> Value => _lazy.Value;
    
    public bool IsValueCreated => _lazy.IsValueCreated;
}

public class DataService
{
    private readonly AsyncLazy<List<Customer>> _customers;
    
    public DataService(ICustomerRepository repository)
    {
        _customers = new AsyncLazy<List<Customer>>(async () =>
        {
            // Expensive async operation
            var customers = await repository.GetAllAsync();
            return customers.OrderBy(c => c.Name).ToList();
        });
    }
    
    public async Task<List<Customer>> GetCustomersAsync()
    {
        return await _customers.Value;
    }
}
```

## Lazy with Expiration

```csharp
public class ExpiringLazy<T>
{
    private readonly Func<T> _factory;
    private readonly TimeSpan _expiration;
    private readonly object _lock = new();
    
    private T? _value;
    private DateTime _expiresAt;
    
    public ExpiringLazy(Func<T> factory, TimeSpan expiration)
    {
        _factory = factory;
        _expiration = expiration;
        _expiresAt = DateTime.MinValue;
    }
    
    public T Value
    {
        get
        {
            lock (_lock)
            {
                if (DateTime.UtcNow >= _expiresAt)
                {
                    _value = _factory();
                    _expiresAt = DateTime.UtcNow + _expiration;
                }
                
                return _value!;
            }
        }
    }
    
    public void Invalidate()
    {
        lock (_lock)
        {
            _expiresAt = DateTime.MinValue;
        }
    }
}

public class CacheService
{
    private readonly ExpiringLazy<List<Product>> _featuredProducts;
    
    public CacheService(IProductRepository repository)
    {
        // Cache expires after 5 minutes
        _featuredProducts = new ExpiringLazy<List<Product>>(
            () => repository.GetFeaturedProducts(),
            TimeSpan.FromMinutes(5));
    }
    
    public List<Product> FeaturedProducts => _featuredProducts.Value;
    
    public void RefreshCache()
    {
        _featuredProducts.Invalidate();
    }
}
```

## Lazy Collection Pattern

```csharp
public class OrderDetails
{
    private readonly OrderId _orderId;
    private readonly IOrderRepository _repository;
    
    // Lazy-loaded related data
    private readonly Lazy<List<OrderItem>> _items;
    private readonly Lazy<Customer> _customer;
    private readonly Lazy<List<OrderEvent>> _history;
    
    public OrderDetails(OrderId orderId, IOrderRepository repository)
    {
        _orderId = orderId;
        _repository = repository;
        
        // Each collection loaded only when accessed
        _items = new Lazy<List<OrderItem>>(() => _repository.GetItems(orderId));
        _customer = new Lazy<Customer>(() => _repository.GetCustomer(orderId));
        _history = new Lazy<List<OrderEvent>>(() => _repository.GetHistory(orderId));
    }
    
    public List<OrderItem> Items => _items.Value;
    public Customer Customer => _customer.Value;
    public List<OrderEvent> History => _history.Value;
}

// Usage: Only loads what you actually use
var order = new OrderDetails(orderId, repository);
Console.WriteLine(order.Customer.Name);  // Only customer loaded, not items or history
```

## Lazy State Machine

```csharp
/// <summary>
/// Represents a value that transitions through states: NotStarted → Computing → Computed.
/// </summary>
public class LazyState<T>
{
    private readonly Func<T> _factory;
    private readonly object _lock = new();
    private ComputationState _state;
    private T? _value;
    private Exception? _error;
    
    public LazyState(Func<T> factory)
    {
        _factory = factory;
        _state = ComputationState.NotStarted;
    }
    
    public ComputationState State
    {
        get
        {
            lock (_lock)
            {
                return _state;
            }
        }
    }
    
    public T Value
    {
        get
        {
            lock (_lock)
            {
                if (_state == ComputationState.Computed)
                    return _value!;
                
                if (_state == ComputationState.Failed)
                    throw new InvalidOperationException("Computation failed", _error);
                
                if (_state == ComputationState.Computing)
                    throw new InvalidOperationException("Recursive lazy initialization detected");
                
                _state = ComputationState.Computing;
            }
            
            try
            {
                T result = _factory();
                
                lock (_lock)
                {
                    _value = result;
                    _state = ComputationState.Computed;
                    return _value;
                }
            }
            catch (Exception ex)
            {
                lock (_lock)
                {
                    _error = ex;
                    _state = ComputationState.Failed;
                    throw;
                }
            }
        }
    }
}

public enum ComputationState
{
    NotStarted,
    Computing,
    Computed,
    Failed
}
```

## Dependency Injection Integration

```csharp
// Register lazy dependencies
services.AddTransient<Lazy<IExpensiveService>>(provider =>
    new Lazy<IExpensiveService>(() => provider.GetRequiredService<IExpensiveService>()));

// Usage
public class ConsumerService
{
    private readonly Lazy<IExpensiveService> _expensiveService;
    
    public ConsumerService(Lazy<IExpensiveService> expensiveService)
    {
        _expensiveService = expensiveService;
    }
    
    public void DoWork()
    {
        if (NeedExpensiveOperation())
        {
            // Service only created when actually needed
            _expensiveService.Value.PerformOperation();
        }
    }
}
```

## Testing

```csharp
public class LazyServiceTests
{
    [Fact]
    public void Value_NotAccessedUntilNeeded()
    {
        var factoryCalled = false;
        var lazy = new Lazy<int>(() =>
        {
            factoryCalled = true;
            return 42;
        });
        
        // Factory not called yet
        Assert.False(lazy.IsValueCreated);
        Assert.False(factoryCalled);
        
        // Access triggers computation
        var value = lazy.Value;
        
        Assert.True(lazy.IsValueCreated);
        Assert.True(factoryCalled);
        Assert.Equal(42, value);
    }
    
    [Fact]
    public void Value_OnlyComputedOnce()
    {
        var callCount = 0;
        var lazy = new Lazy<int>(() =>
        {
            callCount++;
            return 42;
        });
        
        var value1 = lazy.Value;
        var value2 = lazy.Value;
        var value3 = lazy.Value;
        
        Assert.Equal(1, callCount);  // Factory called only once
        Assert.Equal(42, value1);
        Assert.Equal(42, value2);
        Assert.Equal(42, value3);
    }
}
```

## Why It's a Problem

1. **Wasted resources**: Expensive initialization happens even if never used
2. **Slower startup**: Every object pays initialization cost upfront
3. **Manual thread safety**: Hand-rolled lazy patterns often have race conditions
4. **Nullable pollution**: Using `null` to detect initialization spreads nullable checks

## Symptoms

- Constructor takes a long time to complete
- Many objects created but never fully used
- `if (_field == null)` checks before using fields
- Thread safety bugs in lazy initialization code
- Memory usage higher than expected

## Benefits

- **Deferred cost**: Expensive work only happens when needed
- **Fast construction**: Objects created instantly
- **Thread-safe**: `Lazy<T>` handles concurrency correctly
- **Type-safe**: No nullable checks needed
- **One-time guarantee**: Computation only happens once

## Trade-offs

- **First-access latency**: First call pays the initialization cost
- **Memory overhead**: `Lazy<T>` wrapper adds small overhead
- **Debugging complexity**: Initialization happens at unexpected times
- **Dependency lifetime**: Must consider when lazy values are disposed

## See Also

- [Validated Configuration](./validated-configuration.md) — startup validation
- [Static Factory Methods](./static-factory-methods.md) — controlled construction
- [Value Semantics](./value-semantics.md) — when to compute eagerly
- [Honest Functions](./honest-functions.md) — explicit initialization
