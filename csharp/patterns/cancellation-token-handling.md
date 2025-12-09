# CancellationToken Handling

> Support cancellation in async operations—respect timeouts, user cancellations, and resource cleanup.

## Problem

Long-running async operations that can't be cancelled waste resources and create poor user experiences. Cancellation tokens provide a standard way to signal and respond to cancellation requests.

## Example

### ❌ Before

```csharp
public class OrderProcessor
{
    // No way to cancel this operation
    public async Task ProcessBatchAsync(List<Order> orders)
    {
        foreach (var order in orders)
        {
            // If this takes hours, user can't stop it
            await ProcessOrderAsync(order);
            await SendNotificationAsync(order);
            await UpdateInventoryAsync(order);
        }
    }
}

// User closes app or times out - operation continues
```

### ✅ After

```csharp
public class OrderProcessor
{
    public async Task ProcessBatchAsync(
        List<Order> orders,
        CancellationToken cancellationToken = default)
    {
        foreach (var order in orders)
        {
            // Check for cancellation before expensive operations
            cancellationToken.ThrowIfCancellationRequested();
            
            await ProcessOrderAsync(order, cancellationToken);
            await SendNotificationAsync(order, cancellationToken);
            await UpdateInventoryAsync(order, cancellationToken);
        }
    }
}

// Usage with timeout
using var cts = new CancellationTokenSource(TimeSpan.FromMinutes(5));
try
{
    await processor.ProcessBatchAsync(orders, cts.Token);
}
catch (OperationCanceledException)
{
    // Operation timed out or was cancelled
    logger.LogWarning("Batch processing cancelled");
}
```

## Why It's Important

1. **Responsiveness**: Users can cancel long-running operations.

2. **Resource efficiency**: Stop wasting CPU/memory on cancelled work.

3. **Timeouts**: Prevent operations from running indefinitely.

4. **Graceful shutdown**: Clean up resources when app closes.

5. **Coordination**: Cancel related operations together.

## Basic Usage

### 1. Accept CancellationToken Parameter

```csharp
// Add CancellationToken as last parameter with default
public async Task<User> GetUserAsync(
    UserId id,
    CancellationToken cancellationToken = default)
{
    var user = await database.QueryAsync<User>(
        "SELECT * FROM Users WHERE Id = @Id",
        new { Id = id },
        cancellationToken);
    
    return user;
}
```

### 2. Pass Token to Async Methods

```csharp
public async Task ProcessAsync(Order order, CancellationToken cancellationToken)
{
    // Pass token through the call chain
    await ValidateAsync(order, cancellationToken);
    await SaveAsync(order, cancellationToken);
    await PublishEventAsync(order, cancellationToken);
}
```

### 3. Check Token in Loops

```csharp
public async Task ProcessItemsAsync(
    List<Item> items,
    CancellationToken cancellationToken)
{
    foreach (var item in items)
    {
        // Check before expensive work
        cancellationToken.ThrowIfCancellationRequested();
        
        await ProcessItemAsync(item, cancellationToken);
    }
}
```

## Creating Cancellation Tokens

### 1. Manual Cancellation

```csharp
var cts = new CancellationTokenSource();

// Start operation
var task = ProcessAsync(orders, cts.Token);

// Cancel from another thread
if (userClickedCancel)
{
    cts.Cancel();
}

try
{
    await task;
}
catch (OperationCanceledException)
{
    Console.WriteLine("Operation cancelled by user");
}
finally
{
    cts.Dispose();
}
```

### 2. Timeout

```csharp
// Cancel after 30 seconds
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(30));

try
{
    await DownloadAsync(url, cts.Token);
}
catch (OperationCanceledException)
{
    Console.WriteLine("Download timed out");
}
```

### 3. Linked Tokens

Combine multiple cancellation sources:

```csharp
public async Task ProcessWithTimeoutAsync(
    CancellationToken externalToken)
{
    // Combine external token with internal timeout
    using var timeoutCts = new CancellationTokenSource(TimeSpan.FromMinutes(5));
    using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
        externalToken,
        timeoutCts.Token);
    
    // Cancelled if either token fires
    await ProcessAsync(linkedCts.Token);
}
```

## Cancellation Patterns

### 1. Check Before Expensive Operations

```csharp
public async Task ProcessLargeFileAsync(
    string filePath,
    CancellationToken cancellationToken)
{
    cancellationToken.ThrowIfCancellationRequested();
    
    var data = await File.ReadAllBytesAsync(filePath, cancellationToken);
    
    cancellationToken.ThrowIfCancellationRequested();
    
    var processed = ProcessData(data);
    
    cancellationToken.ThrowIfCancellationRequested();
    
    await File.WriteAllBytesAsync(outputPath, processed, cancellationToken);
}
```

### 2. Cooperative Cancellation in CPU-Bound Work

```csharp
public Task<List<int>> FindPrimesAsync(
    int max,
    CancellationToken cancellationToken)
{
    return Task.Run(() =>
    {
        var primes = new List<int>();
        
        for (int i = 2; i < max; i++)
        {
            // Check periodically in CPU-bound loops
            if (i % 1000 == 0)
                cancellationToken.ThrowIfCancellationRequested();
            
            if (IsPrime(i))
                primes.Add(i);
        }
        
        return primes;
    }, cancellationToken);
}
```

### 3. Register Cleanup Callbacks

```csharp
public async Task DownloadWithCleanupAsync(
    string url,
    string tempFile,
    CancellationToken cancellationToken)
{
    // Register cleanup that runs on cancellation
    using var registration = cancellationToken.Register(() =>
    {
        if (File.Exists(tempFile))
        {
            File.Delete(tempFile);
            Console.WriteLine("Cleaned up temp file on cancellation");
        }
    });
    
    await DownloadToFileAsync(url, tempFile, cancellationToken);
}
```

