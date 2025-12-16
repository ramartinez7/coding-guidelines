# Mapping Strategies (Domain-DTO Translation)

> Translate between domain models and DTOs using explicit mapping strategies—keeping domain pure while providing API-friendly representations.

## Problem

Exposing domain entities directly as API DTOs couples external contracts to internal domain models. When DTOs and domain models mix, the domain becomes constrained by API requirements.

## Core Concept

**Explicit Mapping** between layers:
- **DTOs** (Data Transfer Objects) for API contracts
- **Domain Models** for business logic
- **Persistence Models** for database storage
- **Mappers** translate between them

```
HTTP Request (DTO) → Mapper → Domain Model → Business Logic
                                      ↓
                                  Mapper
                                      ↓
                          Persistence Model → Database
```

## Example

### ❌ Before (Domain Exposed as DTO)

```csharp
// ❌ Domain entity used directly in API
[HttpGet("{id}")]
public async Task<IActionResult> GetOrder(Guid id)
{
    var order = await _orderRepository.GetByIdAsync(new OrderId(id));
    
    // ❌ Exposing domain entity directly
    return Ok(order);  // Serializes private fields, exposes internals
}

// ❌ API request modifies domain directly
[HttpPost]
public async Task<IActionResult> CreateOrder([FromBody] Order order)
{
    // ❌ Bypasses domain invariants
    await _orderRepository.SaveAsync(order);
    return Ok();
}
```

### ✅ After (Explicit Mapping)

```csharp
// WebApi/DTOs/OrderDto.cs
namespace MyApp.WebApi.DTOs
{
    /// <summary>
    /// DTO for API responses - stable external contract.
    /// </summary>
    public sealed record OrderDto(
        Guid Id,
        Guid CustomerId,
        string CustomerName,
        decimal TotalAmount,
        string Currency,
        string Status,
        DateTime CreatedAt,
        List<OrderItemDto> Items);
    
    public sealed record OrderItemDto(
        Guid ProductId,
        string ProductName,
        int Quantity,
        decimal Price);
}

// WebApi/DTOs/CreateOrderRequest.cs
namespace MyApp.WebApi.DTOs
{
    /// <summary>
    /// DTO for API requests - validation attributes, no business logic.
    /// </summary>
    public sealed record CreateOrderRequest(
        [Required] Guid CustomerId,
        [Required, MinLength(1)] List<CreateOrderItemRequest> Items);
    
    public sealed record CreateOrderItemRequest(
        [Required] Guid ProductId,
        [Range(1, int.MaxValue)] int Quantity);
}

// Application/Mappers/OrderMapper.cs
namespace MyApp.Application.Mappers
{
    /// <summary>
    /// Maps between DTOs and domain models.
    /// </summary>
    public static class OrderMapper
    {
        public static OrderDto ToDto(Order order, Customer customer)
        {
            return new OrderDto(
                order.Id.Value,
                order.CustomerId.Value,
                customer.Name,
                order.Total.Amount,
                order.Total.Currency.Code,
                order.Status.ToString(),
                order.CreatedAt,
                order.Items.Select(i => new OrderItemDto(
                    i.ProductId.Value,
                    i.Product.Name,  // Assuming eager loading
                    i.Quantity,
                    i.Price.Amount)).ToList());
        }
        
        public static PlaceOrderCommand ToCommand(CreateOrderRequest request)
        {
            return new PlaceOrderCommand(
                new CustomerId(request.CustomerId),
                request.Items.Select(i => new OrderItemCommand(
                    new ProductId(i.ProductId),
                    i.Quantity)).ToList());
        }
    }
}

// WebApi/Controllers/OrdersController.cs
namespace MyApp.WebApi.Controllers
{
    [ApiController]
    [Route("api/orders")]
    public class OrdersController : ControllerBase
    {
        private readonly PlaceOrderCommandHandler _placeOrderHandler;
        private readonly GetOrderDetailsQueryHandler _getOrderDetailsHandler;
        
        // ✅ POST - Map request DTO to command
        [HttpPost]
        public async Task<IActionResult> CreateOrder(
            [FromBody] CreateOrderRequest request)
        {
            // ✅ Map DTO to command
            var command = OrderMapper.ToCommand(request);
            
            var result = await _placeOrderHandler.HandleAsync(command);
            
            return result.Match(
                onSuccess: orderId => CreatedAtAction(
                    nameof(GetOrder),
                    new { id = orderId.Value },
                    new { OrderId = orderId.Value }),
                onFailure: error => BadRequest(new { Error = error.ToString() }));
        }
        
        // ✅ GET - Map domain model to response DTO
        [HttpGet("{id}")]
        public async Task<IActionResult> GetOrder(Guid id)
        {
            var query = new GetOrderDetailsQuery(new OrderId(id));
            var result = await _getOrderDetailsHandler.HandleAsync(query);
            
            return result.Match(
                onSome: orderDetail =>
                {
                    // ✅ Query handler already returns DTO
                    return Ok(orderDetail);
                },
                onNone: () => NotFound());
        }
    }
}
```

