# Task-Based Asynchronous Pattern (TAP)

> Follow TAP conventions for consistent, composable async code using async/await and Task/ValueTask.

## Problem

Inconsistent async patterns make code hard to compose, understand, and can lead to deadlocks or poor performance.

## Example

### ❌ Before

```csharp
// Mixing old and new patterns
public class DataService
{
    // APM (Asynchronous Programming Model) - outdated
    public IAsyncResult BeginGetData(AsyncCallback callback, object state) { }
    public Data EndGetData(IAsyncResult result) { }

    // EAP (Event-based Asynchronous Pattern) - outdated
    public event EventHandler<DataReceivedEventArgs> DataReceived;
    public void GetDataAsync() { }
}
```

### ✅ After

```csharp
// TAP (Task-based Asynchronous Pattern)
public class DataService
{
    public async Task<Data> GetDataAsync(
        CancellationToken cancellationToken = default)
    {
        var data = await FetchDataAsync(cancellationToken);
        return data;
    }
}
```

## Best Practices

### 1. Use 'Async' Suffix

```csharp
// ✅ Async suffix for async methods
public async Task<Order> GetOrderAsync(OrderId id) { }

public async Task ProcessOrderAsync(Order order) { }

public async ValueTask<bool> TryConnectAsync() { }
```

### 2. Accept CancellationToken

```csharp
// ✅ CancellationToken as last parameter with default
public async Task<List<Order>> GetOrdersAsync(
    CustomerId customerId,
    CancellationToken cancellationToken = default)
{
    cancellationToken.ThrowIfCancellationRequested();

    return await _repository
        .GetOrdersAsync(customerId, cancellationToken);
}
```

### 3. Return Task or ValueTask

```csharp
// ✅ Task<T> for most cases
public async Task<Order> GetOrderAsync(OrderId id)
{
    return await _repository.GetOrderAsync(id);
}

// ✅ ValueTask<T> for hot path with sync completion
public ValueTask<Order?> GetCachedOrderAsync(OrderId id)
{
    if (_cache.TryGetValue(id, out var order))
    {
        return new ValueTask<Order?>(order);  // Sync completion
    }

    return new ValueTask<Order?>(FetchAndCacheAsync(id));  // Async path
}

// ✅ Task for void return
public async Task ProcessOrderAsync(Order order)
{
    await _processor.ProcessAsync(order);
}
```

### 4. Don't Block on Async Code

```csharp
// ❌ Blocking on async code (deadlock risk)
public Order GetOrder(OrderId id)
{
    return GetOrderAsync(id).Result;  // DEADLOCK!
}

public void ProcessOrder(Order order)
{
    ProcessOrderAsync(order).Wait();  // DEADLOCK!
}

// ✅ Async all the way
public async Task<Order> GetOrderAsync(OrderId id)
{
    return await _repository.GetOrderAsync(id);
}

public async Task ProcessOrderAsync(Order order)
{
    await _processor.ProcessAsync(order);
}
```

### 5. Avoid Async Void (Except Event Handlers)

```csharp
// ❌ Async void (can't catch exceptions)
public async void ProcessOrderAsync(OrderId id)
{
    await _processor.ProcessAsync(id);  // Exception lost!
}

// ✅ Return Task
public async Task ProcessOrderAsync(OrderId id)
{
    await _processor.ProcessAsync(id);
}

// ✅ Only for event handlers
private async void Button_Click(object sender, EventArgs e)
{
    try
    {
        await ProcessOrderAsync(orderId);
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Failed to process order");
        ShowError(ex.Message);
    }
}
```

### 6. ConfigureAwait in Libraries

```csharp
// ✅ ConfigureAwait(false) in library code
public async Task<User> GetUserAsync(UserId id)
{
    var user = await _repository
        .GetUserAsync(id)
        .ConfigureAwait(false);

    var enriched = await EnrichAsync(user)
        .ConfigureAwait(false);

    return enriched;
}

// In application code (UI, ASP.NET Core), don't use ConfigureAwait
```

### 7. Use TaskCompletionSource for Wrapping

