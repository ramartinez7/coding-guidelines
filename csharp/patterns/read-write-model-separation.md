# Read Model vs Write Model (CQRS Application Layer)

> Separate read operations (queries) from write operations (commands) to optimize each for their specific purpose—commands validate and modify state, queries return pre-computed views without business logic.

## Problem

Traditional CRUD applications use the same model for reading and writing data. This creates friction because reads and writes have fundamentally different requirements:
- **Writes** need validation, business rules, and transactional consistency
- **Reads** need speed, denormalization, and specific shapes for UI/API

Using the same model for both leads to bloated entities, complex queries, and N+1 problems.

## Core Concept

**CQRS (Command Query Responsibility Segregation)** separates the application layer into two distinct paths:

```
┌─────────────────────────────────────────┐
│         Command Side (Write)            │
│  - Validate input                       │
│  - Execute business logic               │
│  - Modify state                         │
│  - Raise domain events                  │
│  - Return success/failure               │
└──────────┬──────────────────────────────┘
           │
           │ Updates
           ▼
    ┌────────────┐
    │  Write DB  │ (Normalized, Transactional)
    └────────────┘
           │
           │ Domain Events
           ▼
    ┌────────────┐
    │  Read DB   │ (Denormalized, Optimized for queries)
    └────────────┘
           ▲
           │ Queries
┌──────────┴──────────────────────────────┐
│         Query Side (Read)               │
│  - No business logic                    │
│  - No validation                        │
│  - Return DTOs directly                 │
│  - Optimized queries                    │
└─────────────────────────────────────────┘
```

## Example

### ❌ Before (Single Model for Reads and Writes)

```csharp
// Same entity used for both reads and writes
public class OrderController : ControllerBase
{
    private readonly IOrderRepository _orderRepository;
    
    // ❌ Write operation loads full entity
    [HttpPost]
    public async Task<IActionResult> PlaceOrder(PlaceOrderRequest request)
    {
        var order = new Order
        {
            CustomerId = request.CustomerId,
            // ... set all properties
        };
        
        // Business logic mixed with persistence
        order.Submit();
        
        await _orderRepository.SaveAsync(order);
        return Ok(order);  // ❌ Returning domain entity directly
    }
    
    // ❌ Read operation has to navigate domain entities
    [HttpGet("{customerId}/orders")]
    public async Task<IActionResult> GetCustomerOrders(Guid customerId)
    {
        // ❌ Loads full entities with all relationships
        var orders = await _orderRepository.GetByCustomerIdAsync(customerId);
        
        // ❌ N+1 queries to get customer names, product names, etc.
        var dtos = orders.Select(o => new OrderListDto
        {
            Id = o.Id,
            CustomerName = o.Customer.Name,  // N+1
            Total = o.Total,
            Items = o.Items.Select(i => new OrderItemDto
            {
                ProductName = i.Product.Name,  // N+1
                Quantity = i.Quantity
            }).ToList()
        }).ToList();
        
        return Ok(dtos);
    }
}
```

**Problems:**
- Write operations load more data than needed
- Read operations trigger N+1 queries
- Same entity serves two masters with conflicting needs
- Domain entities exposed as DTOs (tight coupling)

### ✅ After (CQRS with Separate Read and Write Models)

#### 1. Command Side (Write Model)

