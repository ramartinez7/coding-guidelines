# Query Objects (Read-Side Abstraction)

> Encapsulate database queries in dedicated query objects to separate data retrieval concerns from controllers and keep read-side logic organized and testable.

## Problem

When database queries are scattered throughout controllers or use cases, they become hard to test, reuse, and optimize. Query logic mixes with HTTP concerns, making the codebase rigid and difficult to maintain.

## Core Concept

**Query Objects** encapsulate specific data retrieval operations:
- Accept parameters through constructor or method
- Return DTOs optimized for the UI/API
- Contain NO business logic (read-only)
- Can be tested independently
- Can be optimized without affecting callers

## Example

### ❌ Before (Queries in Controllers)

```csharp
[ApiController]
[Route("api/orders")]
public class OrdersController : ControllerBase
{
    private readonly IDbConnection _connection;
    
    [HttpGet("customer/{customerId}")]
    public async Task<IActionResult> GetCustomerOrders(Guid customerId)
    {
        // ❌ Raw SQL in controller
        var sql = @"
            SELECT o.Id, o.OrderDate, o.Total, c.Name AS CustomerName
            FROM Orders o
            JOIN Customers c ON o.CustomerId = c.Id
            WHERE o.CustomerId = @CustomerId";
        
        var orders = await _connection.QueryAsync<OrderDto>(sql, 
            new { CustomerId = customerId });
        
        return Ok(orders);
    }
    
    [HttpGet("{orderId}/details")]
    public async Task<IActionResult> GetOrderDetails(Guid orderId)
    {
        // ❌ Duplicate query logic
        var sql = @"
            SELECT o.Id, o.OrderDate, o.Total,
                   oi.ProductId, p.Name AS ProductName, oi.Quantity
            FROM Orders o
            JOIN OrderItems oi ON o.Id = oi.OrderId
            JOIN Products p ON oi.ProductId = p.Id
            WHERE o.Id = @OrderId";
        
        var order = await _connection.QueryAsync<OrderDetailDto>(sql,
            new { OrderId = orderId });
        
        return Ok(order);
    }
}
```

### ✅ After (Query Objects)

