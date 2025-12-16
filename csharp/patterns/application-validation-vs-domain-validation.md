# Application Layer Validation vs Domain Validation

> Application layer validates input and authorization, while domain layer enforces business invariants—separating concerns and preventing invalid domain objects from existing.

## Problem

When validation responsibilities blur between layers, business rules leak into controllers, or input validation pollutes domain entities.

## Core Concept

**Two Types of Validation:**

**Application Validation:**
- Input format (required fields, ranges, regex)
- Authorization (can user perform this action?)
- External constraints (does customer exist?)
- Fails fast before domain logic executes

**Domain Validation:**
- Business invariants (order total must be positive)
- Domain rules (cannot cancel shipped order)
- Consistency rules (item quantity cannot exceed stock)
- Enforced by domain entities/services

```
Request → Application Validation → Domain Validation → Success
            ↓ Invalid Input          ↓ Business Rule
          400 Bad Request          Domain Error in Result<T>
```

## Example

### ❌ Before (Mixed Validation)

```csharp
// ❌ Business logic mixed with input validation in controller
[HttpPost]
public async Task<IActionResult> PlaceOrder(PlaceOrderRequest request)
{
    // ❌ Input validation in controller
    if (string.IsNullOrEmpty(request.CustomerId))
        return BadRequest("CustomerId required");
    
    if (request.Items.Count == 0)
        return BadRequest("At least one item required");
    
    // ❌ Business rule in controller
    if (request.Items.Sum(i => i.Quantity) > 100)
        return BadRequest("Cannot order more than 100 items");
    
    var order = Order.Create(request.CustomerId);
    await _orderRepository.SaveAsync(order);
    return Ok();
}
```

### ✅ After (Layered Validation)