```csharp
// Application/Commands/PlaceOrder/PlaceOrderCommand.cs
namespace MyApp.Application.Commands.PlaceOrder
{
    /// <summary>
    /// Command for write operation.
    /// Represents intent to change state.
    /// </summary>
    public sealed record PlaceOrderCommand(
        CustomerId CustomerId,
        List<OrderItemCommand> Items);
    
    public sealed record OrderItemCommand(ProductId ProductId, int Quantity);
}

// Application/Commands/PlaceOrder/PlaceOrderCommandHandler.cs
namespace MyApp.Application.Commands.PlaceOrder
{
    /// <summary>
    /// Handler for write operations.
    /// Contains business logic, validation, and state changes.
    /// </summary>
    public sealed class PlaceOrderCommandHandler
    {
        private readonly IOrderRepository _orderRepository;
        private readonly IProductRepository _productRepository;
        private readonly IUnitOfWork _unitOfWork;
        private readonly IDomainEventDispatcher _eventDispatcher;
        
        public PlaceOrderCommandHandler(
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
        
        public async Task<Result<OrderId, PlaceOrderError>> HandleAsync(
            PlaceOrderCommand command)
        {
            // ✅ Create domain entity
            var order = Order.Create(command.CustomerId);
            
            // ✅ Execute business logic
            foreach (var item in command.Items)
            {
                var product = await _productRepository.GetByIdAsync(item.ProductId);
                
                var addResult = await product.Match(
                    onSome: p => Task.FromResult(order.AddItem(p, item.Quantity)),
                    onNone: () => Task.FromResult(
                        Result<Unit, string>.Failure("Product not found")));
                
                if (!addResult.IsSuccess)
                    return Result<OrderId, PlaceOrderError>.Failure(
                        new PlaceOrderError.InvalidItem(addResult.Error));
            }
            
            var submitResult = order.Submit();
            if (!submitResult.IsSuccess)
                return Result<OrderId, PlaceOrderError>.Failure(
                    new PlaceOrderError.SubmitFailed(submitResult.Error));
            
            // ✅ Persist changes
            await _orderRepository.SaveAsync(order);
            await _unitOfWork.CommitAsync();
            
            // ✅ Dispatch events to update read model
            await _eventDispatcher.DispatchAsync(order.DomainEvents);
            
            // ✅ Return only ID, not full entity
            return Result<OrderId, PlaceOrderError>.Success(order.Id);
        }
    }
    
    public abstract record PlaceOrderError
    {
        public sealed record InvalidItem(string Reason) : PlaceOrderError;
        public sealed record SubmitFailed(string Reason) : PlaceOrderError;
    }
}
```

#### 2. Query Side (Read Model)

```csharp
// Application/Queries/GetCustomerOrders/GetCustomerOrdersQuery.cs
namespace MyApp.Application.Queries.GetCustomerOrders
{
    /// <summary>
    /// Query for read operation.
    /// Specifies what data to retrieve.
    /// </summary>
    public sealed record GetCustomerOrdersQuery(
        CustomerId CustomerId,
        int PageNumber,
        int PageSize);
}

// Application/Queries/GetCustomerOrders/CustomerOrderDto.cs
namespace MyApp.Application.Queries.GetCustomerOrders
{
    /// <summary>
    /// Read model optimized for display.
    /// Denormalized - contains all data needed by UI.
    /// </summary>
    public sealed record CustomerOrderDto(
        Guid OrderId,
        DateTime OrderDate,
        string CustomerName,
        decimal Total,
        string Currency,
        string Status,
        List<OrderItemDto> Items);
    
    public sealed record OrderItemDto(
        Guid ProductId,
        string ProductName,
        int Quantity,
        decimal Price);
}

// Application/Queries/GetCustomerOrders/GetCustomerOrdersQueryHandler.cs
namespace MyApp.Application.Queries.GetCustomerOrders
{
    /// <summary>
    /// Handler for read operations.
    /// NO business logic - just data retrieval and projection.
    /// </summary>
    public sealed class GetCustomerOrdersQueryHandler
    {
        private readonly IOrderReadRepository _orderReadRepository;
        
        public GetCustomerOrdersQueryHandler(IOrderReadRepository orderReadRepository)
        {
            _orderReadRepository = orderReadRepository;
        }
        
        public async Task<PagedResult<CustomerOrderDto>> HandleAsync(
            GetCustomerOrdersQuery query)
        {
            // ✅ Single optimized query, no business logic
            return await _orderReadRepository.GetCustomerOrdersAsync(
                query.CustomerId,
                query.PageNumber,
                query.PageSize);
        }
    }
}

// Application/Queries/IOrderReadRepository.cs
namespace MyApp.Application.Queries
{
    /// <summary>
    /// Read repository returns DTOs, not domain entities.
    /// Optimized for queries.
    /// </summary>
    public interface IOrderReadRepository
    {
        Task<PagedResult<CustomerOrderDto>> GetCustomerOrdersAsync(
            CustomerId customerId,
            int pageNumber,
            int pageSize);
        
        Task<Option<OrderDetailDto>> GetOrderDetailAsync(OrderId orderId);
    }
}
```