```csharp
// ✅ Wrap callback-based API with TaskCompletionSource
public Task<Data> GetDataAsync()
{
    var tcs = new TaskCompletionSource<Data>();

    _legacyApi.GetData(data =>
    {
        tcs.SetResult(data);
    }, error =>
    {
        tcs.SetException(error);
    });

    return tcs.Task;
}
```

### 8. Parallelize Independent Operations

```csharp
// ❌ Sequential (slow)
public async Task<OrderSummary> GetOrderSummaryAsync(OrderId id)
{
    var order = await GetOrderAsync(id);
    var customer = await GetCustomerAsync(order.CustomerId);
    var products = await GetProductsAsync(order.ProductIds);

    return new OrderSummary(order, customer, products);
}

// ✅ Parallel (fast)
public async Task<OrderSummary> GetOrderSummaryAsync(OrderId id)
{
    var order = await GetOrderAsync(id);

    var customerTask = GetCustomerAsync(order.CustomerId);
    var productsTask = GetProductsAsync(order.ProductIds);

    await Task.WhenAll(customerTask, productsTask);

    return new OrderSummary(
        order,
        await customerTask,
        await productsTask);
}
```

### 9. Report Progress

```csharp
// ✅ IProgress<T> for progress reporting
public async Task ProcessBatchAsync(
    List<Order> orders,
    IProgress<int>? progress = null,
    CancellationToken cancellationToken = default)
{
    for (int i = 0; i < orders.Count; i++)
    {
        cancellationToken.ThrowIfCancellationRequested();

        await ProcessOrderAsync(orders[i], cancellationToken);

        progress?.Report((i + 1) * 100 / orders.Count);
    }
}

// Usage
var progress = new Progress<int>(percent =>
{
    Console.WriteLine($"{percent}% complete");
});

await ProcessBatchAsync(orders, progress);
```

### 10. Use Task.WhenAll for Aggregation

```csharp
// ✅ Process all, collect results
public async Task<List<Order>> GetOrdersAsync(
    List<OrderId> orderIds,
    CancellationToken cancellationToken = default)
{
    var tasks = orderIds
        .Select(id => GetOrderAsync(id, cancellationToken))
        .ToList();

    var orders = await Task.WhenAll(tasks);

    return orders.ToList();
}
```

### 11. Handle Exceptions in Async

```csharp
// ✅ Proper exception handling
public async Task<Result<Order, OrderError>> ProcessOrderAsync(
    OrderId id,
    CancellationToken cancellationToken = default)
{
    try
    {
        var order = await GetOrderAsync(id, cancellationToken);

        await ProcessPaymentAsync(order, cancellationToken);

        return Result.Success<Order, OrderError>(order);
    }
    catch (PaymentException ex)
    {
        _logger.LogError(ex, "Payment failed for order {OrderId}", id);
        return Result.Failure<Order, OrderError>(
            new PaymentFailedError(ex.Message));
    }
    catch (OperationCanceledException)
    {
        _logger.LogInformation("Order processing cancelled for {OrderId}", id);
        throw;
    }
}
```

### 12. Use AsyncLocal for Context

```csharp
// ✅ AsyncLocal for async context
public static class CorrelationContext
{
    private static readonly AsyncLocal<string?> CorrelationId = new();

    public static string? CurrentCorrelationId
    {
        get => CorrelationId.Value;
        set => CorrelationId.Value = value;
    }
}

public async Task ProcessRequestAsync()
{
    CorrelationContext.CurrentCorrelationId = Guid.NewGuid().ToString();

    await ProcessAsync();  // CorrelationId flows to nested calls
}
```

## Symptoms

- Deadlocks in async code
- Unhandled exceptions in async methods
- Poor performance from sequential operations
- Difficulty cancelling long-running operations
- Context loss in async flows

## Benefits

- **Composable async code** with consistent patterns
- **Better performance** with proper parallelization
- **Cancellation support** built-in
- **Exception handling** that works
- **Progress reporting** for long operations

## See Also

- [Async Patterns](./async-patterns.md) — Task vs ValueTask
- [Async/Await Anti-Patterns](./async-await-anti-patterns.md) — Common mistakes
- [CancellationToken Handling](./cancellation-token-handling.md) — Cancellation
