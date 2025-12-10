# Type-Safe CQRS (Command Query Responsibility Segregation with Types)

> Mixing reads and writes in the same model creates complexity—separate commands and queries using types to enforce the distinction at compile time.

## Problem

When the same model serves both read and write operations, it becomes bloated with properties and methods for both concerns. Type safety can't enforce the distinction between queries (which shouldn't modify state) and commands (which do).

## Example

### ❌ Before

```csharp
// Mixed read/write concerns
public class OrderService
{
    public async Task<Order> GetOrder(int id)  // Query
    {
        return await _db.Orders.FindAsync(id);
    }
    
    public async Task<Order> CreateOrder(Order order)  // Command
    {
        _db.Orders.Add(order);
        await _db.SaveChangesAsync();
        return order;
    }
    
    public async Task<List<Order>> GetCustomerOrders(int customerId)  // Query
    {
        return await _db.Orders
            .Where(o => o.CustomerId == customerId)
            .ToListAsync();
    }
    
    public async Task UpdateOrder(Order order)  // Command
    {
        _db.Orders.Update(order);
        await _db.SaveChangesAsync();
    }
}
```

**Problems:**
- No distinction between commands and queries
- Same model for reads and writes
- Can't optimize reads separately from writes
- No type-level guarantee queries don't modify state

### ✅ After

