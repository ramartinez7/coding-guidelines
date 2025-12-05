# Idempotency Keys (Distributed Consistency)

> Network retries cause duplicate operations—require an `IdempotencyKey` type to make operations safely repeatable.

## Problem

In distributed systems, networks fail. If a client sends a "Pay $100" request and the network times out *after* the server processes it but *before* the response returns, the client might retry. Without idempotency, the server sees two independent requests and charges $200.

## The Distributed Reality

```
Client                          Network                         Server
   |                               |                               |
   |------- Pay $100 ------------>|                               |
   |                               |------- Pay $100 ------------->|
   |                               |                               | ✓ Charged $100
   |                               |<------ 200 OK ----------------|
   |                               X (timeout/dropped)             |
   |                               |                               |
   | (no response, retry...)       |                               |
   |------- Pay $100 ------------>|                               |
   |                               |------- Pay $100 ------------->|
   |                               |                               | ✓ Charged $100 AGAIN!
   |                               |<------ 200 OK ----------------|
   |<------ 200 OK ----------------|                               |
   |                               |                               |
   | Total charged: $200 💥        |                               |
```

## Example

### ❌ Before

```csharp
public class PaymentController
{
    [HttpPost("payments")]
    public async Task<IActionResult> ProcessPayment(PaymentRequest request)
    {
        // No way to detect a retry—every call creates a new charge
        var result = await _paymentService.ChargeAsync(
            request.CustomerId,
            request.Amount);

        return Ok(new { TransactionId = result.TransactionId });
    }
}

public class PaymentService
{
    public async Task<PaymentResult> ChargeAsync(Guid customerId, decimal amount)
    {
        // Called twice = charged twice
        var transaction = await _gateway.Charge(customerId, amount);
        await _db.Transactions.AddAsync(transaction);
        await _db.SaveChangesAsync();

        return new PaymentResult(transaction.Id);
    }
}

// Client code with retry logic
async Task PayWithRetry(PaymentRequest request)
{
    for (int i = 0; i < 3; i++)
    {
        try
        {
            return await _httpClient.PostAsync("/payments", request);
        }
        catch (TimeoutException)
        {
            // Retry... but did the first one succeed? 🤷
        }
    }
}
```

### ✅ After

```csharp
// Strongly-typed idempotency key
public readonly record struct IdempotencyKey
{
    public string Value { get; }

    private IdempotencyKey(string value) => Value = value;

    public static IdempotencyKey New() => new(Guid.NewGuid().ToString("N"));

    public static IdempotencyKey Parse(string value)
    {
        if (string.IsNullOrWhiteSpace(value))
            throw new ArgumentException("Idempotency key cannot be empty");
        return new IdempotencyKey(value);
    }
}

// Request includes the key
public sealed record PaymentRequest(
    IdempotencyKey IdempotencyKey,
    CustomerId CustomerId,
    Money Amount);

public class PaymentController
{
    [HttpPost("payments")]
    public async Task<IActionResult> ProcessPayment(
        [FromHeader(Name = "Idempotency-Key")] string idempotencyKeyHeader,
        [FromBody] PaymentRequestDto request)
    {
        var idempotencyKey = IdempotencyKey.Parse(idempotencyKeyHeader);

        var result = await _paymentService.ChargeAsync(
            idempotencyKey,
            CustomerId.Parse(request.CustomerId),
            Money.Create(request.Amount, request.Currency));

        return result.Match<IActionResult>(
            success => Ok(new { success.TransactionId }),
            duplicate => Ok(new { duplicate.OriginalTransactionId }));  // Return cached result
    }
}

public class PaymentService
{
    public async Task<OneOf<PaymentSuccess, DuplicateRequest>> ChargeAsync(
        IdempotencyKey key,
        CustomerId customerId,
        Money amount)
    {
        // Check if we've seen this key before
        var existing = await _db.IdempotencyRecords
            .FirstOrDefaultAsync(r => r.Key == key.Value);

        if (existing is not null)
        {
            // Return the cached result—don't charge again
            return new DuplicateRequest(existing.TransactionId);
        }

        // Process the payment
        var transaction = await _gateway.Charge(customerId, amount);

        // Store the idempotency record atomically with the transaction
        _db.IdempotencyRecords.Add(new IdempotencyRecord
        {
            Key = key.Value,
            TransactionId = transaction.Id,
            CreatedAt = DateTimeOffset.UtcNow
        });
        await _db.Transactions.AddAsync(transaction);
        await _db.SaveChangesAsync();

        return new PaymentSuccess(transaction.Id);
    }
}
```

**Client code—safe retries:**

