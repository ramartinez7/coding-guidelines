# Code Organization Patterns

> Organize code by feature, maintain clean folder structures, and group related functionality for maintainability.

## Problem

Generic folder structures like "Controllers," "Services," "Models" hide what the system does and make features hard to locate and change.

## Example

### ❌ Before (Technical Grouping)

```
/Controllers
  ├── UserController.cs
  ├── OrderController.cs
  ├── ProductController.cs
/Services
  ├── UserService.cs
  ├── OrderService.cs
  ├── ProductService.cs
/Models
  ├── User.cs
  ├── Order.cs
  ├── Product.cs
/Repositories
  ├── UserRepository.cs
  ├── OrderRepository.cs
  ├── ProductRepository.cs
```

### ✅ After (Feature Grouping)

```
/Users
  ├── UserController.cs
  ├── UserService.cs
  ├── User.cs
  ├── UserRepository.cs
  ├── Commands
  │   ├── CreateUser.cs
  │   ├── UpdateUser.cs
  ├── Queries
  │   ├── GetUserById.cs
  │   ├── SearchUsers.cs
/Orders
  ├── OrderController.cs
  ├── OrderService.cs
  ├── Order.cs
  ├── OrderRepository.cs
  ├── Commands
  │   ├── PlaceOrder.cs
  │   ├── CancelOrder.cs
  ├── Queries
  │   ├── GetOrderById.cs
  │   ├── GetCustomerOrders.cs
/Products
  ├── ProductController.cs
  ├── ProductService.cs
  ├── Product.cs
  ├── ProductRepository.cs
```

## Best Practices

### 1. Organize by Feature, Not Layer

```csharp
// ❌ Organized by technical concern
/Services
  UserService.cs
  OrderService.cs
/Repositories
  UserRepository.cs
  OrderRepository.cs

// ✅ Organized by business capability
/UserManagement
  /Core
    User.cs
    UserService.cs
  /Data
    UserRepository.cs
  /Api
    UserController.cs
/OrderProcessing
  /Core
    Order.cs
    OrderService.cs
  /Data
    OrderRepository.cs
  /Api
    OrderController.cs
```

### 2. Use Vertical Slice Architecture

```csharp
// ✅ Each feature is self-contained
/Features
  /PlaceOrder
    PlaceOrderCommand.cs
    PlaceOrderHandler.cs
    PlaceOrderValidator.cs
    PlaceOrderController.cs
  /CancelOrder
    CancelOrderCommand.cs
    CancelOrderHandler.cs
    CancelOrderValidator.cs
    CancelOrderController.cs
  /GetOrderHistory
    GetOrderHistoryQuery.cs
    GetOrderHistoryHandler.cs
    GetOrderHistoryController.cs
```

### 3. Group Related Types Together

```csharp
// ❌ Scattered related types
/Models/OrderStatus.cs
/Enums/PaymentMethod.cs
/ValueObjects/Money.cs
/Entities/OrderItem.cs

// ✅ Keep related types together
/Orders
  Order.cs
  OrderStatus.cs
  OrderItem.cs
  PaymentMethod.cs
  Money.cs  // Or in /SharedKernel if used elsewhere
```

### 4. Use Namespace-Based Organization

```csharp
// ✅ Namespaces reflect feature boundaries
namespace ECommerce.Orders.Domain
{
    public class Order { }
    public record OrderId(Guid Value);
}

namespace ECommerce.Orders.Application
{
    public class PlaceOrderHandler { }
}

namespace ECommerce.Orders.Infrastructure
{
    public class OrderRepository { }
}
```

### 5. Separate Shared Code

```csharp
// ✅ Clearly identify shared vs feature-specific
/SharedKernel
  /ValueObjects
    Money.cs
    EmailAddress.cs
  /Results
    Result.cs
    Error.cs
/Orders
  OrderId.cs  // Order-specific
  Order.cs
/Customers
  CustomerId.cs  // Customer-specific
  Customer.cs
```

### 6. Use Partial Classes for Organization

```csharp
// ✅ Split large classes logically
// Order.cs (core entity)
public partial class Order
{
    public OrderId Id { get; }
    public CustomerId CustomerId { get; }
    public OrderStatus Status { get; private set; }
}

// Order.Validation.cs
public partial class Order
{
    private void ValidateOrderItems()
    {
        // Validation logic
    }
}

// Order.StateMachine.cs
public partial class Order
{
    public Result<Unit, OrderError> Ship()
    {
        // State transition logic
    }
}
```

### 7. Keep Related Tests Close

```csharp
// ✅ Tests mirror production structure
/src
  /Orders
    Order.cs
    PlaceOrderHandler.cs
/tests
  /Orders
    OrderTests.cs
    PlaceOrderHandlerTests.cs
```

### 8. Use Internal by Default

```csharp
// ✅ Only expose what's necessary
namespace ECommerce.Orders
{
    // Public API
    public class PlaceOrderHandler { }

    // Internal implementation
    internal class OrderValidator { }
    internal class OrderEmailNotifier { }
}
```

### 9. Avoid Deep Nesting

```csharp
// ❌ Too deep
/Features/Orders/Commands/Place/Handlers/PlaceOrderCommandHandler.cs

// ✅ Shallow, clear structure
/Orders/Commands/PlaceOrder/Handler.cs
// or
/Orders/PlaceOrderHandler.cs
```

### 10. Use Folder Structure for Documentation

```csharp
// ✅ Structure explains architecture
/Core          // Domain and application logic
  /Domain
  /Application
/Infrastructure  // External concerns
  /Persistence
  /Email
  /ExternalApis
/Api          // Entry points
  /Controllers
  /Middleware
```

### 11. Separate Read and Write Models (CQRS)

```csharp
// ✅ Clear separation
/Orders
  /Commands    // Writes
    PlaceOrder.cs
    CancelOrder.cs
  /Queries     // Reads
    GetOrder.cs
    SearchOrders.cs
  /Domain      // Write model
    Order.cs
  /ReadModels  // Read model
    OrderSummary.cs
    OrderHistory.cs
```

### 12. Use Analyzers to Enforce Rules

```csharp
// ✅ Use .editorconfig and analyzers
// Enforce namespace matches folder structure
dotnet_diagnostic.IDE0130.severity = error

// Prevent circular dependencies
// Use tools like NDepend or ArchUnitNET
[Test]
public void Domain_Should_Not_Reference_Infrastructure()
{
    var architecture = new ArchLoader()
        .LoadAssemblies(typeof(Order).Assembly);

    var domainLayer = architecture
        .Types()
        .That()
        .ResideInNamespace("Domain");

    var infrastructureLayer = architecture
        .Types()
        .That()
        .ResideInNamespace("Infrastructure");

    Assert.That(
        domainLayer.Should().NotDependOnAny(infrastructureLayer));
}
```

## Symptoms

- Difficulty finding related code
- Large files with unrelated functionality
- Circular dependencies
- Changes affecting many unrelated areas
- Unclear system boundaries

## Benefits

- **Easier navigation** with feature-based organization
- **Better encapsulation** with internal types
- **Clearer boundaries** between features
- **Faster development** by grouping related code
- **Easier testing** with colocated tests

## See Also

- [Screaming Architecture](./screaming-architecture.md) — Structure reveals intent
- [Bounded Contexts](./bounded-contexts.md) — Strategic domain boundaries
- [Clean Architecture](./clean-architecture.md) — Layer organization
- [Use Cases](./use-cases.md) — Feature-centric design
