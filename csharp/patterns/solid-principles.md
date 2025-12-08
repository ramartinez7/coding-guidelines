# SOLID Principles

> Five principles for creating maintainable, flexible object-oriented designs in C#.

## Overview

SOLID is an acronym for five design principles that make software designs more understandable, flexible, and maintainable:

| Principle | Description |
|-----------|-------------|
| **S**ingle Responsibility | A class should have one reason to change |
| **O**pen/Closed | Open for extension, closed for modification |
| **L**iskov Substitution | Subtypes must be substitutable for base types |
| **I**nterface Segregation | Many specific interfaces better than one general |
| **D**ependency Inversion | Depend on abstractions, not concretions |

---

## Single Responsibility Principle (SRP)

> A class should have one, and only one, reason to change.

### Problem

Classes with multiple responsibilities are harder to understand, test, and change. Changes to one responsibility affect code for other responsibilities.

### ❌ Before

```csharp
public class Employee
{
    public EmployeeId Id { get; set; }
    public string Name { get; set; }
    public decimal Salary { get; set; }

    // Responsibility 1: Calculate pay
    public decimal CalculatePay()
    {
        return Salary * 1.1m;  // 10% bonus
    }

    // Responsibility 2: Save to database
    public void Save()
    {
        var connection = new SqlConnection("...");
        // Save employee...
    }

    // Responsibility 3: Generate report
    public string GenerateReport()
    {
        return $"Employee Report: {Name}, Salary: {Salary}";
    }
}
```

**Problems:**
- Changes to pay calculation affect the entire class
- Changes to database logic affect the entire class
- Changes to report format affect the entire class
- Hard to test each responsibility in isolation

### ✅ After

```csharp
// Responsibility 1: Employee domain model
public sealed record Employee(EmployeeId Id, Name Name, Money Salary);

// Responsibility 2: Pay calculation
public class PayrollCalculator
{
    public Money CalculatePay(Employee employee)
    {
        return employee.Salary.Multiply(1.1m);  // 10% bonus
    }
}

// Responsibility 3: Persistence
public class EmployeeRepository
{
    private readonly IDbConnection connection;

    public void Save(Employee employee)
    {
        // Save logic
    }
}

// Responsibility 4: Reporting
public class EmployeeReportGenerator
{
    public string GenerateReport(Employee employee)
    {
        return $"Employee Report: {employee.Name}, Salary: {employee.Salary}";
    }
}
```

---

## Open/Closed Principle (OCP)

> Software entities should be open for extension but closed for modification.

### Problem

Adding new functionality requires modifying existing code, risking bugs in working features.

### ❌ Before

```csharp
public class OrderProcessor
{
    public decimal CalculateDiscount(Order order, string customerType)
    {
        return customerType switch
        {
            "Regular" => order.Total * 0.05m,
            "Premium" => order.Total * 0.10m,
            "VIP" => order.Total * 0.15m,
            _ => 0m
        };
    }
}
```

**Problems:**
- Adding new customer types requires modifying `CalculateDiscount`
- Risk of breaking existing discount logic
- Violates type safety (string-based types)

### ✅ After

```csharp
// Abstract discount policy
public abstract record DiscountPolicy
{
    public abstract Money CalculateDiscount(Money orderTotal);
}

// Concrete policies (closed for modification)
public sealed record RegularDiscount() : DiscountPolicy
{
    public override Money CalculateDiscount(Money orderTotal)
        => orderTotal.Multiply(0.05m);
}

public sealed record PremiumDiscount() : DiscountPolicy
{
    public override Money CalculateDiscount(Money orderTotal)
        => orderTotal.Multiply(0.10m);
}

public sealed record VipDiscount() : DiscountPolicy
{
    public override Money CalculateDiscount(Money orderTotal)
        => orderTotal.Multiply(0.15m);
}

// Processor works with abstraction (open for extension)
public class OrderProcessor
{
    public Money CalculateDiscount(Order order, DiscountPolicy policy)
    {
        return policy.CalculateDiscount(order.Total);
    }
}

// Adding new discount type doesn't modify existing code
public sealed record CorporateDiscount() : DiscountPolicy
{
    public override Money CalculateDiscount(Money orderTotal)
        => orderTotal.Multiply(0.20m);
}
```