```csharp
async Task PayWithRetry(CustomerId customerId, Money amount)
{
    // Generate key ONCE, reuse on retries
    var idempotencyKey = IdempotencyKey.New();

    for (int i = 0; i < 3; i++)
    {
        try
        {
            var response = await _httpClient.PostAsync(
                "/payments",
                new PaymentRequestDto(customerId, amount),
                headers: new { IdempotencyKey = idempotencyKey.Value });

            return response;  // Success or duplicate—both are fine
        }
        catch (TimeoutException)
        {
            // Safe to retry—same key means same operation
        }
    }
}
```

## Why It's a Problem

1. **At-least-once delivery**: Networks guarantee messages arrive *at least* once—duplicates are the norm, not the exception.

2. **Missing identity**: Without a key, each request looks like a new operation.

3. **Silent corruption**: Double-charges, duplicate orders, and phantom records appear without errors.

4. **Impossible debugging**: "Why was the customer charged twice?" has no answer in the logs.

5. **Client complexity**: Without idempotency, clients must implement complex "check then act" logic.

## Symptoms

- Customer complaints about duplicate charges
- Database records with near-identical timestamps
- "Retry storm" incidents causing cascading duplicates
- Complex client code trying to detect "did my request succeed?"
- Manual reconciliation processes to find and fix duplicates

## The Idempotency Record

```csharp
public sealed class IdempotencyRecord
{
    public required string Key { get; init; }
    public required string TransactionId { get; init; }
    public required DateTimeOffset CreatedAt { get; init; }
    public required string ResponseBody { get; init; }  // Cache the full response
    public required int ResponseStatusCode { get; init; }
}

// With a unique constraint on Key
modelBuilder.Entity<IdempotencyRecord>()
    .HasIndex(r => r.Key)
    .IsUnique();
```

## Middleware Approach

For cross-cutting idempotency:

```csharp
public class IdempotencyMiddleware
{
    private readonly RequestDelegate _next;

    public async Task InvokeAsync(HttpContext context, IdempotencyStore store)
    {
        if (context.Request.Method != "POST" && context.Request.Method != "PUT")
        {
            await _next(context);
            return;
        }

        if (!context.Request.Headers.TryGetValue("Idempotency-Key", out var keyHeader))
        {
            context.Response.StatusCode = 400;
            await context.Response.WriteAsJsonAsync(new { error = "Idempotency-Key header required" });
            return;
        }

        var key = IdempotencyKey.Parse(keyHeader!);

        // Check for cached response
        var cached = await store.GetAsync(key);
        if (cached is not null)
        {
            context.Response.StatusCode = cached.StatusCode;
            await context.Response.WriteAsync(cached.Body);
            return;
        }

        // Capture the response
        var originalBody = context.Response.Body;
        using var memoryStream = new MemoryStream();
        context.Response.Body = memoryStream;

        await _next(context);

        // Store the response
        memoryStream.Seek(0, SeekOrigin.Begin);
        var responseBody = await new StreamReader(memoryStream).ReadToEndAsync();

        await store.SaveAsync(key, new CachedResponse(
            context.Response.StatusCode,
            responseBody));

        // Write to original stream
        memoryStream.Seek(0, SeekOrigin.Begin);
        await memoryStream.CopyToAsync(originalBody);
    }
}
```

## Scoped Idempotency Keys

Sometimes the same key should allow different operations for different resources:

```csharp
public readonly record struct ScopedIdempotencyKey(
    IdempotencyKey Key,
    string Scope)  // e.g., "payments", "orders", "refunds"
{
    public string CompositeKey => $"{Scope}:{Key.Value}";
}

// Usage
var paymentKey = new ScopedIdempotencyKey(key, "payments");
var refundKey = new ScopedIdempotencyKey(key, "refunds");
// Same key, different scopes—both allowed
```

## Expiration Strategy

Idempotency records shouldn't live forever:

```csharp
public class IdempotencyCleanupJob : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            var cutoff = DateTimeOffset.UtcNow.AddHours(-24);

            await _db.IdempotencyRecords
                .Where(r => r.CreatedAt < cutoff)
                .ExecuteDeleteAsync(stoppingToken);

            await Task.Delay(TimeSpan.FromHours(1), stoppingToken);
        }
    }
}
```

## Benefits

- **Safe retries**: Clients can retry without fear of duplicates
- **Network resilience**: Timeouts and failures don't corrupt data
- **Simpler clients**: No complex "did it work?" detection logic
- **Audit trail**: Idempotency records document retry patterns
- **Type safety**: `IdempotencyKey` in the signature makes the requirement explicit

## See Also

- [Strongly Typed IDs](./strongly-typed-ids.md)
- [Honest Functions](./honest-functions.md)
- [Capability Security (Token Types)](./capability-security.md)
