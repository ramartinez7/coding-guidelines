# Testing Event Raising

> Verify that events are raised correctly, with proper event arguments and timing using FluentAssertions event monitoring.

## Problem

Testing event-driven code requires verifying that events are raised at the right time, with correct arguments, and in the proper sequence. Manual event handling in tests is verbose and error-prone.

## Pattern

Use FluentAssertions' event monitoring to test that events are raised, verify event arguments, and check event sequence and timing.

## Example

### ❌ Before - Manual Event Testing

```csharp
[TestMethod]
public void ProcessOrder_RaisesEvent()
{
    bool eventRaised = false;
    var processor = new OrderProcessor();
    processor.OrderProcessed += (s, e) => eventRaised = true;
    
    processor.Process(order);
    
    Assert.IsTrue(eventRaised);
    // ❌ Verbose and doesn't verify event arguments
}
```

### ✅ After - FluentAssertions Event Monitoring

```csharp
using FluentAssertions;
using FluentAssertions.Events;
using Microsoft.VisualStudio.TestTools.UnitTesting;

public class OrderProcessor
{
    public event EventHandler<OrderProcessedEventArgs>? OrderProcessed;

    public Result Process(Order order)
    {
        // Processing logic...
        OnOrderProcessed(new OrderProcessedEventArgs(order.Id, DateTime.Now));
        return Result.Success();
    }

    protected virtual void OnOrderProcessed(OrderProcessedEventArgs e)
    {
        OrderProcessed?.Invoke(this, e);
    }
}

[TestClass]
public class OrderProcessorEventTests
{
    [TestMethod]
    public void Process_RaisesOrderProcessedEvent()
    {
        // Arrange
        var processor = new OrderProcessor();
        var order = CreateOrder();

        using var monitoredProcessor = processor.Monitor();

        // Act
        processor.Process(order);

        // Assert
        monitoredProcessor.Should().Raise(nameof(OrderProcessor.OrderProcessed));
    }

    [TestMethod]
    public void Process_RaisesEventWithCorrectArguments()
    {
        // Arrange
        var processor = new OrderProcessor();
        var order = CreateOrder();

        using var monitoredProcessor = processor.Monitor();

        // Act
        processor.Process(order);

        // Assert
        monitoredProcessor.Should()
            .Raise(nameof(OrderProcessor.OrderProcessed))
            .WithArgs<OrderProcessedEventArgs>(args => 
                args.OrderId == order.Id
            );
    }

    [TestMethod]
    public void Process_RaisesEventOnce()
    {
        // Arrange
        var processor = new OrderProcessor();
        var order = CreateOrder();

        using var monitoredProcessor = processor.Monitor();

        // Act
        processor.Process(order);

        // Assert
        monitoredProcessor.Should()
            .Raise(nameof(OrderProcessor.OrderProcessed))
            .WithSender(processor);
    }
}
```

## Testing Event Arguments

```csharp
public class OrderProcessedEventArgs : EventArgs
{
    public OrderId OrderId { get; }
    public DateTime ProcessedAt { get; }
    public decimal Total { get; }

    public OrderProcessedEventArgs(OrderId orderId, DateTime processedAt, decimal total)
    {
        OrderId = orderId;
        ProcessedAt = processedAt;
        Total = total;
    }
}

[TestClass]
public class EventArgumentsTests
{
    [TestMethod]
    public void Process_RaisesEventWithAllExpectedArguments()
    {
        // Arrange
        var processor = new OrderProcessor();
        var order = CreateOrder(total: 100m);

        using var monitoredProcessor = processor.Monitor();

        // Act
        var processedAt = DateTime.Now;
        processor.Process(order);

        // Assert
        monitoredProcessor.Should()
            .Raise(nameof(OrderProcessor.OrderProcessed))
            .WithArgs<OrderProcessedEventArgs>(args =>
                args.OrderId == order.Id &&
                args.Total == 100m &&
                args.ProcessedAt >= processedAt
            );
    }

    [TestMethod]
    public void Process_EventArgsContainExpectedData()
    {
        // Arrange
        var processor = new OrderProcessor();
        var order = CreateOrder();

        using var monitoredProcessor = processor.Monitor();

        // Act
        processor.Process(order);

        // Assert
        var raisedEvent = monitoredProcessor.OccurredEvents
            .Should().ContainSingle(e => e.EventName == nameof(OrderProcessor.OrderProcessed))
            .Which;

        var eventArgs = raisedEvent.Parameters.OfType<OrderProcessedEventArgs>().First();
        eventArgs.OrderId.Should().Be(order.Id);
        eventArgs.Total.Should().BePositive();
    }
}
```

## Testing Event Sequence