#### 3. Read Model Projection (Event Handler)

```csharp
// Application/EventHandlers/OrderPlaced/UpdateOrderReadModelHandler.cs
namespace MyApp.Application.EventHandlers.OrderPlaced
{
    /// <summary>
    /// Updates read model when order is placed.
    /// Keeps read side eventually consistent with write side.
    /// </summary>
    public sealed class UpdateOrderReadModelHandler 
        : IDomainEventHandler<OrderPlacedEvent>
    {
        private readonly IOrderReadModelRepository _readModelRepository;
        
        public UpdateOrderReadModelHandler(
            IOrderReadModelRepository readModelRepository)
        {
            _readModelRepository = readModelRepository;
        }
        
        public async Task HandleAsync(OrderPlacedEvent domainEvent)
        {
            // ✅ Project domain event to read model
            var readModel = new OrderReadModel
            {
                OrderId = domainEvent.OrderId.Value,
                CustomerId = domainEvent.CustomerId.Value,
                OrderDate = domainEvent.OccurredAt,
                Total = domainEvent.Total.Amount,
                Currency = domainEvent.Total.Currency.Code,
                Status = "Submitted",
                Items = domainEvent.Items.Select(i => new OrderItemReadModel
                {
                    ProductId = i.ProductId.Value,
                    Quantity = i.Quantity,
                    Price = i.Price.Amount
                }).ToList()
            };
            
            await _readModelRepository.SaveAsync(readModel);
        }
    }
}
```

#### 4. Infrastructure (Optimized Read Repository)

```csharp
// Infrastructure/Queries/OrderReadRepository.cs
namespace MyApp.Infrastructure.Queries
{
    /// <summary>
    /// Read repository with denormalized queries.
    /// Can use different database than write side.
    /// </summary>
    public sealed class OrderReadRepository : IOrderReadRepository
    {
        private readonly IDbConnection _connection;
        
        public OrderReadRepository(IDbConnection connection)
        {
            _connection = connection;
        }
        
        public async Task<PagedResult<CustomerOrderDto>> GetCustomerOrdersAsync(
            CustomerId customerId,
            int pageNumber,
            int pageSize)
        {
            // ✅ Single denormalized query with JOIN
            var sql = @"
                SELECT 
                    o.Id AS OrderId,
                    o.OrderDate,
                    c.Name AS CustomerName,
                    o.Total,
                    o.Currency,
                    o.Status,
                    oi.ProductId,
                    p.Name AS ProductName,
                    oi.Quantity,
                    oi.Price
                FROM OrderReadModel o
                JOIN CustomerReadModel c ON o.CustomerId = c.Id
                JOIN OrderItemReadModel oi ON o.Id = oi.OrderId
                JOIN ProductReadModel p ON oi.ProductId = p.Id
                WHERE o.CustomerId = @CustomerId
                ORDER BY o.OrderDate DESC
                OFFSET @Offset ROWS
                FETCH NEXT @PageSize ROWS ONLY";
            
            var offset = (pageNumber - 1) * pageSize;
            
            var orderDictionary = new Dictionary<Guid, CustomerOrderDto>();
            
            var orders = await _connection.QueryAsync<dynamic>(sql, new
            {
                CustomerId = customerId.Value,
                Offset = offset,
                PageSize = pageSize
            });
            
            foreach (var row in orders)
            {
                if (!orderDictionary.TryGetValue(row.OrderId, out var order))
                {
                    order = new CustomerOrderDto(
                        row.OrderId,
                        row.OrderDate,
                        row.CustomerName,
                        row.Total,
                        row.Currency,
                        row.Status,
                        new List<OrderItemDto>());
                    
                    orderDictionary.Add(row.OrderId, order);
                }
                
                order.Items.Add(new OrderItemDto(
                    row.ProductId,
                    row.ProductName,
                    row.Quantity,
                    row.Price));
            }
            
            var totalCount = await _connection.ExecuteScalarAsync<int>(
                "SELECT COUNT(*) FROM OrderReadModel WHERE CustomerId = @CustomerId",
                new { CustomerId = customerId.Value });
            
            return new PagedResult<CustomerOrderDto>(
                orderDictionary.Values.ToList(),
                totalCount,
                pageNumber,
                pageSize);
        }
    }
}
```

