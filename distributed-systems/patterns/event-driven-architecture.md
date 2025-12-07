# Event-Driven Architecture

> Services communicate through events published to a message bus—enables loose coupling, asynchrony, and scalability.

## Problem

Direct service-to-service communication creates tight coupling. When Service A calls Service B synchronously, they become dependent on each other's availability, performance, and API contracts. Event-driven architecture decouples services by having them communicate through events.

## Synchronous vs. Event-Driven

```
Synchronous Communication          Event-Driven Architecture
      
┌─────────┐                        ┌─────────┐
│Service A│                        │Service A│
└────┬────┘                        └────┬────┘
     │                                  │
     │ HTTP POST                        │ Publish Event
     │ (waits for B)                    │ (fire & forget)
     ▼                                  ▼
┌─────────┐                        ┌──────────┐
│Service B│                        │Event Bus │
└─────────┘                        └────┬─────┘
                                        │
If B is down,                           ├─► Service B
A fails too!                            ├─► Service C
                                        └─► Service D
                                        
                                   Services subscribe
                                   independently
```

## Example

### ❌ Synchronous Service Calls

```csharp
public class OrderService
{
    private readonly HttpClient _inventoryClient;
    private readonly HttpClient _paymentClient;
    private readonly HttpClient _shippingClient;
    private readonly HttpClient _emailClient;
    
    public async Task<Result<Order, OrderError>> CreateOrder(CreateOrderRequest request)
    {
        // Synchronous calls—all must succeed
        try
        {
            // Call inventory service
            var inventoryResponse = await _inventoryClient.PostAsync(
                "/reserve",
                JsonContent.Create(request.Items));
            inventoryResponse.EnsureSuccessStatusCode();
            
            // Call payment service
            var paymentResponse = await _paymentClient.PostAsync(
                "/charge",
                JsonContent.Create(new { request.CustomerId, request.Total }));
            paymentResponse.EnsureSuccessStatusCode();
            
            // Call shipping service
            var shippingResponse = await _shippingClient.PostAsync(
                "/create-shipment",
                JsonContent.Create(new { request.Address }));
            shippingResponse.EnsureSuccessStatusCode();
            
            // Call email service
            var emailResponse = await _emailClient.PostAsync(
                "/send-confirmation",
                JsonContent.Create(new { request.Email }));
            // Don't check success—email isn't critical
            
            return Result.Success(new Order(request));
        }
        catch (HttpRequestException ex)
        {
            // Any service failure fails the whole operation
            return Result.Failure<Order, OrderError>(
                new OrderError.ServiceUnavailable(ex.Message));
        }
    }
}
```

**Problems:**
- Tight coupling between services
- All services must be available
- Slow (sequential calls)
- Hard to add new subscribers
- Cascading failures

### ✅ Event-Driven Architecture

