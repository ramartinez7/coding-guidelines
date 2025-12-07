# Correlation IDs (Type-Safe Distributed Tracing)

> Passing correlation IDs as strings or relying on ambient context—use typed correlation tokens to track requests across service boundaries.

## Problem

In distributed systems, a single user request might touch multiple services. When something goes wrong, you need to correlate logs across services. Typically, correlation IDs are passed as strings in headers or stored in thread-local storage, both error-prone approaches.

## Example

### ❌ Before

```csharp
public class OrderService
{
    private readonly ILogger<OrderService> _logger;
    
    public async Task<Order> CreateOrder(CreateOrderRequest request)
    {
        // Where's the correlation ID? In a header? In HttpContext?
        _logger.LogInformation("Creating order for customer {CustomerId}", 
            request.CustomerId);
        
        var order = new Order(request);
        await _repository.SaveAsync(order);
        
        // Calling another service—correlation ID might not propagate
        await _inventoryService.ReserveItemsAsync(order.Items);
        await _paymentService.ChargeAsync(order.Total);
        
        return order;
    }
}

public class PaymentService
{
    public async Task ChargeAsync(decimal amount)
    {
        // Is this the same request? Different correlation ID?
        _logger.LogInformation("Processing payment of {Amount}", amount);
        
        // Making HTTP call—must manually add correlation header
        var request = new HttpRequestMessage(HttpMethod.Post, "/charge");
        request.Headers.Add("X-Correlation-Id", GetCorrelationId()); // Where from?
        
        await _httpClient.SendAsync(request);
    }
}

// Storing in thread-local storage is fragile
public static class CorrelationContext
{
    private static AsyncLocal<string?> _correlationId = new();
    
    public static string? Current
    {
        get => _correlationId.Value;
        set => _correlationId.Value = value;
    }
}
```

**Problems:**
- Correlation ID is ambient state (thread-local)
- No compile-time guarantee it's set
- Easy to forget to propagate across service calls
- String-based with no validation
- Lost when crossing async boundaries
- No type safety

### ✅ After

```csharp
/// <summary>
/// Correlation ID that tracks a request through distributed system.
/// Cannot be constructed directly except at entry points.
/// </summary>
public readonly record struct CorrelationId
{
    public Guid Value { get; }
    
    private CorrelationId(Guid value) => Value = value;
    
    /// <summary>
    /// Creates a new correlation ID at system entry point (API gateway, message handler).
    /// </summary>
    public static CorrelationId New() => new(Guid.NewGuid());
    
    /// <summary>
    /// Parses correlation ID from incoming request header.
    /// </summary>
    public static Result<CorrelationId, string> Parse(string? value)
    {
        if (string.IsNullOrWhiteSpace(value))
            return Result<CorrelationId, string>.Failure("Correlation ID cannot be empty");
        
        if (!Guid.TryParse(value, out var guid))
            return Result<CorrelationId, string>.Failure("Invalid correlation ID format");
        
        return Result<CorrelationId, string>.Success(new CorrelationId(guid));
    }
    
    public override string ToString() => Value.ToString("N");
}

/// <summary>
/// Request context that carries correlation through the call chain.
/// </summary>
public sealed record RequestContext
{
    public required CorrelationId CorrelationId { get; init; }
    public required DateTime Timestamp { get; init; }
    public Option<UserId> UserId { get; init; } = Option<UserId>.None;
    public Option<string> TraceParent { get; init; } = Option<string>.None;
    
    public static RequestContext Create(CorrelationId correlationId, Option<UserId> userId = default)
    {
        return new RequestContext
        {
            CorrelationId = correlationId,
            Timestamp = DateTime.UtcNow,
            UserId = userId
        };
    }
    
    /// <summary>
    /// Creates child context for downstream call (preserves correlation).
    /// </summary>
    public RequestContext CreateChild()
    {
        return new RequestContext
        {
            CorrelationId = this.CorrelationId,  // Same correlation
            Timestamp = DateTime.UtcNow,          // New timestamp
            UserId = this.UserId,                 // Preserve user
            TraceParent = this.TraceParent
        };
    }
}

// All service methods accept RequestContext as first parameter
public interface IOrderService
{
    Task<Order> CreateOrderAsync(RequestContext context, CreateOrderRequest request);
}

public class OrderService : IOrderService
{
    private readonly ILogger<OrderService> _logger;
    private readonly IInventoryService _inventoryService;
    private readonly IPaymentService _paymentService;
    
    public async Task<Order> CreateOrderAsync(
        RequestContext context,
        CreateOrderRequest request)
    {
        // Correlation ID is explicit—part of the signature
        _logger.LogInformation(
            "Creating order for customer {CustomerId} [CorrelationId: {CorrelationId}]",
            request.CustomerId,
            context.CorrelationId);
        
        var order = new Order(request);
        await _repository.SaveAsync(order);
        
        // Pass context to downstream services—compiler enforces it
        await _inventoryService.ReserveItemsAsync(
            context.CreateChild(),  // Child context preserves correlation
            order.Items);
        
        await _paymentService.ChargeAsync(
            context.CreateChild(),
            order.Total);
        
        return order;
    }
}

public interface IPaymentService
{
    Task ChargeAsync(RequestContext context, decimal amount);
}

public class PaymentService : IPaymentService
{
    private readonly HttpClient _httpClient;
    private readonly ILogger<PaymentService> _logger;
    
    public async Task ChargeAsync(RequestContext context, decimal amount)
    {
        _logger.LogInformation(
            "Processing payment of {Amount} [CorrelationId: {CorrelationId}]",
            amount,
            context.CorrelationId);
        
        // Propagate correlation ID to external service
        var request = new HttpRequestMessage(HttpMethod.Post, "/charge");
        request.Headers.Add("X-Correlation-Id", context.CorrelationId.ToString());
        
        var response = await _httpClient.SendAsync(request);
        response.EnsureSuccessStatusCode();
    }
}

// Middleware creates RequestContext from headers
public class CorrelationMiddleware
{
    private readonly RequestDelegate _next;
    
    public async Task InvokeAsync(HttpContext context)
    {
        // Try to get correlation ID from header
        var correlationId = context.Request.Headers.TryGetValue("X-Correlation-Id", out var headerValue)
            ? CorrelationId.Parse(headerValue.ToString()).Match(
                onSuccess: id => id,
                onFailure: _ => CorrelationId.New())  // Invalid, create new
            : CorrelationId.New();  // Missing, create new
        
        // Create request context
        var requestContext = RequestContext.Create(correlationId);
        
        // Store in HttpContext.Items (scoped to this request)
        context.Items["RequestContext"] = requestContext;
        
        // Add to response headers
        context.Response.Headers.Add("X-Correlation-Id", correlationId.ToString());
        
        await _next(context);
    }
}

// Controller extracts RequestContext
[ApiController]
[Route("api/orders")]
public class OrdersController : ControllerBase
{
    private readonly IOrderService _orderService;
    
    [HttpPost]
    public async Task<IActionResult> CreateOrder([FromBody] CreateOrderRequest request)
    {
        // Get context from middleware
        var context = (RequestContext)HttpContext.Items["RequestContext"]!;
        
        var order = await _orderService.CreateOrderAsync(context, request);
        
        return Ok(order);
    }
}
```