```csharp
// Application/Commands/PlaceOrder/PlaceOrderCommandValidator.cs
namespace MyApp.Application.Commands.PlaceOrder
{
    /// <summary>
    /// Application-level validation using FluentValidation.
    /// Validates input format and authorization.
    /// </summary>
    public sealed class PlaceOrderCommandValidator : AbstractValidator<PlaceOrderCommand>
    {
        private readonly ICustomerRepository _customerRepository;
        
        public PlaceOrderCommandValidator(ICustomerRepository customerRepository)
        {
            _customerRepository = customerRepository;
            
            // ✅ Input validation - format, required fields
            RuleFor(x => x.CustomerId)
                .NotEmpty()
                .WithMessage("CustomerId is required");
            
            RuleFor(x => x.Items)
                .NotEmpty()
                .WithMessage("At least one item is required");
            
            RuleFor(x => x.Items)
                .Must(items => items.All(i => i.Quantity > 0))
                .WithMessage("All quantities must be positive");
            
            // ✅ External constraint - does customer exist?
            RuleFor(x => x.CustomerId)
                .MustAsync(async (customerId, cancellation) =>
                {
                    var customer = await _customerRepository.GetByIdAsync(customerId);
                    return customer.IsSome;
                })
                .WithMessage("Customer not found");
        }
    }
}

// Domain/Entities/Order.cs
namespace MyApp.Domain.Entities
{
    public sealed class Order
    {
        private readonly List<OrderItem> _items = new();
        
        public OrderId Id { get; }
        public CustomerId CustomerId { get; }
        public IReadOnlyList<OrderItem> Items => _items.AsReadOnly();
        public Money Total { get; private set; }
        
        private Order(OrderId id, CustomerId customerId)
        {
            Id = id;
            CustomerId = customerId;
            Total = Money.Zero;
        }
        
        public static Order Create(CustomerId customerId)
        {
            return new Order(OrderId.New(), customerId);
        }
        
        public Result<Unit, string> AddItem(Product product, int quantity)
        {
            // ✅ Domain validation - business invariants
            if (quantity <= 0)
                return Result<Unit, string>.Failure(
                    "Quantity must be positive");
            
            // ✅ Business rule - total item limit
            if (_items.Sum(i => i.Quantity) + quantity > 100)
                return Result<Unit, string>.Failure(
                    "Cannot order more than 100 items total");
            
            // ✅ Business rule - stock availability
            if (!product.HasSufficientStock(quantity))
                return Result<Unit, string>.Failure(
                    $"Insufficient stock for product {product.Name}");
            
            var item = new OrderItem(product.Id, quantity, product.Price);
            _items.Add(item);
            RecalculateTotal();
            
            return Result<Unit, string>.Success(Unit.Value);
        }
        
        public Result<Unit, string> Submit()
        {
            // ✅ Business invariant
            if (_items.Count == 0)
                return Result<Unit, string>.Failure(
                    "Cannot submit empty order");
            
            // ✅ Business rule
            if (Total.Amount <= 0)
                return Result<Unit, string>.Failure(
                    "Order total must be positive");
            
            Status = OrderStatus.Submitted;
            return Result<Unit, string>.Success(Unit.Value);
        }
        
        public Result<Unit, string> Cancel(string reason)
        {
            // ✅ Business rule - state transition validation
            if (Status == OrderStatus.Shipped)
                return Result<Unit, string>.Failure(
                    "Cannot cancel shipped order");
            
            if (Status == OrderStatus.Cancelled)
                return Result<Unit, string>.Failure(
                    "Order already cancelled");
            
            if (string.IsNullOrWhiteSpace(reason))
                return Result<Unit, string>.Failure(
                    "Cancellation reason required");
            
            Status = OrderStatus.Cancelled;
            CancellationReason = reason;
            
            return Result<Unit, string>.Success(Unit.Value);
        }
    }
}

// Application/Decorators/ValidationCommandHandlerDecorator.cs
namespace MyApp.Application.Decorators
{
    /// <summary>
    /// Decorator applies application validation before handler execution.
    /// </summary>
    public sealed class ValidationCommandHandlerDecorator<TCommand, TSuccess, TError>
        : ICommandHandler<TCommand, TSuccess, TError>
    {
        private readonly ICommandHandler<TCommand, TSuccess, TError> _inner;
        private readonly IValidator<TCommand> _validator;
        
        public async Task<Result<TSuccess, TError>> HandleAsync(
            TCommand command,
            CancellationToken cancellationToken = default)
        {
            // ✅ Application validation runs first
            var validationResult = await _validator.ValidateAsync(
                command,
                cancellationToken);
            
            if (!validationResult.IsValid)
            {
                var errors = string.Join(", ", 
                    validationResult.Errors.Select(e => e.ErrorMessage));
                
                throw new ValidationException(errors);
            }
            
            // ✅ Then domain validation in handler
            return await _inner.HandleAsync(command, cancellationToken);
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
        
        // ✅ Controller just translates HTTP to command
        [HttpPost]
        public async Task<IActionResult> PlaceOrder(
            [FromBody] PlaceOrderRequest request)
        {
            var command = new PlaceOrderCommand(
                new CustomerId(request.CustomerId),
                request.Items.Select(i => new OrderItemCommand(
                    new ProductId(i.ProductId),
                    i.Quantity)).ToList());
            
            // Validation decorator runs application validation
            // Handler runs domain validation
            var result = await _placeOrderHandler.HandleAsync(command);
            
            return result.Match(
                onSuccess: orderId => CreatedAtAction(
                    nameof(GetOrder),
                    new { id = orderId.Value },
                    new { OrderId = orderId.Value }),
                onFailure: error => BadRequest(new { Error = error.ToString() }));
        }
        
        [HttpGet("{id}")]
        public IActionResult GetOrder(Guid id) => Ok();
    }
}
```

## Validation Comparison

| Aspect | Application Validation | Domain Validation |
|--------|------------------------|-------------------|
| **When** | Before domain logic | During domain logic |
| **What** | Input format, authorization | Business invariants |
| **Where** | Validators, decorators | Domain entities/services |
| **Errors** | Exceptions (400 Bad Request) | Result types |
| **Examples** | Required fields, email format | Stock availability, status transitions |
| **Technology** | FluentValidation, Data Annotations | Pure C# logic |

## Benefits

1. **Clear Separation**: Each layer validates what it owns
2. **Fast Failure**: Invalid input rejected before expensive domain logic
3. **Reusability**: Domain validation works regardless of input source (HTTP, CLI, messages)
4. **Type Safety**: Domain returns Result types, not exceptions

## Common Mistakes

❌ **Business rules in application validators**
```csharp
// ❌ Wrong
RuleFor(x => x.Items)
    .Must(items => items.Sum(i => i.Quantity) <= 100)
    .WithMessage("Cannot order more than 100 items");
```

✅ **Business rules in domain**
```csharp
// ✅ Correct
public Result<Unit, string> AddItem(Product product, int quantity)
{
    if (_items.Sum(i => i.Quantity) + quantity > 100)
        return Result.Failure("Cannot order more than 100 items total");
    // ...
}
```

## See Also

- [Honest Functions](./honest-functions.md) — Result types for domain errors
- [Smart Constructors](./smart-constructors.md) — parse, don't validate
- [Domain Invariants](./domain-invariants.md) — enforcing business rules
- [Cross-Cutting Concerns](./cross-cutting-concerns-decorators.md) — validation decorator
