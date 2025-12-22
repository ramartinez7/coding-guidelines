# Async/Await Anti-Patterns

> Avoid common pitfalls when writing asynchronous code—deadlocks, blocking calls, and fire-and-forget patterns.

## Problem

Misusing async/await leads to deadlocks, poor performance, unhandled exceptions, and difficult-to-debug code.

## Example

### ❌ Before

```csharp
public class UserService
{
    // Anti-pattern 1: Async void (except event handlers)
    public async void LoadUser(UserId userId)
    {
        var user = await _repository.GetUserAsync(userId);
        UpdateUI(user);  // Exception here is unhandled!
    }

    // Anti-pattern 2: Blocking on async code
    public User GetUser(UserId userId)
    {
        return _repository.GetUserAsync(userId).Result;  // Deadlock risk!
    }

    // Anti-pattern 3: Unnecessary Task.Run
    public async Task<int> CalculateTotalAsync(List<int> numbers)
    {
        return await Task.Run(() => numbers.Sum());  // Already CPU-bound, no I/O
    }

    // Anti-pattern 4: Not using ConfigureAwait in libraries
    public async Task<User> GetUserAsync(UserId userId)
    {
        var user = await _repository.GetUserAsync(userId);
        return user;
    }
}
```

### ✅ After

```csharp
public class UserService
{
    // ✅ Return Task for exception handling
    public async Task LoadUserAsync(UserId userId)
    {
        var user = await _repository.GetUserAsync(userId);
        UpdateUI(user);
    }

    // ✅ Async all the way
    public async Task<User> GetUserAsync(UserId userId)
    {
        return await _repository.GetUserAsync(userId);
    }

    // ✅ Don't wrap synchronous code in Task.Run
    public Task<int> CalculateTotalAsync(List<int> numbers)
    {
        var result = numbers.Sum();
        return Task.FromResult(result);
    }

    // ✅ Use ConfigureAwait(false) in libraries
    public async Task<User> GetUserInLibraryAsync(UserId userId)
    {
        var user = await _repository
            .GetUserAsync(userId)
            .ConfigureAwait(false);
        return user;
    }
}
```

## Anti-Patterns and Solutions

### 1. Async Void

```csharp
// ❌ Async void - exceptions are unhandled
public async void ProcessOrderAsync(OrderId orderId)
{
    var order = await _repository.GetOrderAsync(orderId);
    await ProcessPaymentAsync(order);  // Exception lost!
}

// ✅ Async Task - exceptions can be caught
public async Task ProcessOrderAsync(OrderId orderId)
{
    var order = await _repository.GetOrderAsync(orderId);
    await ProcessPaymentAsync(order);
}

// ✅ Only use async void for event handlers
private async void OnButtonClick(object sender, EventArgs e)
{
    try
    {
        await ProcessOrderAsync(orderId);
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Failed to process order");
        ShowErrorMessage(ex.Message);
    }
}
```

### 2. Blocking on Async Code

```csharp
// ❌ Deadlock risk with .Result or .Wait()
public User GetUser(UserId userId)
{
    return _repository.GetUserAsync(userId).Result;  // DEADLOCK!
}

public void ProcessOrder(OrderId orderId)
{
    var order = _repository.GetOrderAsync(orderId).Wait();  // DEADLOCK!
}

// ✅ Async all the way
public async Task<User> GetUserAsync(UserId userId)
{
    return await _repository.GetUserAsync(userId);
}

public async Task ProcessOrderAsync(OrderId orderId)
{
    var order = await _repository.GetOrderAsync(orderId);
}
```

### 3. Unnecessary Task.Run

```csharp
// ❌ Wrapping non-async code unnecessarily
public async Task<decimal> CalculateTotalAsync(Order order)
{
    return await Task.Run(() => 
    {
        return order.Items.Sum(i => i.Price);  // Already synchronous!
    });
}

// ✅ Just return the synchronous result
public decimal CalculateTotal(Order order)
{
    return order.Items.Sum(i => i.Price);
}

// ✅ Or if you must return Task
public Task<decimal> CalculateTotalAsync(Order order)
{
    var total = order.Items.Sum(i => i.Price);
    return Task.FromResult(total);
}
```

### 4. Missing ConfigureAwait(false) in Libraries

```csharp
// ❌ In library code, can cause deadlocks for callers
public async Task<User> GetUserAsync(UserId userId)
{
    var user = await _repository.GetUserAsync(userId);
    var enriched = await EnrichUserDataAsync(user);
    return enriched;
}

// ✅ Use ConfigureAwait(false) in library code
public async Task<User> GetUserAsync(UserId userId)
{
    var user = await _repository
        .GetUserAsync(userId)
        .ConfigureAwait(false);

    var enriched = await EnrichUserDataAsync(user)
        .ConfigureAwait(false);

    return enriched;
}

// Note: Don't use ConfigureAwait(false) in UI or ASP.NET Core apps
```

### 5. Fire-and-Forget