### 4. Timeout with Fallback

```csharp
public async Task<Result<Data, string>> GetDataWithFallbackAsync(
    CancellationToken cancellationToken)
{
    using var timeoutCts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
    using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
        cancellationToken,
        timeoutCts.Token);
    
    try
    {
        var data = await GetFromPrimarySourceAsync(linkedCts.Token);
        return Result<Data, string>.Success(data);
    }
    catch (OperationCanceledException ex) when (ex.CancellationToken == timeoutCts.Token)
    {
        // Timeout - try fallback
        var data = await GetFromCacheAsync(cancellationToken);
        return Result<Data, string>.Success(data);
    }
}
```

## Advanced Patterns

### 1. Cancellable Task.WhenAll

```csharp
public async Task ProcessInParallelAsync(
    List<Order> orders,
    CancellationToken cancellationToken)
{
    var tasks = orders.Select(order => 
        ProcessOrderAsync(order, cancellationToken));
    
    try
    {
        await Task.WhenAll(tasks);
    }
    catch (OperationCanceledException)
    {
        // At least one task was cancelled
        throw;
    }
}
```

### 2. Cancellable Retry Logic

```csharp
public async Task<T> RetryAsync<T>(
    Func<CancellationToken, Task<T>> operation,
    int maxAttempts,
    CancellationToken cancellationToken)
{
    for (int attempt = 1; attempt <= maxAttempts; attempt++)
    {
        try
        {
            cancellationToken.ThrowIfCancellationRequested();
            return await operation(cancellationToken);
        }
        catch (Exception) when (attempt < maxAttempts)
        {
            // Wait before retry, but respect cancellation
            await Task.Delay(
                TimeSpan.FromSeconds(Math.Pow(2, attempt)),
                cancellationToken);
        }
    }
    
    throw new InvalidOperationException("Max retries exceeded");
}
```

### 3. IAsyncEnumerable with Cancellation

```csharp
public async IAsyncEnumerable<Order> StreamOrdersAsync(
    [EnumeratorCancellation] CancellationToken cancellationToken = default)
{
    await using var connection = await database.OpenConnectionAsync(cancellationToken);
    await using var reader = await connection.ExecuteReaderAsync(
        "SELECT * FROM Orders",
        cancellationToken);
    
    while (await reader.ReadAsync(cancellationToken))
    {
        yield return MapToOrder(reader);
    }
}

// Usage
await foreach (var order in repository.StreamOrdersAsync(cancellationToken))
{
    await ProcessAsync(order, cancellationToken);
}
```

## Best Practices

### 1. Always Accept CancellationToken

Make it the last parameter with a default value:

```csharp
// ✅ Good
public async Task ProcessAsync(
    Order order,
    CancellationToken cancellationToken = default)

// ❌ Bad - missing token
public async Task ProcessAsync(Order order)
```

### 2. Don't Swallow OperationCanceledException

```csharp
// ❌ Bad - hides cancellation
try
{
    await ProcessAsync(cancellationToken);
}
catch (OperationCanceledException)
{
    // Silently ignore - bad!
}

// ✅ Good - let it propagate or log it
try
{
    await ProcessAsync(cancellationToken);
}
catch (OperationCanceledException)
{
    logger.LogInformation("Operation cancelled");
    throw;  // Propagate
}
```

### 3. Dispose CancellationTokenSource

```csharp
// ✅ Good - using statement
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(30));
await ProcessAsync(cts.Token);

// ✅ Good - explicit dispose
var cts = new CancellationTokenSource();
try
{
    await ProcessAsync(cts.Token);
}
finally
{
    cts.Dispose();
}
```

### 4. Check Token, Don't Poll Properties

```csharp
// ❌ Bad - polling
while (!cancellationToken.IsCancellationRequested)
{
    await ProcessItemAsync();
}

// ✅ Good - throw on cancellation
while (true)
{
    cancellationToken.ThrowIfCancellationRequested();
    await ProcessItemAsync();
}
```

### 5. Timeout vs Cancellation

Distinguish between timeout and external cancellation:

```csharp
public async Task<Result<Data, string>> GetDataAsync(
    CancellationToken cancellationToken)
{
    using var timeoutCts = new CancellationTokenSource(TimeSpan.FromSeconds(10));
    using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
        cancellationToken,
        timeoutCts.Token);
    
    try
    {
        return await FetchDataAsync(linkedCts.Token);
    }
    catch (OperationCanceledException ex) when (ex.CancellationToken == timeoutCts.Token)
    {
        return Result<Data, string>.Failure("Operation timed out");
    }
    catch (OperationCanceledException)
    {
        throw;  // External cancellation - propagate
    }
}
```

## Common Mistakes

### ❌ Not Passing Token Through

```csharp
public async Task ProcessAsync(CancellationToken cancellationToken)
{
    // Forgot to pass token!
    await SaveAsync();  // Can't be cancelled
    await SendEmailAsync();  // Can't be cancelled
}
```

### ❌ Blocking on Cancelled Task

```csharp
public void Process(CancellationToken cancellationToken)
{
    var task = ProcessAsync(cancellationToken);
    task.Wait();  // Will throw AggregateException instead of OperationCanceledException
}
```

### ❌ Creating Token Without Timeout

```csharp
// Never times out - potential infinite wait
var cts = new CancellationTokenSource();
await ProcessAsync(cts.Token);

// Better - always have a timeout
var cts = new CancellationTokenSource(TimeSpan.FromMinutes(5));
```

## See Also

- [Async Patterns](./async-patterns.md) — Task vs ValueTask
- [Error Handling](./error-handling.md) — handling exceptions
- [Honest Functions](./honest-functions.md) — explicit error handling with Result types