## Mapping Strategies

### 1. Manual Mapping (Explicit)

```csharp
public static class OrderMapper
{
    public static OrderDto ToDto(Order order)
    {
        return new OrderDto(
            order.Id.Value,
            order.CustomerId.Value,
            order.Total.Amount,
            order.Total.Currency.Code);
    }
}
```

**Pros**: Full control, type-safe, explicit  
**Cons**: Verbose, manual updates when models change

### 2. AutoMapper (Convention-Based)

```csharp
// Profile configuration
public class OrderMappingProfile : Profile
{
    public OrderMappingProfile()
    {
        CreateMap<Order, OrderDto>()
            .ForMember(dest => dest.Id, opt => opt.MapFrom(src => src.Id.Value))
            .ForMember(dest => dest.CustomerId, opt => opt.MapFrom(src => src.CustomerId.Value))
            .ForMember(dest => dest.TotalAmount, opt => opt.MapFrom(src => src.Total.Amount))
            .ForMember(dest => dest.Currency, opt => opt.MapFrom(src => src.Total.Currency.Code));
    }
}

// Usage
var orderDto = _mapper.Map<OrderDto>(order);
```

**Pros**: Less code, automatic property matching  
**Cons**: Magic, runtime errors, harder to debug

### 3. Extension Methods

```csharp
public static class OrderExtensions
{
    public static OrderDto ToDto(this Order order)
    {
        return new OrderDto(
            order.Id.Value,
            order.CustomerId.Value,
            order.Total.Amount,
            order.Total.Currency.Code);
    }
    
    public static PlaceOrderCommand ToCommand(this CreateOrderRequest request)
    {
        return new PlaceOrderCommand(
            new CustomerId(request.CustomerId),
            request.Items.Select(i => i.ToCommand()).ToList());
    }
}

// Usage
var dto = order.ToDto();
var command = request.ToCommand();
```

**Pros**: Fluent, discoverable, type-safe  
**Cons**: Can clutter intellisense

## Benefits

1. **Stable API**: Change domain without breaking API contracts
2. **Encapsulation**: Domain internals not exposed
3. **Validation**: Different validation rules for API vs domain
4. **Versioning**: Support multiple API versions with same domain

## Mapping Responsibilities

| Layer | Responsibility |
|-------|----------------|
| **Controller** | HTTP ↔ DTO |
| **Mapper** | DTO ↔ Command/Query |
| **Use Case** | Command ↔ Domain Model |
| **Repository** | Domain Model ↔ Persistence Model |

## See Also

- [DTO vs Domain Boundary](./dto-domain-boundary.md) — separating contracts
- [Clean Architecture](./clean-architecture.md) — layer dependencies
- [Persistence Ignorance](./persistence-ignorance.md) — domain-persistence mapping
