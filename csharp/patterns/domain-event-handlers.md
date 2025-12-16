# Domain Event Handlers

> Domain events decouple side effects from core business logic—publish events when something significant happens in the domain, then handle them with separate event handlers to maintain clean boundaries.

## Problem

Business operations often trigger multiple side effects (send emails, update caches, publish to message queues, update read models). When these side effects are inline with the business logic, the code becomes hard to test, understand, and maintain. A single method becomes a "god method" that does everything.

## Core Concept

Domain events represent something that happened in the domain that domain experts care about. Events are:
- **Past tense** (OrderPlaced, CustomerRegistered, PaymentProcessed)
- **Immutable** (they represent historical facts)
- **Published** by aggregates when state changes
- **Handled** by separate event handlers

```
┌──────────────────────────────────────────────┐
│         Domain Aggregate                     │
│  1. Execute business logic                   │
│  2. Change state                             │
│  3. Raise domain event                       │
└──────────────┬───────────────────────────────┘
               │
               │ (events collected but not dispatched yet)
               │
┌──────────────▼───────────────────────────────┐
│    Application Service / Use Case            │
│  4. Save aggregate                           │
│  5. Commit transaction                       │
│  6. Dispatch events                          │
└──────────────┬───────────────────────────────┘
               │
               │ (events dispatched to handlers)
               ├────────────┬────────────┬──────┤
               │            │            │      │
        ┌──────▼──────┐ ┌──▼──────┐ ┌──▼─────┐│
        │   Handler 1  │ │Handler 2│ │Handler3││
        │ (Send Email) │ │(Update  │ │(Publish││
        │              │ │ Cache)  │ │to Queue││
        └──────────────┘ └─────────┘ └────────┘│
```

## Example

### ❌ Before (Inline Side Effects)

```csharp
public class PlaceOrderUseCase
{
    private readonly IOrderRepository _orderRepository;
    private readonly IEmailService _emailService;
    private readonly ICacheService _cacheService;
    private readonly IMessageBus _messageBus;
    private readonly IAnalyticsService _analyticsService;
    private readonly IInventoryService _inventoryService;
    
    public async Task<Result<OrderId, string>> ExecuteAsync(PlaceOrderCommand command)
    {
        var order = Order.Create(command.CustomerId);
        
        foreach (var item in command.Items)
        {
            order.AddItem(item.ProductId, item.Quantity);
        }
        
        order.Submit();
        
        await _orderRepository.SaveAsync(order);
        
        // ❌ Side effects mixed with core logic
        await _emailService.SendOrderConfirmationAsync(order.Id, order.CustomerId);
        
        // ❌ More side effects
        await _cacheService.InvalidateCustomerOrdersAsync(order.CustomerId);
        
        // ❌ Even more side effects
        await _messageBus.PublishAsync(new
        {
            OrderId = order.Id,
            CustomerId = order.CustomerId,
            Total = order.Total
        });
        
        // ❌ And more...
        await _analyticsService.TrackOrderPlacedAsync(order.Id);
        
        // ❌ What if this fails? The order is already saved!
        await _inventoryService.ReserveStockAsync(order.Items);
        
        return Result<OrderId, string>.Success(order.Id);
    }
}
```

**Problems:**
- Side effects tightly coupled to business logic
- Hard to test order placement without email, cache, analytics
- Can't add new side effects without modifying this method
- Unclear what happens if side effects fail
- Violates Single Responsibility Principle

### ✅ After (Domain Events with Handlers)

#### 1. Domain Layer (Events and Publishing)