#### 5. Presentation Layer

```csharp
// WebApi/Controllers/OrdersController.cs
namespace MyApp.WebApi.Controllers
{
    [ApiController]
    [Route("api/orders")]
    public class OrdersController : ControllerBase
    {
        private readonly PlaceOrderCommandHandler _placeOrderHandler;
        private readonly GetCustomerOrdersQueryHandler _getCustomerOrdersHandler;
        
        public OrdersController(
            PlaceOrderCommandHandler placeOrderHandler,
            GetCustomerOrdersQueryHandler getCustomerOrdersHandler)
        {
            _placeOrderHandler = placeOrderHandler;
            _getCustomerOrdersHandler = getCustomerOrdersHandler;
        }
        
        // ✅ Write endpoint - uses command handler
        [HttpPost]
        public async Task<IActionResult> PlaceOrder(
            [FromBody] PlaceOrderRequest request)
        {
            var command = new PlaceOrderCommand(
                new CustomerId(request.CustomerId),
                request.Items.Select(i => new OrderItemCommand(
                    new ProductId(i.ProductId),
                    i.Quantity)).ToList());
            
            var result = await _placeOrderHandler.HandleAsync(command);
            
            return result.Match(
                onSuccess: orderId => CreatedAtAction(
                    nameof(GetOrder),
                    new { id = orderId.Value },
                    new { OrderId = orderId.Value }),
                onFailure: error => BadRequest(new { Error = error.ToString() }));
        }
        
        // ✅ Read endpoint - uses query handler
        [HttpGet("customer/{customerId}")]
        public async Task<IActionResult> GetCustomerOrders(
            Guid customerId,
            [FromQuery] int page = 1,
            [FromQuery] int pageSize = 20)
        {
            var query = new GetCustomerOrdersQuery(
                new CustomerId(customerId),
                page,
                pageSize);
            
            var result = await _getCustomerOrdersHandler.HandleAsync(query);
            
            return Ok(result);
        }
        
        [HttpGet("{id}")]
        public IActionResult GetOrder(Guid id)
        {
            // Implementation for route reference
            return Ok();
        }
    }
}
```

## Benefits

1. **Optimized Reads**: Query denormalized views without N+1 problems
2. **Optimized Writes**: Validate and persist only what changed
3. **Independent Scaling**: Scale read and write databases independently
4. **Simplified Queries**: No business logic in queries, just data retrieval
5. **Different Storage**: Use different databases for reads (MongoDB) and writes (SQL)

## When to Use CQRS

✅ **Use CQRS when:**
- Read and write operations have very different performance characteristics
- UI needs denormalized views that don't match domain model
- Read load is much higher than write load
- You need to scale reads and writes independently

❌ **Don't use CQRS when:**
- Application is simple CRUD with symmetric reads/writes
- Eventual consistency is unacceptable
- Team is unfamiliar with the pattern

## Why It's a Problem

1. **Single model**: Same entity serves reads and writes with conflicting needs
2. **N+1 queries**: Reading aggregates triggers cascading queries
3. **Bloated entities**: Entities carry data only needed for specific queries
4. **Complex queries**: Business logic mixed with data retrieval

## Symptoms

- Entities with dozens of navigation properties
- Include chains: `.Include(o => o.Customer).Include(o => o.Items).ThenInclude(i => i.Product)`
- Performance issues with "simple" queries
- DTOs that look nothing like domain entities

## See Also

- [Clean Architecture](./clean-architecture.md) — architectural foundation
- [Use Cases](./use-cases.md) — command handlers
- [Query Objects](./query-objects.md) — encapsulating queries
- [Command Handlers](./command-handlers.md) — processing commands
- [Domain Events](./domain-events.md) — updating read models
