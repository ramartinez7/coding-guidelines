# Cross-Cutting Concerns via Decorators

> Apply cross-cutting concerns (logging, validation, authorization, transactions) through the Decorator pattern rather than scattering them throughout code—keeping business logic clean and concerns composable.

## Problem

Cross-cutting concerns like logging, validation, caching, and transaction management end up duplicated across every command handler, use case, or controller. This violates DRY and makes it easy to forget to apply a concern to new code.

## Core Concept

The **Decorator Pattern** wraps handlers to add behavior without modifying the handler itself:

```
Request → Logging → Validation → Authorization → Transaction → Handler
           ↓          ↓             ↓               ↓
         Log      Validate      Check Auth      Begin TX
           
                                                Handler Executes
           
         Log      (none)        (none)          Commit TX
           ↑                                       ↑
Response ←─┴───────────────────────────────────────┘
```

## Example

### ❌ Before (Concerns Scattered)

```csharp
public class PlaceOrderCommandHandler
{
    public async Task<Result<OrderId, PlaceOrderError>> HandleAsync(
        PlaceOrderCommand command)
    {
        // ❌ Logging in every handler
        _logger.LogInformation("Placing order for customer {CustomerId}", 
            command.CustomerId);
        
        // ❌ Validation in every handler
        if (command.Items.Count == 0)
            return Result.Failure(new PlaceOrderError.NoItems());
        
        // ❌ Authorization in every handler
        if (!await _authService.CanPlaceOrderAsync(command.CustomerId))
            return Result.Failure(new PlaceOrderError.Unauthorized());
        
        // ❌ Transaction management in every handler
        using var transaction = await _dbContext.BeginTransactionAsync();
        
        try
        {
            // Actual business logic buried in infrastructure code
            var order = Order.Create(command.CustomerId);
            // ...
            
            await transaction.CommitAsync();
            
            _logger.LogInformation("Order {OrderId} placed successfully", order.Id);
            
            return Result.Success(order.Id);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to place order");
            await transaction.RollbackAsync();
            throw;
        }
    }
}
```

### ✅ After (Decorators)

