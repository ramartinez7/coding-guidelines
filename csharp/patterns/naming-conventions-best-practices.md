# Naming Conventions Best Practices

> Follow consistent C# naming conventions for clarity, maintain style across codebase, and use meaningful names.

## Problem

Inconsistent naming makes code harder to read and maintain. Poor naming conventions confuse developers about intent.

## Example

### ❌ Before

```csharp
public class order_processor
{
    private string m_strCustomerName;
    private int _orderId;
    public const string MAX_ITEMS = "100";

    public bool ProcessOrd(int id)
    {
        var ord = GetOrder(id);
        if (ord == null) return false;
        // Process
        return true;
    }

    private void CALCULATE_TOTAL() { }
}
```

### ✅ After

```csharp
public class OrderProcessor
{
    private string customerName;
    private int orderId;
    public const int MaxItems = 100;

    public Result<Unit, ProcessOrderError> ProcessOrder(OrderId orderId)
    {
        var order = GetOrder(orderId);
        if (order == null)
        {
            return Result.Failure<Unit, ProcessOrderError>(
                new OrderNotFoundError(orderId));
        }

        CalculateTotal(order);
        return Result.Success<Unit, ProcessOrderError>(Unit.Value);
    }

    private void CalculateTotal(Order order) { }
}
```

## Naming Conventions

### 1. Use PascalCase for Public Members

```csharp
// ✅ PascalCase for classes, methods, properties
public class CustomerService
{
    public string FirstName { get; set; }
    public string LastName { get; set; }

    public void ProcessCustomer() { }
    public Result<Customer, CustomerError> GetCustomer(CustomerId id) { }
}
```

### 2. Use camelCase for Private Fields and Local Variables

```csharp
// ✅ camelCase for private fields (no underscore prefix per style guide)
public class OrderService
{
    private readonly IOrderRepository orderRepository;
    private readonly ILogger<OrderService> logger;

    public void ProcessOrder(OrderId orderId)
    {
        var order = this.orderRepository.Find(orderId);
        var total = CalculateTotal(order);
    }
}
```

### 3. Use PascalCase for Constants

```csharp
// ❌ ALL_CAPS (C/C++ style)
private const string DATABASE_CONNECTION_STRING = "...";

// ✅ PascalCase
private const string DatabaseConnectionString = "...";
public const int MaxRetryAttempts = 3;
```

### 4. Use 'I' Prefix for Interfaces

```csharp
// ✅ Interface naming
public interface IOrderRepository
{
    Order? Find(OrderId id);
}

public interface IOrderService
{
    Task<Result<Unit, OrderError>> ProcessOrderAsync(OrderId id);
}
```

### 5. Avoid Type Prefixes (Hungarian Notation)

```csharp
// ❌ Hungarian notation (outdated)
string strName;
int iCount;
bool bIsActive;
List<Order> lstOrders;

// ✅ Clean names
string name;
int count;
bool isActive;
List<Order> orders;
```

### 6. Use Descriptive Names for Booleans

```csharp
// ❌ Unclear boolean names
bool flag;
bool status;
bool check;

// ✅ Clear intention
bool isActive;
bool hasPermission;
bool canProcess;
bool shouldRetry;
```

### 7. Use Verb Phrases for Methods

```csharp
// ❌ Noun methods
public void Customer() { }
public void Data() { }

// ✅ Verb phrases
public void CreateCustomer() { }
public void ProcessOrder() { }
public void ValidateInput() { }
public bool CanExecute() { }
public Order GetOrder() { }
```

### 8. Use Noun Phrases for Properties

```csharp
// ✅ Property naming
public class Order
{
    public OrderId Id { get; }
    public CustomerId CustomerId { get; }
    public DateTime CreatedAt { get; }
    public OrderStatus Status { get; private set; }
    public Money Total { get; private set; }
}
```

### 9. Use Async Suffix for Async Methods

```csharp
// ✅ Async suffix
public async Task<Order> GetOrderAsync(OrderId id)
{
    return await _repository.GetOrderAsync(id);
}

public async Task ProcessOrderAsync(Order order)
{
    await _processor.ProcessAsync(order);
}
```

### 10. Use Meaningful Exception Names

```csharp
// ❌ Generic names
public class CustomException : Exception { }

// ✅ Specific, descriptive names
public class OrderNotFoundException : Exception { }
public class PaymentFailedException : Exception { }
public class InvalidOrderStatusException : Exception { }
```

### 11. Use Plural for Collections

```csharp
// ❌ Singular for collections
public List<Order> orderList;
public IEnumerable<Customer> customerCollection;

// ✅ Plural
public List<Order> orders;
public IEnumerable<Customer> customers;
public Dictionary<OrderId, Order> ordersById;
```

### 12. Avoid Abbreviations

```csharp
// ❌ Cryptic abbreviations
var cust = GetCustomer();
var ord = ProcessOrder();
var qty = GetQuantity();

// ✅ Full words
var customer = GetCustomer();
var order = ProcessOrder();
var quantity = GetQuantity();

// ✅ Acceptable common abbreviations
var id = GetId();
var max = GetMaximum();
var min = GetMinimum();
```

## File Naming

```csharp
// ✅ File name matches type name
OrderService.cs          // Contains OrderService class
IOrderRepository.cs      // Contains IOrderRepository interface
OrderDto.cs             // Contains OrderDto record
ProcessOrderCommand.cs  // Contains ProcessOrderCommand
```

## Namespace Conventions

```csharp
// ✅ PascalCase namespaces matching folder structure
namespace ECommerce.Orders.Domain
{
    public class Order { }
}

namespace ECommerce.Orders.Application.Commands
{
    public record PlaceOrderCommand { }
}

namespace ECommerce.SharedKernel.ValueObjects
{
    public sealed record Money { }
}
```

## Symptoms

- Code hard to read and understand
- Confusion about member accessibility
- Inconsistent style across codebase
- Difficulty finding files and types
- Unclear intent from names

## Benefits

- **Readable code** with consistent conventions
- **Faster navigation** with predictable naming
- **Clear intent** from descriptive names
- **Team alignment** on style
- **Professional appearance** of codebase

## See Also

- [Meaningful Names](./meaningful-names.md) — Intent-revealing names
- [Ubiquitous Language](./ubiquitous-language.md) — Domain terminology
- [Code Organization](./code-organization-patterns.md) — File structure