## HttpClient Integration

```csharp
/// <summary>
/// DelegatingHandler that adds correlation ID to outgoing requests.
/// </summary>
public class CorrelationIdHandler : DelegatingHandler
{
    private readonly IHttpContextAccessor _httpContextAccessor;
    
    public CorrelationIdHandler(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }
    
    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request,
        CancellationToken cancellationToken)
    {
        // Get correlation ID from current request context
        var httpContext = _httpContextAccessor.HttpContext;
        if (httpContext?.Items["RequestContext"] is RequestContext context)
        {
            request.Headers.Add("X-Correlation-Id", context.CorrelationId.ToString());
        }
        
        return await base.SendAsync(request, cancellationToken);
    }
}

// Register in DI
services.AddHttpClient<IPaymentService, PaymentService>()
    .AddHttpMessageHandler<CorrelationIdHandler>();
```

## Structured Logging Integration

```csharp
/// <summary>
/// Logger that automatically includes correlation ID.
/// </summary>
public interface ICorrelatedLogger<T>
{
    void LogInformation(RequestContext context, string message, params object[] args);
    void LogError(RequestContext context, Exception ex, string message, params object[] args);
    void LogWarning(RequestContext context, string message, params object[] args);
}

public class CorrelatedLogger<T> : ICorrelatedLogger<T>
{
    private readonly ILogger<T> _logger;
    
    public CorrelatedLogger(ILogger<T> logger)
    {
        _logger = logger;
    }
    
    public void LogInformation(RequestContext context, string message, params object[] args)
    {
        using var scope = _logger.BeginScope(new Dictionary<string, object>
        {
            ["CorrelationId"] = context.CorrelationId,
            ["Timestamp"] = context.Timestamp,
            ["UserId"] = context.UserId.Match(
                onSome: id => id.ToString(),
                onNone: () => "anonymous")
        });
        
        _logger.LogInformation(message, args);
    }
    
    public void LogError(RequestContext context, Exception ex, string message, params object[] args)
    {
        using var scope = _logger.BeginScope(new Dictionary<string, object>
        {
            ["CorrelationId"] = context.CorrelationId,
            ["Timestamp"] = context.Timestamp
        });
        
        _logger.LogError(ex, message, args);
    }
    
    public void LogWarning(RequestContext context, string message, params object[] args)
    {
        using var scope = _logger.BeginScope(new Dictionary<string, object>
        {
            ["CorrelationId"] = context.CorrelationId
        });
        
        _logger.LogWarning(message, args);
    }
}

// Usage
public class OrderService : IOrderService
{
    private readonly ICorrelatedLogger<OrderService> _logger;
    
    public async Task<Order> CreateOrderAsync(
        RequestContext context,
        CreateOrderRequest request)
    {
        // Correlation ID automatically included in all logs
        _logger.LogInformation(context, 
            "Creating order for customer {CustomerId}", 
            request.CustomerId);
        
        try
        {
            var order = await CreateOrderInternalAsync(request);
            return order;
        }
        catch (Exception ex)
        {
            _logger.LogError(context, ex, 
                "Failed to create order for customer {CustomerId}", 
                request.CustomerId);
            throw;
        }
    }
}
```