```csharp
// Domain/Events/DomainEvent.cs
namespace MyApp.Domain.Events
{
    /// <summary>
    /// Base class for all domain events.
    /// </summary>
    public abstract record DomainEvent
    {
        public Guid EventId { get; init; } = Guid.NewGuid();
        public DateTime OccurredAt { get; init; } = DateTime.UtcNow;
    }
}

// Domain/Events/OrderPlacedEvent.cs
namespace MyApp.Domain.Events
{
    /// <summary>
    /// Event published when an order is successfully placed.
    /// Past tense - represents something that already happened.
    /// </summary>
    public sealed record OrderPlacedEvent(
        OrderId OrderId,
        CustomerId CustomerId,
        Money Total,
        List<OrderItemSnapshot> Items) : DomainEvent;
    
    public sealed record OrderItemSnapshot(
        ProductId ProductId,
        int Quantity,
        Money Price);
}

// Domain/Entities/Order.cs
namespace MyApp.Domain.Entities
{
    public sealed class Order
    {
        private readonly List<OrderItem> _items = new();
        private readonly List<DomainEvent> _domainEvents = new();
        
        public OrderId Id { get; }
        public CustomerId CustomerId { get; }
        public IReadOnlyList<OrderItem> Items => _items.AsReadOnly();
        public Money Total { get; private set; }
        public OrderStatus Status { get; private set; }
        
        /// <summary>
        /// Events raised by this aggregate.
        /// Collected but not dispatched until transaction commits.
        /// </summary>
        public IReadOnlyList<DomainEvent> DomainEvents => _domainEvents.AsReadOnly();
        
        public void ClearDomainEvents() => _domainEvents.Clear();
        
        private Order(OrderId id, CustomerId customerId)
        {
            Id = id;
            CustomerId = customerId;
            Total = Money.Zero;
            Status = OrderStatus.Draft;
        }
        
        public static Order Create(CustomerId customerId)
        {
            return new Order(OrderId.New(), customerId);
        }
        
        public Result<Unit, string> AddItem(
            ProductId productId,
            int quantity,
            Money price)
        {
            if (quantity <= 0)
                return Result<Unit, string>.Failure("Quantity must be positive");
            
            var item = new OrderItem(productId, quantity, price);
            _items.Add(item);
            RecalculateTotal();
            
            return Result<Unit, string>.Success(Unit.Value);
        }
        
        public Result<Unit, string> Submit()
        {
            if (_items.Count == 0)
                return Result<Unit, string>.Failure("Cannot submit empty order");
            
            if (Status != OrderStatus.Draft)
                return Result<Unit, string>.Failure("Order already submitted");
            
            Status = OrderStatus.Submitted;
            
            // ✅ Raise domain event - business logic complete
            RaiseDomainEvent(new OrderPlacedEvent(
                Id,
                CustomerId,
                Total,
                _items.Select(i => new OrderItemSnapshot(
                    i.ProductId,
                    i.Quantity,
                    i.Price)).ToList()));
            
            return Result<Unit, string>.Success(Unit.Value);
        }
        
        private void RecalculateTotal()
        {
            Total = _items
                .Select(i => i.Total)
                .Aggregate(Money.Zero, (sum, price) => sum + price);
        }
        
        private void RaiseDomainEvent(DomainEvent domainEvent)
        {
            _domainEvents.Add(domainEvent);
        }
    }
}
```

#### 2. Application Layer (Event Dispatcher)