```csharp
// Core handler - ONLY business logic
public sealed class PlaceOrderCommandHandler 
    : ICommandHandler<PlaceOrderCommand, OrderId, PlaceOrderError>
{
    private readonly IOrderRepository _orderRepository;
    private readonly IProductRepository _productRepository;
    
    public async Task<Result<OrderId, PlaceOrderError>> HandleAsync(
        PlaceOrderCommand command,
        CancellationToken cancellationToken = default)
    {
        // ✅ Pure business logic - no infrastructure concerns
        var order = Order.Create(command.CustomerId);
        
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
        
        await _orderRepository.SaveAsync(order);
        
        return Result<OrderId, PlaceOrderError>.Success(order.Id);
    }
}

// Application/Decorators/LoggingDecorator.cs
public sealed class LoggingCommandHandlerDecorator<TCommand, TSuccess, TError>
    : ICommandHandler<TCommand, TSuccess, TError>
{
    private readonly ICommandHandler<TCommand, TSuccess, TError> _inner;
    private readonly ILogger<TCommand> _logger;
    
    public async Task<Result<TSuccess, TError>> HandleAsync(
        TCommand command,
        CancellationToken cancellationToken = default)
    {
        _logger.LogInformation("Executing {Command}: {@CommandData}", 
            typeof(TCommand).Name, command);
        
        var stopwatch = Stopwatch.StartNew();
        var result = await _inner.HandleAsync(command, cancellationToken);
        stopwatch.Stop();
        
        if (result.IsSuccess)
            _logger.LogInformation("{Command} completed successfully in {Ms}ms",
                typeof(TCommand).Name, stopwatch.ElapsedMilliseconds);
        else
            _logger.LogWarning("{Command} failed: {Error}",
                typeof(TCommand).Name, result.Error);
        
        return result;
    }
}

// Application/Decorators/ValidationDecorator.cs
public sealed class ValidationCommandHandlerDecorator<TCommand, TSuccess, TError>
    : ICommandHandler<TCommand, TSuccess, TError>
{
    private readonly ICommandHandler<TCommand, TSuccess, TError> _inner;
    private readonly IValidator<TCommand> _validator;
    
    public async Task<Result<TSuccess, TError>> HandleAsync(
        TCommand command,
        CancellationToken cancellationToken = default)
    {
        var validationResult = await _validator.ValidateAsync(command, cancellationToken);
        
        if (!validationResult.IsValid)
        {
            var errors = string.Join(", ", validationResult.Errors);
            throw new ValidationException(errors);
        }
        
        return await _inner.HandleAsync(command, cancellationToken);
    }
}

// Application/Decorators/TransactionDecorator.cs
public sealed class TransactionCommandHandlerDecorator<TCommand, TSuccess, TError>
    : ICommandHandler<TCommand, TSuccess, TError>
{
    private readonly ICommandHandler<TCommand, TSuccess, TError> _inner;
    private readonly IUnitOfWork _unitOfWork;
    
    public async Task<Result<TSuccess, TError>> HandleAsync(
        TCommand command,
        CancellationToken cancellationToken = default)
    {
        var result = await _inner.HandleAsync(command, cancellationToken);
        
        if (result.IsSuccess)
            await _unitOfWork.CommitAsync(cancellationToken);
        else
            await _unitOfWork.RollbackAsync(cancellationToken);
        
        return result;
    }
}

// Application/Decorators/AuthorizationDecorator.cs
public sealed class AuthorizationCommandHandlerDecorator<TCommand, TSuccess, TError>
    : ICommandHandler<TCommand, TSuccess, TError>
    where TCommand : IAuthorizedCommand
{
    private readonly ICommandHandler<TCommand, TSuccess, TError> _inner;
    private readonly IAuthorizationService _authService;
    
    public async Task<Result<TSuccess, TError>> HandleAsync(
        TCommand command,
        CancellationToken cancellationToken = default)
    {
        if (!await _authService.AuthorizeAsync(command.UserId, command.RequiredPermission))
            throw new UnauthorizedException($"User lacks permission: {command.RequiredPermission}");
        
        return await _inner.HandleAsync(command, cancellationToken);
    }
}

// DI Registration - Apply decorators in order
public static class DependencyInjection
{
    public static IServiceCollection AddCommandHandlers(this IServiceCollection services)
    {
        // Register core handler
        services.AddScoped<PlaceOrderCommandHandler>();
        
        // Wrap with decorators (innermost to outermost)
        services.Decorate<ICommandHandler<PlaceOrderCommand, OrderId, PlaceOrderError>,
            TransactionCommandHandlerDecorator<PlaceOrderCommand, OrderId, PlaceOrderError>>();
        
        services.Decorate<ICommandHandler<PlaceOrderCommand, OrderId, PlaceOrderError>,
            AuthorizationCommandHandlerDecorator<PlaceOrderCommand, OrderId, PlaceOrderError>>();
        
        services.Decorate<ICommandHandler<PlaceOrderCommand, OrderId, PlaceOrderError>,
            ValidationCommandHandlerDecorator<PlaceOrderCommand, OrderId, PlaceOrderError>>();
        
        services.Decorate<ICommandHandler<PlaceOrderCommand, OrderId, PlaceOrderError>,
            LoggingCommandHandlerDecorator<PlaceOrderCommand, OrderId, PlaceOrderError>>();
        
        return services;
    }
}
```

## Execution Flow

```
1. Request arrives at controller
2. LoggingDecorator logs "Executing PlaceOrderCommand"
3. ValidationDecorator validates command
4. AuthorizationDecorator checks permissions
5. TransactionDecorator begins transaction
6. PlaceOrderCommandHandler executes business logic
7. TransactionDecorator commits transaction
8. LoggingDecorator logs "Completed in 150ms"
9. Response returned to client
```

## Benefits

1. **Separation of Concerns**: Business logic isolated from infrastructure
2. **DRY**: Write logging/validation/transaction code once
3. **Composability**: Mix and match decorators
4. **Testability**: Test business logic without decorators
5. **Consistency**: All handlers get same treatment

## Common Decorators

| Decorator | Responsibility |
|-----------|----------------|
| **Logging** | Log command execution, duration, results |
| **Validation** | Validate command using FluentValidation |
| **Authorization** | Check user permissions |
| **Transaction** | Begin/commit/rollback database transactions |
| **Retry** | Retry failed commands with exponential backoff |
| **Caching** | Cache query results (read-side only) |
| **Performance** | Measure and alert on slow commands |
| **Auditing** | Record who executed what command when |

## See Also

- [Command Handlers](./command-handlers.md) — command pattern
- [Use Cases](./use-cases.md) — application services
- [Clean Architecture](./clean-architecture.md) — layered architecture