```csharp
/// <summary>
/// Domain events representing things that happened.
/// </summary>
public abstract record DomainEvent
{
    public Guid EventId { get; init; } = Guid.NewGuid();
    public DateTimeOffset Timestamp { get; init; } = DateTimeOffset.UtcNow;
    public CorrelationId CorrelationId { get; init; }
}

public sealed record OrderCreatedEvent(
    OrderId OrderId,
    CustomerId CustomerId,
    List<OrderItem> Items,
    Money Total,
    Address ShippingAddress,
    Email CustomerEmail) : DomainEvent;

/// <summary>
/// Event bus interface.
/// </summary>
public interface IEventBus
{
    Task PublishAsync<TEvent>(TEvent @event) where TEvent : DomainEvent;
}

/// <summary>
/// Order service publishes events.
/// </summary>
public class OrderService
{
    private readonly IOrderRepository _repository;
    private readonly IEventBus _eventBus;
    
    public async Task<Result<OrderId, OrderError>> CreateOrder(CreateOrderRequest request)
    {
        // Create order
        var orderId = OrderId.New();
        var order = new Order
        {
            Id = orderId,
            CustomerId = request.CustomerId,
            Items = request.Items,
            Total = request.Total,
            Status = OrderStatus.Created
        };
        
        await _repository.SaveAsync(order);
        
        // Publish event—fire and forget!
        await _eventBus.PublishAsync(new OrderCreatedEvent(
            orderId,
            request.CustomerId,
            request.Items,
            request.Total,
            request.ShippingAddress,
            request.CustomerEmail));
        
        // Done! Other services react asynchronously
        return Result.Success(orderId);
    }
}

/// <summary>
/// Inventory service subscribes to OrderCreated events.
/// </summary>
public class InventoryEventHandler : IEventHandler<OrderCreatedEvent>
{
    private readonly IInventoryService _inventoryService;
    private readonly IEventBus _eventBus;
    private readonly ILogger<InventoryEventHandler> _logger;
    
    public async Task HandleAsync(OrderCreatedEvent @event)
    {
        try
        {
            await _inventoryService.ReserveAsync(@event.Items);
            
            // Publish success event
            await _eventBus.PublishAsync(new InventoryReservedEvent(
                @event.OrderId,
                @event.Items));
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to reserve inventory for order {OrderId}", 
                @event.OrderId);
            
            // Publish failure event
            await _eventBus.PublishAsync(new InventoryReservationFailedEvent(
                @event.OrderId,
                ex.Message));
        }
    }
}

/// <summary>
/// Payment service subscribes to OrderCreated events.
/// </summary>
public class PaymentEventHandler : IEventHandler<OrderCreatedEvent>
{
    private readonly IPaymentService _paymentService;
    private readonly IEventBus _eventBus;
    private readonly ILogger<PaymentEventHandler> _logger;
    
    public async Task HandleAsync(OrderCreatedEvent @event)
    {
        try
        {
            var paymentId = await _paymentService.ChargeAsync(
                @event.CustomerId,
                @event.Total);
            
            await _eventBus.PublishAsync(new PaymentChargedEvent(
                @event.OrderId,
                paymentId,
                @event.Total));
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to charge payment for order {OrderId}", 
                @event.OrderId);
            
            await _eventBus.PublishAsync(new PaymentFailedEvent(
                @event.OrderId,
                ex.Message));
        }
    }
}

/// <summary>
/// Shipping service subscribes to OrderCreated events.
/// </summary>
public class ShippingEventHandler : IEventHandler<OrderCreatedEvent>
{
    private readonly IShippingService _shippingService;
    private readonly ILogger<ShippingEventHandler> _logger;
    
    public async Task HandleAsync(OrderCreatedEvent @event)
    {
        try
        {
            await _shippingService.CreateShipmentAsync(
                @event.OrderId,
                @event.ShippingAddress);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to create shipment for order {OrderId}", 
                @event.OrderId);
            // Non-critical—retry later
        }
    }
}

/// <summary>
/// Email service subscribes to OrderCreated events.
/// </summary>
public class EmailEventHandler : IEventHandler<OrderCreatedEvent>
{
    private readonly IEmailService _emailService;
    private readonly ILogger<EmailEventHandler> _logger;
    
    public async Task HandleAsync(OrderCreatedEvent @event)
    {
        try
        {
            await _emailService.SendOrderConfirmationAsync(
                @event.CustomerEmail,
                @event.OrderId);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to send confirmation email for order {OrderId}", 
                @event.OrderId);
            // Non-critical—don't fail the order
        }
    }
}
```

## Event Bus Implementation

### In-Memory Event Bus (Single Process)

```csharp
public class InMemoryEventBus : IEventBus
{
    private readonly IServiceProvider _serviceProvider;
    
    public async Task PublishAsync<TEvent>(TEvent @event) where TEvent : DomainEvent
    {
        // Get all handlers for this event type
        var handlers = _serviceProvider.GetServices<IEventHandler<TEvent>>();
        
        // Invoke all handlers (can be parallel or sequential)
        var tasks = handlers.Select(handler => handler.HandleAsync(@event));
        
        await Task.WhenAll(tasks);
    }
}
```

### Message Queue Event Bus (Distributed)

```csharp
public class RabbitMqEventBus : IEventBus
{
    private readonly IConnection _connection;
    private readonly ILogger<RabbitMqEventBus> _logger;
    
    public async Task PublishAsync<TEvent>(TEvent @event) where TEvent : DomainEvent
    {
        using var channel = _connection.CreateModel();
        
        var exchange = "domain-events";
        var routingKey = typeof(TEvent).Name;
        
        // Declare exchange
        channel.ExchangeDeclare(
            exchange,
            ExchangeType.Topic,
            durable: true);
        
        // Serialize event
        var body = JsonSerializer.SerializeToUtf8Bytes(@event);
        
        // Publish with properties
        var properties = channel.CreateBasicProperties();
        properties.Persistent = true;
        properties.ContentType = "application/json";
        properties.Type = typeof(TEvent).AssemblyQualifiedName;
        properties.CorrelationId = @event.CorrelationId.ToString();
        
        channel.BasicPublish(
            exchange,
            routingKey,
            properties,
            body);
        
        _logger.LogInformation(
            "Published event {EventType} with ID {EventId}",
            typeof(TEvent).Name,
            @event.EventId);
    }
}

/// <summary>
/// Consumer that processes events from queue.
/// </summary>
public class EventConsumer : BackgroundService
{
    private readonly IConnection _connection;
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<EventConsumer> _logger;
    
    protected override Task ExecuteAsync(CancellationToken stoppingToken)
    {
        var channel = _connection.CreateModel();
        
        var exchange = "domain-events";
        var queue = "inventory-service-queue";
        
        // Declare queue
        channel.QueueDeclare(
            queue,
            durable: true,
            exclusive: false,
            autoDelete: false);
        
        // Bind to event types we care about
        channel.QueueBind(queue, exchange, "OrderCreatedEvent");
        channel.QueueBind(queue, exchange, "PaymentChargedEvent");
        
        var consumer = new EventingBasicConsumer(channel);
        consumer.Received += async (model, ea) =>
        {
            try
            {
                // Deserialize event
                var body = ea.Body.ToArray();
                var eventType = Type.GetType(ea.BasicProperties.Type);
                var @event = JsonSerializer.Deserialize(body, eventType) as DomainEvent;
                
                if (@event != null)
                {
                    // Invoke handler
                    await InvokeHandlerAsync(@event);
                    
                    // Acknowledge
                    channel.BasicAck(ea.DeliveryTag, multiple: false);
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to process event");
                
                // Negative acknowledge—will retry
                channel.BasicNack(ea.DeliveryTag, multiple: false, requeue: true);
            }
        };
        
        channel.BasicConsume(queue, autoAck: false, consumer);
        
        return Task.CompletedTask;
    }
    
    private async Task InvokeHandlerAsync(DomainEvent @event)
    {
        var eventType = @event.GetType();
        var handlerType = typeof(IEventHandler<>).MakeGenericType(eventType);
        
        using var scope = _serviceProvider.CreateScope();
        var handlers = scope.ServiceProvider.GetServices(handlerType);
        
        foreach (var handler in handlers)
        {
            var method = handlerType.GetMethod("HandleAsync");
            var task = (Task)method!.Invoke(handler, new object[] { @event })!;
            await task;
        }
    }
}
```