```csharp
// ❌ Fire-and-forget loses exceptions
public void ProcessOrder(OrderId orderId)
{
    _ = ProcessOrderAsync(orderId);  // Exception lost!
}

// ✅ Await the task
public async Task ProcessOrderAsync(OrderId orderId)
{
    await ProcessOrderInternalAsync(orderId);
}

// ✅ If truly fire-and-forget, handle exceptions
public void ProcessOrder(OrderId orderId)
{
    _ = Task.Run(async () =>
    {
        try
        {
            await ProcessOrderAsync(orderId);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Background order processing failed");
        }
    });
}
```

### 6. Not Passing CancellationToken

```csharp
// ❌ No cancellation support
public async Task<List<User>> GetAllUsersAsync()
{
    var users = new List<User>();
    foreach (var id in _userIds)
    {
        var user = await _repository.GetUserAsync(id);
        users.Add(user);
    }
    return users;
}

// ✅ Support cancellation
public async Task<List<User>> GetAllUsersAsync(
    CancellationToken cancellationToken = default)
{
    var users = new List<User>();
    foreach (var id in _userIds)
    {
        cancellationToken.ThrowIfCancellationRequested();
        var user = await _repository.GetUserAsync(id, cancellationToken);
        users.Add(user);
    }
    return users;
}
```

### 7. Async Over Sync

```csharp
// ❌ Synchronous method pretending to be async
public async Task<User> GetUserAsync(UserId userId)
{
    var user = _cache.Get(userId);  // Synchronous!
    return await Task.FromResult(user);  // Unnecessary wrapper
}

// ✅ Just make it synchronous
public User GetUser(UserId userId)
{
    return _cache.Get(userId);
}

// ✅ Or use ValueTask if sometimes async
public ValueTask<User> GetUserAsync(UserId userId)
{
    if (_cache.TryGet(userId, out var user))
    {
        return new ValueTask<User>(user);  // Synchronous path
    }
    return new ValueTask<User>(_repository.GetUserAsync(userId));  // Async path
}
```

### 8. Returning Null from Async Methods

```csharp
// ❌ Returning null forces callers to check
public async Task<User?> GetUserAsync(UserId userId)
{
    var user = await _repository.FindAsync(userId);
    return user;  // Could be null
}

// Caller must check
var user = await GetUserAsync(userId);
if (user == null) { /* handle */ }

// ✅ Return Result or Option type
public async Task<Result<User, UserError>> GetUserAsync(UserId userId)
{
    var user = await _repository.FindAsync(userId);
    return user != null
        ? Result.Success<User, UserError>(user)
        : Result.Failure<User, UserError>(new UserNotFound(userId));
}

// Caller uses type-safe pattern matching
var result = await GetUserAsync(userId);
return result.Match(
    onSuccess: user => ProcessUser(user),
    onFailure: error => HandleError(error)
);
```

### 9. Sequential When Parallel is Possible

```csharp
// ❌ Sequential execution (slow)
public async Task<OrderSummary> GetOrderSummaryAsync(OrderId orderId)
{
    var order = await _orderRepo.GetAsync(orderId);
    var customer = await _customerRepo.GetAsync(order.CustomerId);
    var products = await _productRepo.GetManyAsync(order.ProductIds);

    return new OrderSummary(order, customer, products);
}

// ✅ Parallel execution (fast)
public async Task<OrderSummary> GetOrderSummaryAsync(OrderId orderId)
{
    var order = await _orderRepo.GetAsync(orderId);

    var customerTask = _customerRepo.GetAsync(order.CustomerId);
    var productsTask = _productRepo.GetManyAsync(order.ProductIds);

    await Task.WhenAll(customerTask, productsTask);

    return new OrderSummary(
        order, 
        customerTask.Result, 
        productsTask.Result);
}
```

### 10. Not Disposing Async Resources

```csharp
// ❌ Resource leak with async
public async Task ProcessFileAsync(string path)
{
    var stream = await GetFileStreamAsync(path);
    await ProcessStreamAsync(stream);
    // Forgot to dispose!
}

// ✅ Use await using
public async Task ProcessFileAsync(string path)
{
    await using var stream = await GetFileStreamAsync(path);
    await ProcessStreamAsync(stream);
}
```

## Symptoms

- Deadlocks in UI or ASP.NET applications
- Unhandled exceptions in async code
- Poor performance from blocking async code
- Memory leaks from undisposed async resources
- Requests timeout or hang indefinitely

## Benefits

- **No deadlocks** by avoiding blocking on async code
- **Better performance** with proper async/await usage
- **Proper error handling** by avoiding async void
- **Resource cleanup** with await using
- **Responsive applications** that don't block threads

## See Also

- [Async Patterns](./async-patterns.md) — Task vs ValueTask
- [CancellationToken Handling](./cancellation-token-handling.md) — Proper cancellation support
- [Resource Lifetime](./type-safe-resource-lifetime.md) — RAII pattern in C#