```csharp
public class OrderWorkflow
{
    public event EventHandler<OrderEventArgs>? OrderValidated;
    public event EventHandler<OrderEventArgs>? OrderProcessed;
    public event EventHandler<OrderEventArgs>? OrderCompleted;

    public Result Execute(Order order)
    {
        OnOrderValidated(new OrderEventArgs(order.Id));
        OnOrderProcessed(new OrderEventArgs(order.Id));
        OnOrderCompleted(new OrderEventArgs(order.Id));
        return Result.Success();
    }

    // Event raising methods omitted for brevity
}

[TestClass]
public class EventSequenceTests
{
    [TestMethod]
    public void Execute_RaisesEventsInCorrectOrder()
    {
        // Arrange
        var workflow = new OrderWorkflow();
        var order = CreateOrder();

        using var monitoredWorkflow = workflow.Monitor();

        // Act
        workflow.Execute(order);

        // Assert
        monitoredWorkflow.Should()
            .Raise(nameof(OrderWorkflow.OrderValidated))
            .Raise(nameof(OrderWorkflow.OrderProcessed))
            .Raise(nameof(OrderWorkflow.OrderCompleted));

        var events = monitoredWorkflow.OccurredEvents.ToList();
        events[0].EventName.Should().Be(nameof(OrderWorkflow.OrderValidated));
        events[1].EventName.Should().Be(nameof(OrderWorkflow.OrderProcessed));
        events[2].EventName.Should().Be(nameof(OrderWorkflow.OrderCompleted));
    }

    [TestMethod]
    public void Execute_RaisesAllThreeEvents()
    {
        // Arrange
        var workflow = new OrderWorkflow();
        var order = CreateOrder();

        using var monitoredWorkflow = workflow.Monitor();

        // Act
        workflow.Execute(order);

        // Assert
        monitoredWorkflow.OccurredEvents.Should().HaveCount(3);
    }
}
```

## Testing Events Not Raised

```csharp
[TestClass]
public class EventNotRaisedTests
{
    [TestMethod]
    public void Process_WithInvalidOrder_DoesNotRaiseEvent()
    {
        // Arrange
        var processor = new OrderProcessor();
        var invalidOrder = CreateInvalidOrder();

        using var monitoredProcessor = processor.Monitor();

        // Act
        var result = processor.Process(invalidOrder);

        // Assert
        result.IsFailure.Should().BeTrue();
        monitoredProcessor.Should().NotRaise(nameof(OrderProcessor.OrderProcessed));
    }

    [TestMethod]
    public void Cancel_DoesNotRaiseProcessedEvent()
    {
        // Arrange
        var processor = new OrderProcessor();
        var order = CreateOrder();

        using var monitoredProcessor = processor.Monitor();

        // Act
        processor.Cancel(order);

        // Assert
        monitoredProcessor.Should().NotRaise(nameof(OrderProcessor.OrderProcessed));
    }
}
```

## Testing PropertyChanged Events

```csharp
public class Customer : INotifyPropertyChanged
{
    private string _email;

    public string Email
    {
        get => _email;
        set
        {
            if (_email != value)
            {
                _email = value;
                OnPropertyChanged(nameof(Email));
            }
        }
    }

    public event PropertyChangedEventHandler? PropertyChanged;

    protected virtual void OnPropertyChanged(string propertyName)
    {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
    }
}

[TestClass]
public class PropertyChangedEventTests
{
    [TestMethod]
    public void SetEmail_RaisesPropertyChangedEvent()
    {
        // Arrange
        var customer = new Customer { Email = "old@example.com" };

        using var monitoredCustomer = customer.Monitor();

        // Act
        customer.Email = "new@example.com";

        // Assert
        monitoredCustomer.Should()
            .RaisePropertyChangeFor(c => c.Email);
    }

    [TestMethod]
    public void SetEmail_WithSameValue_DoesNotRaiseEvent()
    {
        // Arrange
        var customer = new Customer { Email = "test@example.com" };

        using var monitoredCustomer = customer.Monitor();

        // Act
        customer.Email = "test@example.com";

        // Assert
        monitoredCustomer.Should()
            .NotRaisePropertyChangeFor(c => c.Email);
    }

    [TestMethod]
    public void SetMultipleProperties_RaisesEventsForEach()
    {
        // Arrange
        var customer = new Customer();

        using var monitoredCustomer = customer.Monitor();

        // Act
        customer.Email = "test@example.com";
        customer.FirstName = "John";
        customer.LastName = "Doe";

        // Assert
        monitoredCustomer.Should()
            .RaisePropertyChangeFor(c => c.Email)
            .RaisePropertyChangeFor(c => c.FirstName)
            .RaisePropertyChangeFor(c => c.LastName);
    }
}
```

## Testing Event Handler Count