## Message Queue Integration

```csharp
/// <summary>
/// Message envelope that carries correlation through async messaging.
/// </summary>
public sealed record MessageEnvelope<T>
{
    public required CorrelationId CorrelationId { get; init; }
    public required DateTime Timestamp { get; init; }
    public required T Payload { get; init; }
    public Option<string> CausationId { get; init; } = Option<string>.None;
}

public interface IMessagePublisher
{
    Task PublishAsync<T>(RequestContext context, T message);
}

public class MessagePublisher : IMessagePublisher
{
    private readonly IMessageBus _messageBus;
    
    public async Task PublishAsync<T>(RequestContext context, T message)
    {
        var envelope = new MessageEnvelope<T>
        {
            CorrelationId = context.CorrelationId,  // Preserve correlation
            Timestamp = DateTime.UtcNow,
            Payload = message,
            CausationId = Option<string>.None
        };
        
        await _messageBus.PublishAsync(envelope);
    }
}

// Message handler extracts context from envelope
public class OrderCreatedHandler : IMessageHandler<MessageEnvelope<OrderCreatedEvent>>
{
    private readonly ICorrelatedLogger<OrderCreatedHandler> _logger;
    
    public async Task HandleAsync(MessageEnvelope<OrderCreatedEvent> envelope)
    {
        // Reconstruct request context from envelope
        var context = RequestContext.Create(envelope.CorrelationId);
        
        _logger.LogInformation(context, 
            "Handling OrderCreated event for order {OrderId}",
            envelope.Payload.OrderId);
        
        // All downstream calls inherit correlation ID
        await ProcessOrderAsync(context, envelope.Payload);
    }
}
```

## Testing

```csharp
public class CorrelationTests
{
    [Fact]
    public async Task ServiceCall_WithContext_PreservesCorrelationId()
    {
        var correlationId = CorrelationId.New();
        var context = RequestContext.Create(correlationId);
        
        var service = new OrderService(_logger, _inventory, _payment);
        await service.CreateOrderAsync(context, new CreateOrderRequest());
        
        // Verify downstream services received same correlation ID
        _inventory.Verify(x => x.ReserveItemsAsync(
            It.Is<RequestContext>(ctx => ctx.CorrelationId == correlationId),
            It.IsAny<IEnumerable<OrderItem>>()));
        
        _payment.Verify(x => x.ChargeAsync(
            It.Is<RequestContext>(ctx => ctx.CorrelationId == correlationId),
            It.IsAny<decimal>()));
    }
    
    [Fact]
    public void CorrelationId_Parse_InvalidFormat_ReturnsFailure()
    {
        var result = CorrelationId.Parse("not-a-guid");
        
        Assert.True(result.IsFailure);
    }
    
    [Fact]
    public void RequestContext_CreateChild_PreservesCorrelation()
    {
        var correlationId = CorrelationId.New();
        var parent = RequestContext.Create(correlationId);
        
        var child = parent.CreateChild();
        
        Assert.Equal(parent.CorrelationId, child.CorrelationId);
        Assert.NotEqual(parent.Timestamp, child.Timestamp);
    }
}
```

## Why It's a Problem

1. **Ambient state**: Thread-local storage is fragile and implicit
2. **Lost across boundaries**: Easy to forget propagation
3. **No type safety**: String-based correlation IDs
4. **Manual logging**: Must remember to include correlation ID
5. **Async issues**: Lost when crossing async boundaries

## Symptoms

- Logs from different services can't be correlated
- Thread-local storage used for correlation
- Correlation ID in headers but not propagated
- Missing correlation IDs in logs
- Different correlation IDs for same request

## Benefits

- **Explicit propagation**: Correlation ID in method signatures
- **Type safety**: Cannot create invalid correlation IDs
- **Compile-time enforcement**: Cannot forget to pass context
- **Automatic logging**: Logger includes correlation automatically
- **Works across boundaries**: HTTP, message queues, etc.

## See Also

- [Authentication Context](./authentication-context.md) — user identity
- [Idempotency Keys](./idempotency-keys.md) — request deduplication
- [Strongly Typed IDs](./strongly-typed-ids.md) — ID types
- [Capability Security](./capability-security.md) — token passing
