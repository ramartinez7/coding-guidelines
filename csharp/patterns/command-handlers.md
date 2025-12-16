# Command Handlers (Write-Side Orchestration)

> Encapsulate write operations in dedicated command handlers that validate input, execute business logic, and coordinate domain operations—separating concerns and enabling decorator-based cross-cutting concerns.

## Problem

When write operations are implemented directly in controllers or scattered across service classes, business logic becomes hard to test, cross-cutting concerns (logging, validation, transactions) get duplicated, and adding new features requires modifying existing code.

## Core Concept

**Command Handlers** implement the Command pattern:
- **Command** = immutable data representing intent to change state
- **Handler** = executes the command, validates, and coordinates domain logic
- One handler per command (Single Responsibility)
- Returns `Result<TSuccess, TError>` (no exceptions for business failures)

## Example

### ❌ Before (Logic in Controller)

```csharp
[ApiController]
[Route("api/orders")]
public class OrdersController : ControllerBase
{
    private readonly IOrderRepository _orderRepository;
    private readonly ILogger _logger;
    
    [HttpPost("{orderId}/cancel")]
    public async Task<IActionResult> CancelOrder(Guid orderId, CancelOrderRequest request)
    {
        _logger.LogInformation("Cancelling order {OrderId}", orderId);
        
        // ❌ Validation in controller
        if (string.IsNullOrEmpty(request.Reason))
            return BadRequest("Reason is required");
        
        // ❌ Business logic in controller
        var order = await _orderRepository.GetByIdAsync(new OrderId(orderId));
        if (order == null)
            return NotFound();
        
        order.Cancel(request.Reason);
        
        await _orderRepository.SaveAsync(order);
        
        _logger.LogInformation("Order {OrderId} cancelled", orderId);
        
        return Ok();
    }
}
```

### ✅ After (Command Handler)

```csharp
// Application/Commands/CancelOrder/CancelOrderCommand.cs
namespace MyApp.Application.Commands.CancelOrder
{
    /// <summary>
    /// Immutable command representing intent to cancel an order.
    /// </summary>
    public sealed record CancelOrderCommand(
        OrderId OrderId,
        string Reason,
        UserId CancelledBy);
}

// Application/Commands/CancelOrder/CancelOrderCommandHandler.cs
namespace MyApp.Application.Commands.CancelOrder
{
    /// <summary>
    /// Handler for CancelOrderCommand.
    /// Contains all logic for cancelling orders.
    /// </summary>
    public sealed class CancelOrderCommandHandler 
        : ICommandHandler<CancelOrderCommand, Unit, CancelOrderError>
    {
        private readonly IOrderRepository _orderRepository;
        private readonly IUnitOfWork _unitOfWork;
        private readonly IDomainEventDispatcher _eventDispatcher;
        
        public CancelOrderCommandHandler(
            IOrderRepository orderRepository,
            IUnitOfWork unitOfWork,
            IDomainEventDispatcher eventDispatcher)
        {
            _orderRepository = orderRepository;
            _unitOfWork = unitOfWork;
            _eventDispatcher = eventDispatcher;
        }
        
        public async Task<Result<Unit, CancelOrderError>> HandleAsync(
            CancelOrderCommand command,
            CancellationToken cancellationToken = default)
        {
            // ✅ Fetch aggregate
            var orderResult = await _orderRepository.GetByIdAsync(command.OrderId);
            
            var order = await orderResult.Match(
                onSome: o => Task.FromResult<Order?>(o),
                onNone: () => Task.FromResult<Order?>(null));
            
            if (order == null)
                return Result<Unit, CancelOrderError>.Failure(
                    new CancelOrderError.OrderNotFound(command.OrderId));
            
            // ✅ Execute domain logic
            var cancelResult = order.Cancel(command.Reason, command.CancelledBy);
            if (!cancelResult.IsSuccess)
                return Result<Unit, CancelOrderError>.Failure(
                    new CancelOrderError.CannotCancel(cancelResult.Error));
            
            // ✅ Persist
            await _orderRepository.SaveAsync(order);
            await _unitOfWork.CommitAsync(cancellationToken);
            
            // ✅ Dispatch events
            await _eventDispatcher.DispatchAsync(order.DomainEvents);
            order.ClearDomainEvents();
            
            return Result<Unit, CancelOrderError>.Success(Unit.Value);
        }
    }
    
    /// <summary>
    /// Errors that can occur when cancelling an order.
    /// </summary>
    public abstract record CancelOrderError
    {
        public sealed record OrderNotFound(OrderId OrderId) : CancelOrderError;
        public sealed record CannotCancel(string Reason) : CancelOrderError;
    }
}

// Application/Commands/ICommandHandler.cs
namespace MyApp.Application.Commands
{
    /// <summary>
    /// Base interface for all command handlers.
    /// </summary>
    public interface ICommandHandler<in TCommand, TSuccess, TError>
    {
        Task<Result<TSuccess, TError>> HandleAsync(
            TCommand command,
            CancellationToken cancellationToken = default);
    }
}

// WebApi/Controllers/OrdersController.cs
namespace MyApp.WebApi.Controllers
{
    [ApiController]
    [Route("api/orders")]
    public class OrdersController : ControllerBase
    {
        private readonly CancelOrderCommandHandler _cancelOrderHandler;
        private readonly ICurrentUserService _currentUserService;
        
        public OrdersController(
            CancelOrderCommandHandler cancelOrderHandler,
            ICurrentUserService currentUserService)
        {
            _cancelOrderHandler = cancelOrderHandler;
            _currentUserService = currentUserService;
        }
        
        // ✅ Controller just translates HTTP to command
        [HttpPost("{orderId}/cancel")]
        public async Task<IActionResult> CancelOrder(
            Guid orderId,
            [FromBody] CancelOrderRequest request)
        {
            var command = new CancelOrderCommand(
                new OrderId(orderId),
                request.Reason,
                _currentUserService.GetCurrentUserId());
            
            var result = await _cancelOrderHandler.HandleAsync(command);
            
            return result.Match(
                onSuccess: _ => Ok(new { Message = "Order cancelled successfully" }),
                onFailure: error => error switch
                {
                    CancelOrderError.OrderNotFound => NotFound(),
                    CancelOrderError.CannotCancel e => 
                        BadRequest(new { Error = e.Reason }),
                    _ => StatusCode(500)
                });
        }
    }
    
    public sealed record CancelOrderRequest(string Reason);
}
```

