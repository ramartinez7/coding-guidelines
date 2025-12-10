# Async Patterns (Task vs ValueTask)

> Choose the right async return type—`Task<T>` for most cases, `ValueTask<T>` for hot paths with synchronous completions.

## Problem

Async methods always create `Task` objects on the heap, even when the result is immediately available. For high-throughput code, this allocation overhead adds up. `ValueTask<T>` avoids allocation when operations complete synchronously, but comes with usage constraints.

## Example

### ❌ Before (Task Allocation in Hot Path)

```csharp
public class UserRepository
{
    private readonly ConcurrentDictionary<UserId, User> cache = new();
    private readonly IDatabase database;

    // Always allocates Task, even when cached
    public async Task<User?> GetByIdAsync(UserId id)
    {
        if (cache.TryGetValue(id, out var user))
        {
            // Cache hit - but still allocated Task!
            return user;
        }

        user = await database.QueryAsync<User>("SELECT * FROM Users WHERE Id = @Id", new { Id = id });
        cache[id] = user;
        return user;
    }
}

// In tight loops, this allocates many Task objects
for (int i = 0; i < 10_000; i++)
{
    var user = await repository.GetByIdAsync(userId);  // Heap allocation each time
}
```

### ✅ After (ValueTask for Zero Allocation)

```csharp
public class UserRepository
{
    private readonly ConcurrentDictionary<UserId, User> cache = new();
    private readonly IDatabase database;

    // Zero allocation on cache hits
    public ValueTask<User?> GetByIdAsync(UserId id)
    {
        if (cache.TryGetValue(id, out var user))
        {
            // Synchronous path - no allocation!
            return new ValueTask<User?>(user);
        }

        // Asynchronous path - allocates Task
        return new ValueTask<User?>(GetFromDatabaseAsync(id));
    }

    private async Task<User?> GetFromDatabaseAsync(UserId id)
    {
        var user = await database.QueryAsync<User>("SELECT * FROM Users WHERE Id = @Id", new { Id = id });
        cache[id] = user;
        return user;
    }
}

// Same loop - zero allocations on cache hits
for (int i = 0; i < 10_000; i++)
{
    var user = await repository.GetByIdAsync(userId);  // No allocation if cached
}
```

## When to Use Each

### Use Task\<T\> (Default Choice)

✅ **Most async methods** — simpler, safer, more flexible:
```csharp
public async Task<Order> GetOrderAsync(OrderId id)
{
    return await database.QueryAsync<Order>(id);
}
```

✅ **Methods that always complete asynchronously**:
```csharp
public async Task<string> DownloadAsync(string url)
{
    using var client = new HttpClient();
    return await client.GetStringAsync(url);
}
```

✅ **Public APIs and libraries** — consumers can await multiple times:
```csharp
// Task can be awaited multiple times safely
public interface IOrderService
{
    Task<Order> GetOrderAsync(OrderId id);
}

var task = orderService.GetOrderAsync(id);
var order1 = await task;  // OK
var order2 = await task;  // OK - same result
```

### Use ValueTask\<T\> (Performance Critical)

✅ **Frequently synchronous operations** (caching, buffering):
```csharp
public ValueTask<string?> GetFromCacheAsync(string key)
{
    if (cache.TryGetValue(key, out var value))
        return new ValueTask<string?>(value);  // No allocation

    return new ValueTask<string?>(FetchAndCacheAsync(key));
}
```

✅ **Hot paths** with measurable allocation pressure:
```csharp
// High-throughput parsing
public ValueTask<ParseResult> ParseNextAsync()
{
    if (buffer.HasData)
    {
        var result = ParseFromBuffer();
        return new ValueTask<ParseResult>(result);  // Synchronous - no allocation
    }

    return new ValueTask<ParseResult>(ReadAndParseAsync());
}
```

✅ **Pooled async state machines** (advanced):
```csharp
public async ValueTask<Result> ProcessAsync(IAsyncEnumerable<Data> stream)
{
    await foreach (var item in stream)
    {
        // ValueTask reuses state machine in some cases
    }
}
```

## ValueTask Usage Rules

### ⚠️ Critical Constraints

1. **Await exactly once**:
```csharp
// ❌ BAD - awaiting ValueTask multiple times
var valueTask = GetFromCacheAsync(key);
var result1 = await valueTask;
var result2 = await valueTask;  // THROWS InvalidOperationException!

// ✅ GOOD - await once
var result = await GetFromCacheAsync(key);
```

2. **Don't store for later**:
```csharp
// ❌ BAD - storing ValueTask
private ValueTask<User> pendingUserTask;

public void StartLoad()
{
    pendingUserTask = GetUserAsync();  // DON'T DO THIS
}

// ✅ GOOD - store Task instead
private Task<User> pendingUserTask;

public void StartLoad()
{
    pendingUserTask = GetUserAsync().AsTask();
}
```

3. **Await immediately or convert to Task**:
```csharp
// ✅ Await directly
var result = await GetFromCacheAsync(key);

// ✅ Convert to Task if needed for storage
Task<string?> task = GetFromCacheAsync(key).AsTask();
```

## Performance Comparison

### Benchmark: Cache Hit Scenario