```csharp
// Application/UseCases/PlaceOrder/PlaceOrderUseCase.cs
namespace MyApp.Application.UseCases.PlaceOrder
{
    public sealed class PlaceOrderUseCase
    {
        private readonly IOrderRepository _orderRepository;
        private readonly IProductRepository _productRepository;
        private readonly IUnitOfWork _unitOfWork;
        private readonly IDomainEventDispatcher _eventDispatcher;
        
        public PlaceOrderUseCase(
            IOrderRepository orderRepository,
            IProductRepository productRepository,
            IUnitOfWork unitOfWork,
            IDomainEventDispatcher eventDispatcher)
        {
            _orderRepository = orderRepository;
            _productRepository = productRepository;
            _unitOfWork = unitOfWork;
            _eventDispatcher = eventDispatcher;
        }
        
        public async Task<Result<OrderId, PlaceOrderError>> ExecuteAsync(
            PlaceOrderCommand command)
        {
            // ✅ Pure orchestration - no side effects
            var order = Order.Create(command.CustomerId);
            
            foreach (var itemCommand in command.Items)
            {
                var productResult = await _productRepository.GetByIdAsync(
                    itemCommand.ProductId);
                
                var product = await productResult.Match(
                    onSome: p => Task.FromResult<Product?>(p),
                    onNone: () => Task.FromResult<Product?>(null));
                
                if (product == null)
                    return Result<OrderId, PlaceOrderError>.Failure(
                        new PlaceOrderError.ProductNotFound(itemCommand.ProductId));
                
                var addResult = order.AddItem(
                    itemCommand.ProductId,
                    itemCommand.Quantity,
                    product.Price);
                
                if (!addResult.IsSuccess)
                    return Result<OrderId, PlaceOrderError>.Failure(
                        new PlaceOrderError.InvalidItem(addResult.Error));
            }
            
            var submitResult = order.Submit();
            if (!submitResult.IsSuccess)
                return Result<OrderId, PlaceOrderError>.Failure(
                    new PlaceOrderError.SubmitFailed(submitResult.Error));
            
            // ✅ Save aggregate
            await _orderRepository.SaveAsync(order);
            
            // ✅ Commit transaction
            await _unitOfWork.CommitAsync();
            
            // ✅ Dispatch events AFTER successful commit
            // Events are immutable facts about what happened
            await _eventDispatcher.DispatchAsync(order.DomainEvents);
            
            order.ClearDomainEvents();
            
            return Result<OrderId, PlaceOrderError>.Success(order.Id);
        }
    }
}

// Application/Events/IDomainEventDispatcher.cs
namespace MyApp.Application.Events
{
    /// <summary>
    /// Dispatches domain events to registered handlers.
    /// </summary>
    public interface IDomainEventDispatcher
    {
        Task DispatchAsync(IEnumerable<DomainEvent> events);
    }
}

// Application/Events/IDomainEventHandler.cs
namespace MyApp.Application.Events
{
    /// <summary>
    /// Handler for a specific domain event type.
    /// </summary>
    public interface IDomainEventHandler<in TEvent> where TEvent : DomainEvent
    {
        Task HandleAsync(TEvent domainEvent);
    }
}
```

#### 3. Event Handlers (Decoupled Side Effects)

