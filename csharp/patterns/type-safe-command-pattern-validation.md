# Type-Safe Command Pattern with Validation

> Commands executed without validation or with runtime type checking—use typed commands with compile-time validation to ensure command integrity.

## Problem

Command pattern implementations using generic command interfaces or base classes don't enforce validation rules at compile time. Commands can be executed in invalid states, with missing required data, or without proper authorization checks.

## Example

### ❌ Before

```csharp
public interface ICommand
{
    Task ExecuteAsync();
}

public class CreateOrderCommand : ICommand
{
    public int? UserId { get; set; }  // Nullable—might be missing!
    public List<int>? ProductIds { get; set; }  // Might be null or empty!
    
    public async Task ExecuteAsync()
    {
        // Runtime validation required
        if (UserId == null)
            throw new InvalidOperationException("UserId required");
        
        if (ProductIds == null || ProductIds.Count == 0)
            throw new InvalidOperationException("ProductIds required");
        
        // Execute command
    }
}
```

### ✅ After

```csharp
public interface ICommand<TResult, TError>
{
    Task<Result<TResult, TError>> ExecuteAsync();
}

public interface IValidatedCommand<TResult, TError> : ICommand<TResult, TError>
{
    Result<Unit, TError> Validate();
}

public sealed record CreateOrderCommandData
{
    public required UserId UserId { get; init; }
    public required NonEmptyList<ProductId> ProductIds { get; init; }
    public required ShippingAddress ShippingAddress { get; init; }
}

public sealed class CreateOrderCommand 
    : IValidatedCommand<OrderId, CreateOrderError>
{
    private readonly CreateOrderCommandData _data;
    private readonly IOrderRepository _orders;
    
    private CreateOrderCommand(
        CreateOrderCommandData data,
        IOrderRepository orders)
    {
        _data = data;
        _orders = orders;
    }
    
    public static Result<CreateOrderCommand, CreateOrderError> Create(
        CreateOrderCommandData data,
        IOrderRepository orders)
    {
        // Validation at construction
        var command = new CreateOrderCommand(data, orders);
        var validationResult = command.Validate();
        
        return validationResult.Match(
            _ => Result<CreateOrderCommand, CreateOrderError>.Success(command),
            error => Result<CreateOrderCommand, CreateOrderError>.Failure(error));
    }
    
    public Result<Unit, CreateOrderError> Validate()
    {
        // All data guaranteed present by required properties
        // Additional business rule validation here
        return Result<Unit, CreateOrderError>.Success(Unit.Value);
    }
    
    public async Task<Result<OrderId, CreateOrderError>> ExecuteAsync()
    {
        // Guaranteed valid because of construction
        var order = Order.Create(
            _data.UserId,
            _data.ProductIds,
            _data.ShippingAddress);
        
        await _orders.SaveAsync(order.Value);
        
        return Result<OrderId, CreateOrderError>.Success(order.Value.Id);
    }
}

public abstract record CreateOrderError
{
    public sealed record InvalidUser : CreateOrderError;
    public sealed record EmptyProductList : CreateOrderError;
    public sealed record InvalidAddress : CreateOrderError;
}

// Non-empty list enforced at type level
public sealed class NonEmptyList<T>
{
    private readonly List<T> _items;
    
    private NonEmptyList(T first, IEnumerable<T> rest)
    {
        _items = new List<T> { first };
        _items.AddRange(rest);
    }
    
    public static NonEmptyList<T> Create(T first, params T[] rest)
        => new(first, rest);
    
    public static Result<NonEmptyList<T>, string> FromList(List<T> items)
    {
        if (items.Count == 0)
            return Result<NonEmptyList<T>, string>.Failure("List cannot be empty");
        
        return Result<NonEmptyList<T>, string>.Success(
            new NonEmptyList<T>(items[0], items.Skip(1)));
    }
    
    public IReadOnlyList<T> Items => _items.AsReadOnly();
    public T Head => _items[0];
    public IEnumerable<T> Tail => _items.Skip(1);
}

// Usage: Type-safe command execution
public class OrderController
{
    public async Task<IActionResult> CreateOrder(CreateOrderRequest request)
    {
        var productIds = NonEmptyList<ProductId>
            .FromList(request.ProductIds.Select(id => new ProductId(id)).ToList());
        
        if (productIds.IsFailure)
            return BadRequest(productIds.Error);
        
        var commandData = new CreateOrderCommandData
        {
            UserId = new UserId(request.UserId),
            ProductIds = productIds.Value,
            ShippingAddress = ShippingAddress.Create(request.Address).Value
        };
        
        var command = CreateOrderCommand.Create(commandData, _orderRepository);
        
        if (command.IsFailure)
            return BadRequest(command.Error);
        
        var result = await command.Value.ExecuteAsync();
        
        return result.Match(
            orderId => Ok(new { OrderId = orderId.Value }),
            error => BadRequest(error));
    }
}

// Won't compile:
// var data = new CreateOrderCommandData { };  // Missing required properties
```

## See Also

- [Smart Constructors](./smart-constructors.md)
- [Result Monad](./result-monad.md)
- [Refinement Types](./refinement-types.md)
- [Domain Invariants](./domain-invariants.md)