```csharp
[Benchmark]
public async Task<int> Task_CacheHit()
{
    return await GetFromCache_Task(key);
}

[Benchmark]
public async ValueTask<int> ValueTask_CacheHit()
{
    return await GetFromCache_ValueTask(key);
}

// Results (cache hit, 10,000 iterations):
// Task:      45 MB allocated, 250 ns/op
// ValueTask:  0 MB allocated, 150 ns/op
```

### When Performance Matters

Profile before optimizing. Use `ValueTask<T>` only when:

- Measured allocation pressure from `Task<T>`
- Synchronous path is common (>50% of calls)
- Method is in a hot path (called thousands of times)

## Common Patterns

### 1. Caching with ValueTask

```csharp
public class CachedRepository<T>
{
    private readonly IRepository<T> repository;
    private readonly ConcurrentDictionary<int, T> cache = new();

    public ValueTask<T?> GetByIdAsync(int id)
    {
        if (cache.TryGetValue(id, out var cached))
            return new ValueTask<T?>(cached);

        return new ValueTask<T?>(GetAndCacheAsync(id));
    }

    private async Task<T?> GetAndCacheAsync(int id)
    {
        var entity = await repository.GetByIdAsync(id);
        if (entity != null)
            cache[id] = entity;
        return entity;
    }
}
```

### 2. Buffered Reading

```csharp
public class BufferedReader
{
    private readonly Stream stream;
    private readonly byte[] buffer;
    private int position;
    private int length;

    public ValueTask<int> ReadByteAsync()
    {
        if (position < length)
        {
            // Data in buffer - synchronous
            return new ValueTask<int>(buffer[position++]);
        }

        // Buffer empty - asynchronous read
        return new ValueTask<int>(ReadByteSlowAsync());
    }

    private async Task<int> ReadByteSlowAsync()
    {
        length = await stream.ReadAsync(buffer, 0, buffer.Length);
        position = 0;

        if (length == 0)
            return -1;  // EOF

        return buffer[position++];
    }
}
```

### 3. Validation with Early Returns

```csharp
public ValueTask<Result<Order, string>> ValidateAndProcessAsync(Order order)
{
    // Synchronous validation
    if (order.Items.Count == 0)
        return new ValueTask<Result<Order, string>>(
            Result<Order, string>.Failure("Order has no items"));

    if (order.Total <= Money.Zero)
        return new ValueTask<Result<Order, string>>(
            Result<Order, string>.Failure("Order total must be positive"));

    // Async processing
    return new ValueTask<Result<Order, string>>(ProcessAsync(order));
}
```

## Async Best Practices

### 1. Avoid Async Void

```csharp
// ❌ BAD - exceptions can't be caught
public async void ProcessOrderAsync(Order order)
{
    await SaveAsync(order);
}

// ✅ GOOD - return Task
public async Task ProcessOrderAsync(Order order)
{
    await SaveAsync(order);
}

// Exception: Event handlers can be async void
button.Click += async (sender, e) => await HandleClickAsync();
```

### 2. ConfigureAwait in Libraries

```csharp
// Library code - don't capture context
public async Task<User> GetUserAsync(UserId id)
{
    var json = await httpClient.GetStringAsync(url)
        .ConfigureAwait(false);
    
    return JsonSerializer.Deserialize<User>(json);
}

// Application code - capture context is fine (it's the default)
public async Task<ActionResult> GetUser(Guid id)
{
    var user = await userService.GetUserAsync(new UserId(id));
    return View(user);
}
```

### 3. Async All the Way

```csharp
// ❌ BAD - blocking on async
public User GetUser(UserId id)
{
    return GetUserAsync(id).Result;  // DEADLOCK RISK!
}

// ✅ GOOD - async all the way
public async Task<User> GetUserAsync(UserId id)
{
    return await repository.GetByIdAsync(id);
}
```

### 4. Cancellation Support

```csharp
public async Task<Order> ProcessOrderAsync(
    Order order,
    CancellationToken cancellationToken = default)
{
    await SaveAsync(order, cancellationToken);
    
    await SendConfirmationAsync(order.Customer.Email, cancellationToken);
    
    return order;
}
```

## IAsyncEnumerable\<T\>

For streaming async data:

```csharp
public async IAsyncEnumerable<Order> GetOrdersAsync(
    [EnumeratorCancellation] CancellationToken cancellationToken = default)
{
    await using var connection = await database.OpenConnectionAsync(cancellationToken);
    await using var reader = await connection.ExecuteReaderAsync(
        "SELECT * FROM Orders", cancellationToken);

    while (await reader.ReadAsync(cancellationToken))
    {
        yield return MapToOrder(reader);
    }
}

// Usage
await foreach (var order in orderRepository.GetOrdersAsync(cancellationToken))
{
    await ProcessAsync(order);
}
```

## See Also

- [CancellationToken Handling](./cancellation-token-handling.md) — proper cancellation support
- [Lazy Initialization](./lazy-initialization.md) — deferred computation patterns
- [Allocation Budget](./allocation-budget.md) — zero-allocation patterns
- [Memory Safety (Span)](./memory-safety-span.md) — zero-allocation string operations
