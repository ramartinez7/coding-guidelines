# Use Case Transactions and Unit of Work

> Manage database transactions at the application layer using the Unit of Work pattern—ensuring atomic operations and keeping the domain transaction-agnostic.

## Problem

When transaction management is scattered throughout the codebase or mixed into domain logic, it becomes hard to ensure atomicity and the domain becomes coupled to infrastructure.

## Core Concept

**Unit of Work** pattern:
- Application layer controls transaction boundaries
- Domain layer never knows about transactions
- Single commit point for all changes in a use case
- Rollback on failure

## Example

### ❌ Before (Scattered Transactions)

```csharp
// ❌ Transaction management in domain
public class Order
{
    public async Task Submit(IDbTransaction transaction)  // ❌ Domain knows about transactions
    {
        Status = OrderStatus.Submitted;
        await transaction.CommitAsync();  // ❌ Domain controls transactions
    }
}

// ❌ Manual transaction management in every use case
public class PlaceOrderUseCase
{
    public async Task ExecuteAsync(PlaceOrderCommand command)
    {
        using var transaction = await _dbContext.Database.BeginTransactionAsync();
        
        try
        {
            var order = Order.Create(command.CustomerId);
            await _orderRepository.SaveAsync(order);
            
            await _inventoryService.ReserveStockAsync(order.Items);
            
            await transaction.CommitAsync();
        }
        catch
        {
            await transaction.RollbackAsync();
            throw;
        }
    }
}
```

### ✅ After (Unit of Work Pattern)

```csharp
// Application/Common/IUnitOfWork.cs
namespace MyApp.Application.Common
{
    /// <summary>
    /// Unit of Work controls transaction boundaries.
    /// </summary>
    public interface IUnitOfWork
    {
        Task CommitAsync(CancellationToken cancellationToken = default);
        Task RollbackAsync(CancellationToken cancellationToken = default);
    }
}

// Infrastructure/Persistence/UnitOfWork.cs
namespace MyApp.Infrastructure.Persistence
{
    /// <summary>
    /// EF Core implementation of Unit of Work.
    /// </summary>
    public sealed class UnitOfWork : IUnitOfWork
    {
        private readonly MyDbContext _dbContext;
        
        public UnitOfWork(MyDbContext dbContext)
        {
            _dbContext = dbContext;
        }
        
        public async Task CommitAsync(CancellationToken cancellationToken = default)
        {
            await _dbContext.SaveChangesAsync(cancellationToken);
        }
        
        public async Task RollbackAsync(CancellationToken cancellationToken = default)
        {
            // EF Core: just don't call SaveChanges
            // Changes are automatically rolled back when context is disposed
            await Task.CompletedTask;
        }
    }
}

// Application/Commands/PlaceOrder/PlaceOrderCommandHandler.cs
namespace MyApp.Application.Commands.PlaceOrder
{
    public sealed class PlaceOrderCommandHandler
        : ICommandHandler<PlaceOrderCommand, OrderId, PlaceOrderError>
    {
        private readonly IOrderRepository _orderRepository;
        private readonly IProductRepository _productRepository;
        private readonly IInventoryService _inventoryService;
        private readonly IUnitOfWork _unitOfWork;
        
        public async Task<Result<OrderId, PlaceOrderError>> HandleAsync(
            PlaceOrderCommand command,
            CancellationToken cancellationToken = default)
        {
            // ✅ Domain logic - no transaction knowledge
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
            
            // ✅ Save to repository (tracked by UoW)
            await _orderRepository.SaveAsync(order);
            
            // ✅ External service call
            var reserveResult = await _inventoryService.ReserveStockAsync(order.Items);
            if (!reserveResult.IsSuccess)
                return Result<OrderId, PlaceOrderError>.Failure(
                    new PlaceOrderError.InsufficientStock());
            
            // ✅ Single commit point
            await _unitOfWork.CommitAsync(cancellationToken);
            
            return Result<OrderId, PlaceOrderError>.Success(order.Id);
        }
    }
}

// Application/Decorators/TransactionCommandHandlerDecorator.cs
namespace MyApp.Application.Decorators
{
    /// <summary>
    /// Decorator that adds automatic transaction management.
    /// Commits on success, rolls back on failure.
    /// </summary>
    public sealed class TransactionCommandHandlerDecorator<TCommand, TSuccess, TError>
        : ICommandHandler<TCommand, TSuccess, TError>
    {
        private readonly ICommandHandler<TCommand, TSuccess, TError> _inner;
        private readonly IUnitOfWork _unitOfWork;
        
        public async Task<Result<TSuccess, TError>> HandleAsync(
            TCommand command,
            CancellationToken cancellationToken = default)
        {
            // ✅ Execute handler
            var result = await _inner.HandleAsync(command, cancellationToken);
            
            // ✅ Automatic transaction management based on result
            if (result.IsSuccess)
                await _unitOfWork.CommitAsync(cancellationToken);
            else
                await _unitOfWork.RollbackAsync(cancellationToken);
            
            return result;
        }
    }
}

// DI Registration - Apply transaction decorator to all commands
services.AddScoped<IUnitOfWork, UnitOfWork>();

services.AddScoped<PlaceOrderCommandHandler>();
services.Decorate<ICommandHandler<PlaceOrderCommand, OrderId, PlaceOrderError>,
    TransactionCommandHandlerDecorator<PlaceOrderCommand, OrderId, PlaceOrderError>>();
```