```csharp
// ============================================
// COMMANDS (Write Model)
// ============================================
namespace Orders.Commands
{
    /// <summary>
    /// Base for all commands—represents intent to change state.
    /// </summary>
    public abstract record Command;
    
    public sealed record CreateOrderCommand(
        CustomerId CustomerId,
        IReadOnlyList<OrderLineDto> Lines) : Command;
    
    public sealed record OrderLineDto(
        ProductId ProductId,
        int Quantity,
        decimal UnitPrice);
    
    public sealed record CancelOrderCommand(
        OrderId OrderId,
        string Reason) : Command;
    
    public sealed record ShipOrderCommand(
        OrderId OrderId,
        string TrackingNumber) : Command;
    
    /// <summary>
    /// Command handler—executes command, returns result.
    /// </summary>
    public interface ICommandHandler<TCommand, TResult>
        where TCommand : Command
    {
        Task<Result<TResult, string>> Handle(TCommand command);
    }
    
    public sealed class CreateOrderHandler : ICommandHandler<CreateOrderCommand, OrderId>
    {
        private readonly IOrderRepository _repo;
        private readonly IEventPublisher _publisher;
        
        public async Task<Result<OrderId, string>> Handle(CreateOrderCommand cmd)
        {
            var order = Order.Create(OrderId.New(), cmd.CustomerId);
            
            foreach (var line in cmd.Lines)
            {
                var result = order.AddLine(
                    line.ProductId,
                    line.Quantity,
                    Money.USD(line.UnitPrice));
                
                if (!result.IsSuccess)
                    return Result<OrderId, string>.Failure(result.Error);
            }
            
            await _repo.SaveAsync(order);
            await _publisher.PublishAsync(new OrderCreatedEvent(order.Id));
            
            return Result<OrderId, string>.Success(order.Id);
        }
    }
}

// ============================================
// QUERIES (Read Model)
// ============================================
namespace Orders.Queries
{
    /// <summary>
    /// Base for all queries—represents request for data.
    /// Pure functions—no side effects.
    /// </summary>
    public abstract record Query;
    
    public sealed record GetOrderQuery(OrderId OrderId) : Query;
    
    public sealed record GetCustomerOrdersQuery(
        CustomerId CustomerId,
        int Page,
        int PageSize) : Query;
    
    public sealed record GetOrderSummaryQuery(OrderId OrderId) : Query;
    
    /// <summary>
    /// Read model—optimized for queries, denormalized.
    /// </summary>
    public sealed record OrderDto
    {
        public string OrderId { get; init; }
        public string CustomerId { get; init; }
        public string CustomerName { get; init; }
        public decimal Total { get; init; }
        public string Status { get; init; }
        public DateTime CreatedAt { get; init; }
        public List<OrderLineDto> Lines { get; init; }
    }
    
    public sealed record OrderLineDto
    {
        public string ProductId { get; init; }
        public string ProductName { get; init; }
        public int Quantity { get; init; }
        public decimal UnitPrice { get; init; }
        public decimal LineTotal { get; init; }
    }
    
    public sealed record OrderSummaryDto
    {
        public string OrderId { get; init; }
        public decimal Total { get; init; }
        public string Status { get; init; }
        public int ItemCount { get; init; }
    }
    
    /// <summary>
    /// Query handler—reads data, returns DTO.
    /// No database writes allowed.
    /// </summary>
    public interface IQueryHandler<TQuery, TResult>
        where TQuery : Query
    {
        Task<Result<TResult, string>> Handle(TQuery query);
    }
    
    public sealed class GetOrderHandler : IQueryHandler<GetOrderQuery, OrderDto>
    {
        private readonly IOrderReadRepository _repo;
        
        public async Task<Result<OrderDto, string>> Handle(GetOrderQuery query)
        {
            // Read from optimized read model
            var order = await _repo.GetByIdAsync(query.OrderId);
            
            if (order == null)
                return Result<OrderDto, string>.Failure("Order not found");
            
            return Result<OrderDto, string>.Success(order);
        }
    }
    
    public sealed class GetCustomerOrdersHandler 
        : IQueryHandler<GetCustomerOrdersQuery, PagedResult<OrderSummaryDto>>
    {
        private readonly IOrderReadRepository _repo;
        
        public async Task<Result<PagedResult<OrderSummaryDto>, string>> Handle(
            GetCustomerOrdersQuery query)
        {
            var orders = await _repo.GetCustomerOrdersAsync(
                query.CustomerId,
                query.Page,
                query.PageSize);
            
            return Result<PagedResult<OrderSummaryDto>, string>.Success(orders);
        }
    }
}

// ============================================
// READ MODEL PROJECTION
// ============================================
namespace Orders.Projections
{
    /// <summary>
    /// Event handler updates read model when commands execute.
    /// Eventual consistency between write and read models.
    /// </summary>
    public sealed class OrderProjection : IEventHandler<OrderCreatedEvent>
    {
        private readonly IOrderReadRepository _readRepo;
        private readonly ICustomerRepository _customerRepo;
        
        public async Task HandleAsync(OrderCreatedEvent evt)
        {
            // Build denormalized read model
            var customer = await _customerRepo.GetByIdAsync(evt.CustomerId);
            
            var orderDto = new OrderDto
            {
                OrderId = evt.OrderId.Value.ToString(),
                CustomerId = evt.CustomerId.Value.ToString(),
                CustomerName = customer.Name.Value,
                Total = evt.Total.Amount,
                Status = "Created",
                CreatedAt = evt.CreatedAt,
                Lines = evt.Lines.Select(l => new OrderLineDto
                {
                    ProductId = l.ProductId.Value.ToString(),
                    ProductName = l.ProductName,
                    Quantity = l.Quantity,
                    UnitPrice = l.UnitPrice.Amount,
                    LineTotal = (l.UnitPrice * l.Quantity).Amount
                }).ToList()
            };
            
            await _readRepo.SaveAsync(orderDto);
        }
    }
}

// ============================================
// MEDIATOR PATTERN
// ============================================
namespace Orders.Application
{
    public interface IMediator
    {
        Task<Result<TResult, string>> Send<TResult>(Command command);
        Task<Result<TResult, string>> Query<TResult>(Query query);
    }
    
    public sealed class Mediator : IMediator
    {
        private readonly IServiceProvider _services;
        
        public async Task<Result<TResult, string>> Send<TResult>(Command command)
        {
            var handlerType = typeof(ICommandHandler<,>)
                .MakeGenericType(command.GetType(), typeof(TResult));
            
            dynamic handler = _services.GetRequiredService(handlerType);
            return await handler.Handle((dynamic)command);
        }
        
        public async Task<Result<TResult, string>> Query<TResult>(Query query)
        {
            var handlerType = typeof(IQueryHandler<,>)
                .MakeGenericType(query.GetType(), typeof(TResult));
            
            dynamic handler = _services.GetRequiredService(handlerType);
            return await handler.Handle((dynamic)query);
        }
    }
}

// Usage
var mediator = serviceProvider.GetRequiredService<IMediator>();

// Execute command (writes)
var createCommand = new CreateOrderCommand(customerId, lines);
var createResult = await mediator.Send<OrderId>(createCommand);

// Execute query (reads)
var getQuery = new GetOrderQuery(orderId);
var queryResult = await mediator.Query<OrderDto>(getQuery);
```

## Benefits

- **Separation**: Clear distinction between reads and writes
- **Optimization**: Optimize read and write models independently
- **Scalability**: Scale reads and writes separately
- **Type safety**: Compiler enforces query/command distinction

## See Also

- [Domain Events](./domain-events.md)
- [Type-Safe Workflow Modeling](./type-safe-workflow-modeling.md)
- [Domain Model Isolation](./domain-model-isolation.md)