```csharp
[TestClass]
public class EventHandlerTests
{
    [TestMethod]
    public void Subscribe_IncreasesHandlerCount()
    {
        // Arrange
        var processor = new OrderProcessor();
        var handlerCalled = false;

        using var monitoredProcessor = processor.Monitor();

        // Act
        processor.OrderProcessed += (s, e) => handlerCalled = true;
        processor.Process(CreateOrder());

        // Assert
        handlerCalled.Should().BeTrue();
        monitoredProcessor.Should().Raise(nameof(OrderProcessor.OrderProcessed));
    }

    [TestMethod]
    public void Unsubscribe_PreventsHandlerExecution()
    {
        // Arrange
        var processor = new OrderProcessor();
        var handlerCalled = false;
        EventHandler<OrderProcessedEventArgs> handler = (s, e) => handlerCalled = true;

        processor.OrderProcessed += handler;
        processor.OrderProcessed -= handler;

        // Act
        processor.Process(CreateOrder());

        // Assert
        handlerCalled.Should().BeFalse();
    }
}
```

## Domain Event Testing

```csharp
public abstract class DomainEvent
{
    public DateTime OccurredAt { get; } = DateTime.Now;
}

public class OrderPlacedEvent : DomainEvent
{
    public OrderId OrderId { get; }
    public CustomerId CustomerId { get; }

    public OrderPlacedEvent(OrderId orderId, CustomerId customerId)
    {
        OrderId = orderId;
        CustomerId = customerId;
    }
}

public class Order
{
    private readonly List<DomainEvent> _domainEvents = new();

    public IReadOnlyList<DomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    public static Result<Order> Create(OrderId id, CustomerId customerId, Money total)
    {
        var order = new Order(id, customerId, total);
        order.AddDomainEvent(new OrderPlacedEvent(id, customerId));
        return Result.Success(order);
    }

    private void AddDomainEvent(DomainEvent domainEvent)
    {
        _domainEvents.Add(domainEvent);
    }
}

[TestClass]
public class DomainEventTests
{
    [TestMethod]
    public void Create_AddsDomainEvent()
    {
        // Arrange
        var orderId = OrderId.New();
        var customerId = CustomerId.New();

        // Act
        var result = Order.Create(orderId, customerId, Money.USD(100m));

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.DomainEvents.Should().ContainSingle();
        result.Value.DomainEvents.Should().ContainItemsAssignableTo<OrderPlacedEvent>();
    }

    [TestMethod]
    public void Create_DomainEventContainsCorrectData()
    {
        // Arrange
        var orderId = OrderId.New();
        var customerId = CustomerId.New();

        // Act
        var result = Order.Create(orderId, customerId, Money.USD(100m));

        // Assert
        var domainEvent = result.Value.DomainEvents.OfType<OrderPlacedEvent>().First();
        domainEvent.OrderId.Should().Be(orderId);
        domainEvent.CustomerId.Should().Be(customerId);
        domainEvent.OccurredAt.Should().BeCloseTo(DateTime.Now, TimeSpan.FromSeconds(1));
    }
}
```

## Why It's Important

1. **Event Contracts**: Verify events are raised as documented
2. **Integration**: Test event-driven communication
3. **Side Effects**: Verify observable behavior
4. **Sequence**: Ensure events occur in correct order
5. **Data**: Validate event arguments contain correct information

## Guidelines

**Event Monitoring Setup**
- Use `Monitor()` extension to create monitored object
- Wrap monitor in `using` statement for proper disposal
- Monitor before executing action that raises event

**Common Event Assertions**
- `.Should().Raise(eventName)` - event was raised
- `.Should().NotRaise(eventName)` - event was not raised
- `.WithArgs<T>(predicate)` - verify event arguments
- `.WithSender(sender)` - verify event sender
- `.RaisePropertyChangeFor(property)` - PropertyChanged event

**Best Practices**
- Test both positive (event raised) and negative (not raised) cases
- Verify event arguments, not just that event occurred
- Test event sequence when multiple events are involved
- Use specific event arg types for type safety

## Common Pitfalls

❌ **Not monitoring before action**
```csharp
// Wrong - monitor after event
processor.Process(order);
using var monitor = processor.Monitor();
```

✅ **Monitor before action**
```csharp
// Correct
using var monitor = processor.Monitor();
processor.Process(order);
```

❌ **Not verifying event arguments**
```csharp
// Incomplete
monitor.Should().Raise("OrderProcessed");
```

✅ **Verify arguments**
```csharp
// Complete
monitor.Should()
    .Raise("OrderProcessed")
    .WithArgs<OrderProcessedEventArgs>(args => 
        args.OrderId == expectedId
    );
```

## See Also

- [Testing Side Effects](./testing-side-effects.md) — observable effects
- [Testing Async Code](./testing-async-code.md) — async event handling
- [Domain Events](../patterns/domain-events.md) — domain event pattern
- [Event Sourcing](../patterns/type-safe-event-sourcing.md) — event-driven architecture