## Advanced: Explicit Transaction Scope

```csharp
// Infrastructure/Persistence/ExplicitUnitOfWork.cs
namespace MyApp.Infrastructure.Persistence
{
    /// <summary>
    /// Unit of Work with explicit transaction control.
    /// </summary>
    public sealed class ExplicitUnitOfWork : IUnitOfWork, IDisposable
    {
        private readonly MyDbContext _dbContext;
        private IDbContextTransaction? _transaction;
        
        public async Task BeginTransactionAsync(CancellationToken cancellationToken = default)
        {
            _transaction = await _dbContext.Database.BeginTransactionAsync(cancellationToken);
        }
        
        public async Task CommitAsync(CancellationToken cancellationToken = default)
        {
            await _dbContext.SaveChangesAsync(cancellationToken);
            
            if (_transaction != null)
                await _transaction.CommitAsync(cancellationToken);
        }
        
        public async Task RollbackAsync(CancellationToken cancellationToken = default)
        {
            if (_transaction != null)
                await _transaction.RollbackAsync(cancellationToken);
        }
        
        public void Dispose()
        {
            _transaction?.Dispose();
        }
    }
}

// Usage in command handler
public async Task<Result<OrderId, PlaceOrderError>> HandleAsync(
    PlaceOrderCommand command,
    CancellationToken cancellationToken = default)
{
    await _unitOfWork.BeginTransactionAsync(cancellationToken);
    
    try
    {
        // ... domain logic
        
        await _unitOfWork.CommitAsync(cancellationToken);
        return Result.Success(order.Id);
    }
    catch (Exception)
    {
        await _unitOfWork.RollbackAsync(cancellationToken);
        throw;
    }
}
```

## Benefits

1. **Atomic Operations**: All changes committed together or not at all
2. **Domain Purity**: Domain never knows about transactions
3. **Consistency**: Single commit point per use case
4. **Testability**: Fake UnitOfWork for testing

## Isolation Levels

```csharp
public interface IUnitOfWork
{
    Task BeginTransactionAsync(
        IsolationLevel isolationLevel = IsolationLevel.ReadCommitted,
        CancellationToken cancellationToken = default);
    
    Task CommitAsync(CancellationToken cancellationToken = default);
    Task RollbackAsync(CancellationToken cancellationToken = default);
}
```

## See Also

- [Clean Architecture](./clean-architecture.md) — layer responsibilities
- [Use Cases](./use-cases.md) — application services
- [Repository Pattern](./repository-pattern.md) — data access
- [Cross-Cutting Concerns](./cross-cutting-concerns-decorators.md) — transaction decorator