---

## Liskov Substitution Principle (LSP)

> Objects of a superclass should be replaceable with objects of a subclass without breaking the application.

### Problem

Subtypes that violate base class contracts force callers to special-case specific implementations.

### ❌ Before

```csharp
public class Rectangle
{
    public virtual int Width { get; set; }
    public virtual int Height { get; set; }

    public int Area() => Width * Height;
}

public class Square : Rectangle
{
    public override int Width
    {
        get => base.Width;
        set
        {
            base.Width = value;
            base.Height = value;  // Violates LSP: unexpected side effect
        }
    }

    public override int Height
    {
        get => base.Height;
        set
        {
            base.Width = value;
            base.Height = value;  // Violates LSP: unexpected side effect
        }
    }
}

// This breaks when using Square
void ResizeRectangle(Rectangle rect)
{
    rect.Width = 5;
    rect.Height = 10;
    Console.WriteLine(rect.Area());  // Expects 50, but Square gives 100
}
```

### ✅ After

```csharp
// Option 1: Use separate types instead of inheritance
public sealed record Rectangle(int Width, int Height)
{
    public int Area => Width * Height;
}

public sealed record Square(int Side)
{
    public int Area => Side * Side;
}

// Option 2: Use a common abstraction if polymorphism is needed
public interface IShape
{
    int Area { get; }
}

public sealed record RectangleShape(int Width, int Height) : IShape
{
    public int Area => Width * Height;
}

public sealed record SquareShape(int Side) : IShape
{
    public int Area => Side * Side;
}
```

**Key Point:** Don't force inheritance where it doesn't fit. Not every "is-a" relationship in English translates to good inheritance in code.

---

## Interface Segregation Principle (ISP)

> Clients should not be forced to depend on interfaces they do not use.

### Problem

Large, general-purpose interfaces force implementers to implement methods they don't need.

### ❌ Before

```csharp
public interface IRepository
{
    void Add(Entity entity);
    void Update(Entity entity);
    void Delete(EntityId id);
    Entity FindById(EntityId id);
    IEnumerable<Entity> GetAll();
    IEnumerable<Entity> Search(string query);
    void BulkInsert(IEnumerable<Entity> entities);
    void Archive(EntityId id);
}

// Read-only cache doesn't need write operations
public class ReadOnlyCache : IRepository
{
    public void Add(Entity entity) => throw new NotSupportedException();
    public void Update(Entity entity) => throw new NotSupportedException();
    public void Delete(EntityId id) => throw new NotSupportedException();
    public void BulkInsert(IEnumerable<Entity> entities) => throw new NotSupportedException();
    public void Archive(EntityId id) => throw new NotSupportedException();

    public Entity FindById(EntityId id) { /* Implementation */ }
    public IEnumerable<Entity> GetAll() { /* Implementation */ }
    public IEnumerable<Entity> Search(string query) { /* Implementation */ }
}
```

### ✅ After

```csharp
// Segregated interfaces
public interface IReadRepository
{
    Entity? FindById(EntityId id);
    IEnumerable<Entity> GetAll();
}

public interface ISearchRepository
{
    IEnumerable<Entity> Search(string query);
}

public interface IWriteRepository
{
    void Add(Entity entity);
    void Update(Entity entity);
    void Delete(EntityId id);
}

public interface IBulkRepository
{
    void BulkInsert(IEnumerable<Entity> entities);
}

// Implementations only implement what they need
public class ReadOnlyCache : IReadRepository, ISearchRepository
{
    public Entity? FindById(EntityId id) { /* Implementation */ }
    public IEnumerable<Entity> GetAll() { /* Implementation */ }
    public IEnumerable<Entity> Search(string query) { /* Implementation */ }
}

// Full repository can compose interfaces
public class FullRepository : IReadRepository, IWriteRepository, IBulkRepository
{
    // Implements all operations
}
```