## Event Versioning

Events are contracts—handle versioning carefully:

```csharp
/// <summary>
/// Version 1 of OrderCreated event.
/// </summary>
public sealed record OrderCreatedEventV1(
    OrderId OrderId,
    CustomerId CustomerId,
    List<OrderItem> Items,
    decimal Total) : DomainEvent;

/// <summary>
/// Version 2 adds shipping address.
/// </summary>
public sealed record OrderCreatedEventV2(
    OrderId OrderId,
    CustomerId CustomerId,
    List<OrderItem> Items,
    Money Total,  // Changed from decimal to Money type
    Address ShippingAddress) : DomainEvent;

/// <summary>
/// Handler supports both versions.
/// </summary>
public class BackwardCompatibleHandler : 
    IEventHandler<OrderCreatedEventV1>,
    IEventHandler<OrderCreatedEventV2>
{
    public Task HandleAsync(OrderCreatedEventV1 @event)
    {
        // Convert V1 to internal representation
        return ProcessOrder(
            @event.OrderId,
            @event.CustomerId,
            @event.Items,
            Money.USD(@event.Total),
            Address.Unknown);  // V1 didn't have address
    }
    
    public Task HandleAsync(OrderCreatedEventV2 @event)
    {
        // V2 has all fields
        return ProcessOrder(
            @event.OrderId,
            @event.CustomerId,
            @event.Items,
            @event.Total,
            @event.ShippingAddress);
    }
    
    private Task ProcessOrder(
        OrderId orderId,
        CustomerId customerId,
        List<OrderItem> items,
        Money total,
        Address shippingAddress)
    {
        // Actual processing logic
        return Task.CompletedTask;
    }
}
```

## Event Enrichment

Add context to events automatically:

```csharp
public class EnrichedEventBus : IEventBus
{
    private readonly IEventBus _innerBus;
    private readonly IHttpContextAccessor _httpContext;
    
    public async Task PublishAsync<TEvent>(TEvent @event) where TEvent : DomainEvent
    {
        // Enrich event with metadata
        var enriched = new EnrichedEvent<TEvent>
        {
            Event = @event,
            UserId = _httpContext.HttpContext?.User.FindFirst("sub")?.Value,
            TenantId = _httpContext.HttpContext?.Request.Headers["X-Tenant-Id"],
            IpAddress = _httpContext.HttpContext?.Connection.RemoteIpAddress?.ToString(),
            UserAgent = _httpContext.HttpContext?.Request.Headers["User-Agent"]
        };
        
        await _innerBus.PublishAsync(enriched);
    }
}

public class EnrichedEvent<TEvent> : DomainEvent where TEvent : DomainEvent
{
    public required TEvent Event { get; init; }
    public string? UserId { get; init; }
    public string? TenantId { get; init; }
    public string? IpAddress { get; init; }
    public string? UserAgent { get; init; }
}
```

## Why It's a Problem (Not Using Event-Driven Architecture)

1. **Tight coupling**: Services depend on each other's availability
2. **Cascading failures**: One slow service affects all
3. **Difficult to scale**: Can't scale services independently
4. **Hard to extend**: Adding new functionality requires changing existing code

## Symptoms

- Services frequently fail together
- Adding new features requires coordinating deployments
- One slow service degrades entire system
- Complex retry and error handling logic

## Benefits

- **Loose coupling**: Services don't know about each other
- **Scalability**: Scale event handlers independently
- **Resilience**: Service failures don't propagate
- **Extensibility**: Add new subscribers without changing publishers
- **Audit trail**: Events provide natural audit log

## See Also

- [Message Queues](./message-queues.md) — durable event transport
- [Saga Pattern](./saga-pattern.md) — coordinating via events
- [Event Sourcing](./event-sourcing.md) — events as source of truth
- [CQRS](./cqrs.md) — events connect write and read models
- [Eventual Consistency](./eventual-consistency.md) — accepting async updates