```csharp
// Application/Queries/GetCustomerOrders/GetCustomerOrdersQuery.cs
namespace MyApp.Application.Queries.GetCustomerOrders
{
    /// <summary>
    /// Query object for retrieving customer orders.
    /// </summary>
    public sealed record GetCustomerOrdersQuery(
        CustomerId CustomerId,
        int PageNumber = 1,
        int PageSize = 20,
        OrderSortBy SortBy = OrderSortBy.DateDescending);
    
    public enum OrderSortBy
    {
        DateAscending,
        DateDescending,
        TotalAscending,
        TotalDescending
    }
    
    /// <summary>
    /// Query handler executes the query.
    /// </summary>
    public sealed class GetCustomerOrdersQueryHandler
    {
        private readonly IDbConnection _connection;
        
        public GetCustomerOrdersQueryHandler(IDbConnection connection)
        {
            _connection = connection;
        }
        
        public async Task<PagedResult<CustomerOrderDto>> ExecuteAsync(
            GetCustomerOrdersQuery query)
        {
            var orderBy = query.SortBy switch
            {
                OrderSortBy.DateAscending => "o.OrderDate ASC",
                OrderSortBy.DateDescending => "o.OrderDate DESC",
                OrderSortBy.TotalAscending => "o.Total ASC",
                OrderSortBy.TotalDescending => "o.Total DESC",
                _ => "o.OrderDate DESC"
            };
            
            var sql = $@"
                SELECT 
                    o.Id AS OrderId,
                    o.OrderDate,
                    o.Total,
                    o.Currency,
                    c.Name AS CustomerName,
                    o.Status
                FROM Orders o
                JOIN Customers c ON o.CustomerId = c.Id
                WHERE o.CustomerId = @CustomerId
                ORDER BY {orderBy}
                OFFSET @Offset ROWS
                FETCH NEXT @PageSize ROWS ONLY";
            
            var offset = (query.PageNumber - 1) * query.PageSize;
            
            var orders = await _connection.QueryAsync<CustomerOrderDto>(sql, new
            {
                CustomerId = query.CustomerId.Value,
                Offset = offset,
                PageSize = query.PageSize
            });
            
            var totalCount = await _connection.ExecuteScalarAsync<int>(
                "SELECT COUNT(*) FROM Orders WHERE CustomerId = @CustomerId",
                new { CustomerId = query.CustomerId.Value });
            
            return new PagedResult<CustomerOrderDto>(
                orders.ToList(),
                totalCount,
                query.PageNumber,
                query.PageSize);
        }
    }
    
    public sealed record CustomerOrderDto(
        Guid OrderId,
        DateTime OrderDate,
        decimal Total,
        string Currency,
        string CustomerName,
        string Status);
}

// Application/Queries/GetOrderDetails/GetOrderDetailsQuery.cs
namespace MyApp.Application.Queries.GetOrderDetails
{
    public sealed record GetOrderDetailsQuery(OrderId OrderId);
    
    public sealed class GetOrderDetailsQueryHandler
    {
        private readonly IDbConnection _connection;
        
        public GetOrderDetailsQueryHandler(IDbConnection connection)
        {
            _connection = connection;
        }
        
        public async Task<Option<OrderDetailDto>> ExecuteAsync(
            GetOrderDetailsQuery query)
        {
            var sql = @"
                SELECT 
                    o.Id AS OrderId,
                    o.OrderDate,
                    o.Total,
                    o.Currency,
                    o.Status,
                    c.Name AS CustomerName,
                    c.Email AS CustomerEmail,
                    oi.ProductId,
                    p.Name AS ProductName,
                    oi.Quantity,
                    oi.Price
                FROM Orders o
                JOIN Customers c ON o.CustomerId = c.Id
                LEFT JOIN OrderItems oi ON o.Id = oi.OrderId
                LEFT JOIN Products p ON oi.ProductId = p.Id
                WHERE o.Id = @OrderId";
            
            OrderDetailDto? orderDetail = null;
            var items = new List<OrderItemDto>();
            
            await _connection.QueryAsync<dynamic>(sql,
                new { OrderId = query.OrderId.Value },
                (row) =>
                {
                    if (orderDetail == null)
                    {
                        orderDetail = new OrderDetailDto(
                            row.OrderId,
                            row.OrderDate,
                            row.Total,
                            row.Currency,
                            row.Status,
                            row.CustomerName,
                            row.CustomerEmail,
                            items);
                    }
                    
                    if (row.ProductId != null)
                    {
                        items.Add(new OrderItemDto(
                            row.ProductId,
                            row.ProductName,
                            row.Quantity,
                            row.Price));
                    }
                    
                    return row;
                });
            
            return orderDetail != null
                ? Option<OrderDetailDto>.Some(orderDetail)
                : Option<OrderDetailDto>.None;
        }
    }
    
    public sealed record OrderDetailDto(
        Guid OrderId,
        DateTime OrderDate,
        decimal Total,
        string Currency,
        string Status,
        string CustomerName,
        string CustomerEmail,
        List<OrderItemDto> Items);
    
    public sealed record OrderItemDto(
        Guid ProductId,
        string ProductName,
        int Quantity,
        decimal Price);
}

// WebApi/Controllers/OrdersController.cs
namespace MyApp.WebApi.Controllers
{
    [ApiController]
    [Route("api/orders")]
    public class OrdersController : ControllerBase
    {
        private readonly GetCustomerOrdersQueryHandler _getCustomerOrdersHandler;
        private readonly GetOrderDetailsQueryHandler _getOrderDetailsHandler;
        
        public OrdersController(
            GetCustomerOrdersQueryHandler getCustomerOrdersHandler,
            GetOrderDetailsQueryHandler getOrderDetailsHandler)
        {
            _getCustomerOrdersHandler = getCustomerOrdersHandler;
            _getOrderDetailsHandler = getOrderDetailsHandler;
        }
        
        // ✅ Controller just delegates to query handler
        [HttpGet("customer/{customerId}")]
        public async Task<IActionResult> GetCustomerOrders(
            Guid customerId,
            [FromQuery] int page = 1,
            [FromQuery] int pageSize = 20,
            [FromQuery] string sortBy = "date-desc")
        {
            var sortByEnum = sortBy switch
            {
                "date-asc" => OrderSortBy.DateAscending,
                "date-desc" => OrderSortBy.DateDescending,
                "total-asc" => OrderSortBy.TotalAscending,
                "total-desc" => OrderSortBy.TotalDescending,
                _ => OrderSortBy.DateDescending
            };
            
            var query = new GetCustomerOrdersQuery(
                new CustomerId(customerId),
                page,
                pageSize,
                sortByEnum);
            
            var result = await _getCustomerOrdersHandler.ExecuteAsync(query);
            
            return Ok(result);
        }
        
        [HttpGet("{orderId}/details")]
        public async Task<IActionResult> GetOrderDetails(Guid orderId)
        {
            var query = new GetOrderDetailsQuery(new OrderId(orderId));
            var result = await _getOrderDetailsHandler.ExecuteAsync(query);
            
            return result.Match(
                onSome: order => Ok(order),
                onNone: () => NotFound());
        }
    }
}
```

## Benefits

1. **Testability**: Query objects can be tested independently
2. **Reusability**: Same query can be used from multiple controllers
3. **Optimization**: Query logic centralized and easy to optimize
4. **Clear Intent**: Query name describes what data is retrieved
5. **Type Safety**: Parameters and results strongly typed

## Testing

```csharp
[Fact]
public async Task GetCustomerOrdersQuery_ReturnsOrdersForCustomer()
{
    var connection = CreateTestConnection();
    var handler = new GetCustomerOrdersQueryHandler(connection);
    
    var query = new GetCustomerOrdersQuery(
        new CustomerId(TestData.CustomerId),
        pageNumber: 1,
        pageSize: 10);
    
    var result = await handler.ExecuteAsync(query);
    
    Assert.NotEmpty(result.Items);
    Assert.All(result.Items, order => 
        Assert.Equal(TestData.CustomerName, order.CustomerName));
}
```

## See Also

- [Read Model vs Write Model](./read-write-model-separation.md) — CQRS pattern
- [Repository Pattern](./repository-pattern.md) — write-side abstraction
- [Specification Pattern](./specification-pattern.md) — composable query logic