```csharp
// Application/EventHandlers/OrderPlaced/SendOrderConfirmationEmailHandler.cs
namespace MyApp.Application.EventHandlers.OrderPlaced
{
    /// <summary>
    /// Handles OrderPlacedEvent by sending confirmation email.
    /// Completely decoupled from order placement logic.
    /// </summary>
    public sealed class SendOrderConfirmationEmailHandler 
        : IDomainEventHandler<OrderPlacedEvent>
    {
        private readonly IEmailService _emailService;
        private readonly ICustomerRepository _customerRepository;
        private readonly ILogger<SendOrderConfirmationEmailHandler> _logger;
        
        public SendOrderConfirmationEmailHandler(
            IEmailService emailService,
            ICustomerRepository customerRepository,
            ILogger<SendOrderConfirmationEmailHandler> logger)
        {
            _emailService = emailService;
            _customerRepository = customerRepository;
            _logger = logger;
        }
        
        public async Task HandleAsync(OrderPlacedEvent domainEvent)
        {
            try
            {
                var customer = await _customerRepository.GetByIdAsync(
                    domainEvent.CustomerId);
                
                await customer.Match(
                    onSome: async c =>
                    {
                        await _emailService.SendOrderConfirmationAsync(
                            domainEvent.OrderId,
                            c.Email,
                            domainEvent.Total);
                    },
                    onNone: () =>
                    {
                        _logger.LogWarning(
                            "Customer {CustomerId} not found for order {OrderId}",
                            domainEvent.CustomerId,
                            domainEvent.OrderId);
                        return Task.CompletedTask;
                    });
            }
            catch (Exception ex)
            {
                // ✅ Handler failures don't break order placement
                _logger.LogError(ex,
                    "Failed to send confirmation email for order {OrderId}",
                    domainEvent.OrderId);
            }
        }
    }
}

// Application/EventHandlers/OrderPlaced/InvalidateCustomerCacheHandler.cs
namespace MyApp.Application.EventHandlers.OrderPlaced
{
    /// <summary>
    /// Handles OrderPlacedEvent by invalidating customer cache.
    /// Separate handler - can be added/removed without touching order logic.
    /// </summary>
    public sealed class InvalidateCustomerCacheHandler 
        : IDomainEventHandler<OrderPlacedEvent>
    {
        private readonly ICacheService _cacheService;
        
        public InvalidateCustomerCacheHandler(ICacheService cacheService)
        {
            _cacheService = cacheService;
        }
        
        public async Task HandleAsync(OrderPlacedEvent domainEvent)
        {
            await _cacheService.InvalidateAsync(
                $"customer:{domainEvent.CustomerId}:orders");
        }
    }
}

// Application/EventHandlers/OrderPlaced/PublishOrderToMessageBusHandler.cs
namespace MyApp.Application.EventHandlers.OrderPlaced
{
    /// <summary>
    /// Publishes order to message bus for other services.
    /// </summary>
    public sealed class PublishOrderToMessageBusHandler 
        : IDomainEventHandler<OrderPlacedEvent>
    {
        private readonly IMessageBus _messageBus;
        
        public PublishOrderToMessageBusHandler(IMessageBus messageBus)
        {
            _messageBus = messageBus;
        }
        
        public async Task HandleAsync(OrderPlacedEvent domainEvent)
        {
            await _messageBus.PublishAsync(new OrderPlacedIntegrationEvent
            {
                OrderId = domainEvent.OrderId.Value,
                CustomerId = domainEvent.CustomerId.Value,
                Total = domainEvent.Total.Amount,
                Currency = domainEvent.Total.Currency.Code,
                PlacedAt = domainEvent.OccurredAt
            });
        }
    }
}

// Application/EventHandlers/OrderPlaced/TrackOrderAnalyticsHandler.cs
namespace MyApp.Application.EventHandlers.OrderPlaced
{
    /// <summary>
    /// Tracks order placement in analytics system.
    /// </summary>
    public sealed class TrackOrderAnalyticsHandler 
        : IDomainEventHandler<OrderPlacedEvent>
    {
        private readonly IAnalyticsService _analyticsService;
        
        public TrackOrderAnalyticsHandler(IAnalyticsService analyticsService)
        {
            _analyticsService = analyticsService;
        }
        
        public async Task HandleAsync(OrderPlacedEvent domainEvent)
        {
            await _analyticsService.TrackEventAsync("order_placed", new
            {
                orderId = domainEvent.OrderId.Value,
                customerId = domainEvent.CustomerId.Value,
                total = domainEvent.Total.Amount,
                itemCount = domainEvent.Items.Count
            });
        }
    }
}
```

#### 4. Infrastructure (Event Dispatcher Implementation)