---

## Dependency Inversion Principle (DIP)

> High-level modules should not depend on low-level modules. Both should depend on abstractions.

### Problem

Direct dependencies on concrete implementations make code rigid and hard to test.

### ❌ Before

```csharp
public class OrderProcessor
{
    private readonly SqlOrderRepository repository;
    private readonly SmtpEmailService emailService;
    private readonly StripePaymentGateway paymentGateway;

    public OrderProcessor()
    {
        // Tightly coupled to concrete implementations
        this.repository = new SqlOrderRepository();
        this.emailService = new SmtpEmailService();
        this.paymentGateway = new StripePaymentGateway();
    }

    public void ProcessOrder(Order order)
    {
        repository.Save(order);
        paymentGateway.Charge(order.Total);
        emailService.Send(order.CustomerEmail, "Order confirmed");
    }
}
```

**Problems:**
- Cannot test without a real database, SMTP server, and Stripe account
- Cannot swap implementations (e.g., use PostgreSQL instead of SQL Server)
- Cannot mock dependencies

### ✅ After

```csharp
// Depend on abstractions
public interface IOrderRepository
{
    void Save(Order order);
}

public interface IEmailService
{
    void Send(Email recipient, string subject, string body);
}

public interface IPaymentGateway
{
    Result<PaymentConfirmation, PaymentError> Charge(Money amount, PaymentMethod method);
}

public class OrderProcessor
{
    private readonly IOrderRepository repository;
    private readonly IEmailService emailService;
    private readonly IPaymentGateway paymentGateway;

    // Dependencies injected (Dependency Injection)
    public OrderProcessor(
        IOrderRepository repository,
        IEmailService emailService,
        IPaymentGateway paymentGateway)
    {
        this.repository = repository;
        this.emailService = emailService;
        this.paymentGateway = paymentGateway;
    }

    public Result<Unit, ProcessOrderError> ProcessOrder(Order order)
    {
        repository.Save(order);
        
        var paymentResult = paymentGateway.Charge(order.Total, order.PaymentMethod);
        if (paymentResult.IsFailure)
            return Result.Failure<Unit, ProcessOrderError>(new PaymentFailed());

        emailService.Send(order.CustomerEmail, "Order confirmed", "...");
        return Result.Success<Unit, ProcessOrderError>(Unit.Value);
    }
}

// Concrete implementations
public class SqlOrderRepository : IOrderRepository { }
public class SmtpEmailService : IEmailService { }
public class StripePaymentGateway : IPaymentGateway { }

// Easy to test with mocks
public class TestOrderProcessor
{
    public void Test_ProcessOrder_Success()
    {
        var mockRepository = new MockOrderRepository();
        var mockEmail = new MockEmailService();
        var mockPayment = new MockPaymentGateway();

        var processor = new OrderProcessor(mockRepository, mockEmail, mockPayment);
        
        // Test in isolation
    }
}
```

---

## Benefits

- **Single Responsibility**: Easier to understand, test, and change
- **Open/Closed**: Add features without modifying existing code
- **Liskov Substitution**: Polymorphism works correctly
- **Interface Segregation**: Smaller, focused contracts
- **Dependency Inversion**: Testable, flexible, maintainable code

## See Also

- [Small Functions](./small-functions.md) — Applying SRP to methods
- [Enum to Class Hierarchy](./enum-to-class-hierarchy.md) — Applying OCP with polymorphism
- [Flag Arguments](./flag-arguments.md) — Replacing flags with polymorphism (OCP)
- [Repository Pattern](./repository-pattern.md) — Applying DIP to data access