## Decorator Pattern for Cross-Cutting Concerns

```csharp
// Application/Commands/Decorators/LoggingCommandHandlerDecorator.cs
namespace MyApp.Application.Commands.Decorators
{
    /// <summary>
    /// Decorator that adds logging to any command handler.
    /// </summary>
    public sealed class LoggingCommandHandlerDecorator<TCommand, TSuccess, TError>
        : ICommandHandler<TCommand, TSuccess, TError>
    {
        private readonly ICommandHandler<TCommand, TSuccess, TError> _inner;
        private readonly ILogger<TCommand> _logger;
        
        public LoggingCommandHandlerDecorator(
            ICommandHandler<TCommand, TSuccess, TError> inner,
            ILogger<TCommand> logger)
        {
            _inner = inner;
            _logger = logger;
        }
        
        public async Task<Result<TSuccess, TError>> HandleAsync(
            TCommand command,
            CancellationToken cancellationToken = default)
        {
            _logger.LogInformation(
                "Executing command {CommandType}: {Command}",
                typeof(TCommand).Name,
                command);
            
            var stopwatch = Stopwatch.StartNew();
            
            try
            {
                var result = await _inner.HandleAsync(command, cancellationToken);
                
                stopwatch.Stop();
                
                _logger.LogInformation(
                    "Command {CommandType} completed in {ElapsedMs}ms with result: {Result}",
                    typeof(TCommand).Name,
                    stopwatch.ElapsedMilliseconds,
                    result.IsSuccess ? "Success" : "Failure");
                
                return result;
            }
            catch (Exception ex)
            {
                _logger.LogError(ex,
                    "Command {CommandType} failed with exception",
                    typeof(TCommand).Name);
                throw;
            }
        }
    }
}

// Application/Commands/Decorators/ValidationCommandHandlerDecorator.cs
namespace MyApp.Application.Commands.Decorators
{
    /// <summary>
    /// Decorator that adds validation to any command handler.
    /// </summary>
    public sealed class ValidationCommandHandlerDecorator<TCommand, TSuccess, TError>
        : ICommandHandler<TCommand, TSuccess, TError>
    {
        private readonly ICommandHandler<TCommand, TSuccess, TError> _inner;
        private readonly IValidator<TCommand> _validator;
        
        public ValidationCommandHandlerDecorator(
            ICommandHandler<TCommand, TSuccess, TError> inner,
            IValidator<TCommand> validator)
        {
            _inner = inner;
            _validator = validator;
        }
        
        public async Task<Result<TSuccess, TError>> HandleAsync(
            TCommand command,
            CancellationToken cancellationToken = default)
        {
            var validationResult = await _validator.ValidateAsync(
                command, 
                cancellationToken);
            
            if (!validationResult.IsValid)
            {
                // Convert validation errors to command error
                // (requires TError to have a ValidationFailed case)
                throw new ValidationException(validationResult.Errors);
            }
            
            return await _inner.HandleAsync(command, cancellationToken);
        }
    }
}

// DI Registration with Decorators
services.AddScoped<CancelOrderCommandHandler>();
services.Decorate<ICommandHandler<CancelOrderCommand, Unit, CancelOrderError>,
    ValidationCommandHandlerDecorator<CancelOrderCommand, Unit, CancelOrderError>>();
services.Decorate<ICommandHandler<CancelOrderCommand, Unit, CancelOrderError>,
    LoggingCommandHandlerDecorator<CancelOrderCommand, Unit, CancelOrderError>>();
```

## Benefits

1. **Single Responsibility**: Each handler does one thing
2. **Testability**: Handlers easy to test in isolation
3. **Cross-Cutting Concerns**: Decorators add logging, validation, transactions
4. **Type Safety**: Commands and errors strongly typed
5. **Extensibility**: Add new commands without modifying existing code

## See Also

- [Use Cases](./use-cases.md) — application services
- [CQRS](./read-write-model-separation.md) — separating reads and writes
- [Domain Events](./domain-event-handlers.md) — handling side effects
- [Result Monad](./result-monad.md) — error handling
