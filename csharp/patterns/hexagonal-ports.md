# Hexagonal Ports (Input/Output Boundaries)

> Define clear boundaries between the application core and external concerns through ports (interfaces)—primary ports for use cases driven by external actors, secondary ports for infrastructure services needed by the core.

## Problem

Without clear boundaries, the application becomes tightly coupled to external systems, making it hard to test and impossible to swap implementations.

## Core Concept

**Ports** are interfaces that define boundaries:
- **Primary (Driving) Ports**: Input boundaries—what the application can do (use cases, commands, queries)
- **Secondary (Driven) Ports**: Output boundaries—what the application needs (repositories, external services)

```
┌──────────────────────────────────────┐
│      Primary Adapters                │
│   (HTTP, CLI, gRPC, Messages)        │
└──────────┬───────────────────────────┘
           │ drives
           ▼
    ┌────────────────┐
    │ Primary Ports  │ (Use Cases, Command Handlers)
    │                │
    │  Application   │
    │     Core       │
    │                │
    │ Secondary Ports│ (IRepository, IEmailService)
    └────────┬───────┘
             │ driven by
             ▼
┌────────────────────────────────────┐
│     Secondary Adapters             │
│ (Database, Email, External APIs)   │
└────────────────────────────────────┘
```

## Example

### Primary Ports (Input - What Application Does)

```csharp
// Application/Commands/ICommandHandler.cs (Primary Port)
namespace MyApp.Application.Commands
{
    /// <summary>
    /// Primary port - defines what commands the application accepts.
    /// Driven by primary adapters (controllers, CLI, messages).
    /// </summary>
    public interface ICommandHandler<in TCommand, TSuccess, TError>
    {
        Task<Result<TSuccess, TError>> HandleAsync(
            TCommand command,
            CancellationToken cancellationToken = default);
    }
}

// Application/Queries/IQueryHandler.cs (Primary Port)
namespace MyApp.Application.Queries
{
    /// <summary>
    /// Primary port - defines what queries the application supports.
    /// </summary>
    public interface IQueryHandler<in TQuery, TResult>
    {
        Task<TResult> HandleAsync(
            TQuery query,
            CancellationToken cancellationToken = default);
    }
}

// Application/Commands/PlaceOrder/PlaceOrderCommand.cs (Primary Port Contract)
public sealed record PlaceOrderCommand(
    CustomerId CustomerId,
    List<OrderItemCommand> Items);

// Application/Commands/PlaceOrder/PlaceOrderCommandHandler.cs (Primary Port Implementation)
public sealed class PlaceOrderCommandHandler 
    : ICommandHandler<PlaceOrderCommand, OrderId, PlaceOrderError>
{
    public async Task<Result<OrderId, PlaceOrderError>> HandleAsync(
        PlaceOrderCommand command,
        CancellationToken cancellationToken = default)
    {
        // Implementation
    }
}
```

### Secondary Ports (Output - What Application Needs)

```csharp
// Domain/Ports/IOrderRepository.cs (Secondary Port)
namespace MyApp.Domain.Ports
{
    /// <summary>
    /// Secondary port - defines what persistence the domain needs.
    /// Will be implemented by secondary adapter.
    /// </summary>
    public interface IOrderRepository
    {
        Task<Option<Order>> GetByIdAsync(OrderId id);
        Task SaveAsync(Order order);
        Task DeleteAsync(Order order);
    }
}

// Application/Ports/IEmailService.cs (Secondary Port)
namespace MyApp.Application.Ports
{
    /// <summary>
    /// Secondary port - defines what email capabilities application needs.
    /// </summary>
    public interface IEmailService
    {
        Task SendOrderConfirmationAsync(OrderId orderId, Email recipientEmail);
        Task SendShippingNotificationAsync(OrderId orderId, Email recipientEmail);
    }
}

// Application/Ports/IPaymentGateway.cs (Secondary Port)
namespace MyApp.Application.Ports
{
    /// <summary>
    /// Secondary port - defines payment processing needs.
    /// </summary>
    public interface IPaymentGateway
    {
        Task<Result<PaymentId, PaymentError>> ProcessPaymentAsync(
            Money amount,
            PaymentMethod paymentMethod);
        
        Task<Result<Unit, RefundError>> RefundPaymentAsync(
            PaymentId paymentId,
            Money amount);
    }
}
```

## Port Ownership

| Port Type | Defined By | Implemented By | Direction |
|-----------|-----------|----------------|-----------|
| **Primary** | Application Core | External Actors | Inbound (Driving) |
| **Secondary** | Application Core | Infrastructure | Outbound (Driven) |

## Benefits

1. **Testability**: Mock secondary ports, invoke primary ports directly
2. **Flexibility**: Swap adapters without changing core
3. **Clear Contracts**: Explicit boundaries between layers
4. **Independence**: Core doesn't know about external technologies

## Port Guidelines

**Primary Ports:**
- Define application capabilities (use cases)
- Accept commands/queries as input
- Return results (not exceptions for business failures)
- Example: `ICommandHandler<TCommand>`, `IQueryHandler<TQuery>`

**Secondary Ports:**
- Define infrastructure needs
- Minimal, focused interfaces
- Technology-agnostic
- Example: `IEmailService`, `IOrderRepository`, `IPaymentGateway`

## See Also

- [Port-Adapter Separation](./port-adapter-separation.md) — hexagonal architecture
- [Primary vs Secondary Adapters](./primary-secondary-adapters.md) — adapter types
- [Clean Architecture](./clean-architecture.md) — layered dependencies