```csharp
// Infrastructure/Events/DomainEventDispatcher.cs
namespace MyApp.Infrastructure.Events
{
    /// <summary>
    /// Dispatches domain events to all registered handlers.
    /// </summary>
    public sealed class DomainEventDispatcher : IDomainEventDispatcher
    {
        private readonly IServiceProvider _serviceProvider;
        private readonly ILogger<DomainEventDispatcher> _logger;
        
        public DomainEventDispatcher(
            IServiceProvider serviceProvider,
            ILogger<DomainEventDispatcher> logger)
        {
            _serviceProvider = serviceProvider;
            _logger = logger;
        }
        
        public async Task DispatchAsync(IEnumerable<DomainEvent> events)
        {
            foreach (var domainEvent in events)
            {
                _logger.LogInformation(
                    "Dispatching domain event {EventType} {EventId}",
                    domainEvent.GetType().Name,
                    domainEvent.EventId);
                
                await DispatchEventAsync(domainEvent);
            }
        }
        
        private async Task DispatchEventAsync(DomainEvent domainEvent)
        {
            var eventType = domainEvent.GetType();
            var handlerType = typeof(IDomainEventHandler<>).MakeGenericType(eventType);
            
            // Get all handlers for this event type
            var handlers = _serviceProvider.GetServices(handlerType);
            
            // Execute all handlers
            var tasks = handlers
                .Cast<object>()
                .Select(handler =>
                {
                    var method = handlerType.GetMethod(nameof(IDomainEventHandler<DomainEvent>.HandleAsync));
                    return (Task)method!.Invoke(handler, new object[] { domainEvent })!;
                });
            
            await Task.WhenAll(tasks);
        }
    }
}

// Infrastructure/DependencyInjection.cs - Handler Registration
namespace MyApp.Infrastructure
{
    public static class DependencyInjection
    {
        public static IServiceCollection AddDomainEvents(
            this IServiceCollection services)
        {
            // Register dispatcher
            services.AddScoped<IDomainEventDispatcher, DomainEventDispatcher>();
            
            // Register all event handlers
            services.AddScoped<IDomainEventHandler<OrderPlacedEvent>, 
                SendOrderConfirmationEmailHandler>();
            services.AddScoped<IDomainEventHandler<OrderPlacedEvent>, 
                InvalidateCustomerCacheHandler>();
            services.AddScoped<IDomainEventHandler<OrderPlacedEvent>, 
                PublishOrderToMessageBusHandler>();
            services.AddScoped<IDomainEventHandler<OrderPlacedEvent>, 
                TrackOrderAnalyticsHandler>();
            
            return services;
        }
    }
}
```

## Benefits

1. **Decoupling**: Side effects separated from core business logic
2. **Testability**: Test order placement without email, analytics, cache
3. **Extensibility**: Add new handlers without modifying existing code (Open/Closed Principle)
4. **Failure Isolation**: Handler failures don't break core business operations
5. **Clarity**: Events document what happened in the domain

## Testing

```csharp
// Test domain logic without any handlers
[Fact]
public void Order_Submit_RaisesOrderPlacedEvent()
{
    var order = Order.Create(new CustomerId(Guid.NewGuid()));
    order.AddItem(new ProductId(Guid.NewGuid()), 2, Money.USD(10));
    
    var result = order.Submit();
    
    Assert.True(result.IsSuccess);
    Assert.Single(order.DomainEvents);
    Assert.IsType<OrderPlacedEvent>(order.DomainEvents.First());
}

// Test handler in isolation
[Fact]
public async Task SendOrderConfirmationEmailHandler_SendsEmail()
{
    var fakeEmailService = new FakeEmailService();
    var fakeCustomerRepo = new FakeCustomerRepository();
    fakeCustomerRepo.AddCustomer(new Customer(/* ... */));
    
    var handler = new SendOrderConfirmationEmailHandler(
        fakeEmailService,
        fakeCustomerRepo,
        NullLogger<SendOrderConfirmationEmailHandler>.Instance);
    
    var @event = new OrderPlacedEvent(/* ... */);
    
    await handler.HandleAsync(@event);
    
    Assert.Single(fakeEmailService.SentEmails);
}
```

## Why It's a Problem

1. **Tight coupling**: Side effects mixed with business logic
2. **Hard to test**: Can't test business logic without infrastructure
3. **Hard to extend**: Adding side effects requires modifying core logic
4. **Unclear failures**: What happens when email fails after order is saved?

## Symptoms

- Use cases have many dependencies (email, cache, analytics, message bus)
- Single method does business logic + 5 side effects
- Can't test order placement without mocking everything
- Adding new side effects requires changing existing code

## See Also

- [Domain Events](./domain-events.md) — existing pattern
- [Clean Architecture](./clean-architecture.md) — layered architecture
- [Use Cases](./use-cases.md) — orchestrating domain logic
- [Aggregate Roots](./aggregate-roots.md) — publishing events
