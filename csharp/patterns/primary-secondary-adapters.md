# Primary vs Secondary Adapters

> Primary adapters drive the application (HTTP controllers, CLI, message consumers), while secondary adapters are driven by the application (database repositories, email services, external APIs)—understanding the direction of dependency flow is key to hexagonal architecture.

## Problem

Confusion between primary and secondary adapters leads to incorrect dependency directions and violated architectural boundaries.

## Core Concept

**Primary (Driving) Adapters:**
- Initiate actions in the application
- Call into the application core
- Examples: HTTP controllers, CLI commands, message queue consumers, scheduled jobs

**Secondary (Driven) Adapters:**
- Called by the application core  
- Implement infrastructure concerns
- Examples: database repositories, email services, external APIs, file storage

```
Primary Adapters          Application Core          Secondary Adapters
     (HTTP)      ──────►   [Use Cases]   ──────►    (Database)
     (CLI)       ──────►   [Commands]    ──────►    (Email)
   (Messages)    ──────►   [Queries]     ──────►    (External API)
```

## Example

### Primary Adapters (Driving)

```csharp
// WebApi/Adapters/Http/OrdersController.cs (Primary Adapter)
namespace MyApp.WebApi.Adapters.Http
{
    /// <summary>
    /// Primary adapter - translates HTTP to application commands.
    /// Drives the application core.
    /// </summary>
    [ApiController]
    [Route("api/orders")]
    public class OrdersController : ControllerBase
    {
        private readonly PlaceOrderCommandHandler _placeOrderHandler;
        
        // ✅ Depends on primary port (command handler)
        public OrdersController(PlaceOrderCommandHandler placeOrderHandler)
        {
            _placeOrderHandler = placeOrderHandler;
        }
        
        [HttpPost]
        public async Task<IActionResult> PlaceOrder([FromBody] PlaceOrderRequest request)
        {
            // ✅ Translates HTTP to command
            var command = new PlaceOrderCommand(
                new CustomerId(request.CustomerId),
                request.Items.Select(i => new OrderItemCommand(
                    new ProductId(i.ProductId),
                    i.Quantity)).ToList());
            
            // ✅ Drives application by invoking command handler
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

// Console/Adapters/Cli/PlaceOrderCliAdapter.cs (Primary Adapter)
namespace MyApp.Console.Adapters.Cli
{
    /// <summary>
    /// Primary adapter - translates CLI arguments to application commands.
    /// </summary>
    public sealed class PlaceOrderCliAdapter
    {
        private readonly PlaceOrderCommandHandler _placeOrderHandler;
        
        public PlaceOrderCliAdapter(PlaceOrderCommandHandler placeOrderHandler)
        {
            _placeOrderHandler = placeOrderHandler;
        }
        
        public async Task<int> ExecuteAsync(string[] args)
        {
            // Parse CLI arguments
            var customerId = Guid.Parse(args[0]);
            
            var command = new PlaceOrderCommand(
                new CustomerId(customerId),
                new List<OrderItemCommand>());
            
            // Drive application
            var result = await _placeOrderHandler.HandleAsync(command);
            
            return result.Match(
                onSuccess: _ => 0,
                onFailure: _ => 1);
        }
    }
}

// MessageHandlers/OrderPlacedMessageHandler.cs (Primary Adapter)
namespace MyApp.MessageHandlers
{
    /// <summary>
    /// Primary adapter - consumes messages and drives application.
    /// </summary>
    public sealed class OrderPlacedMessageHandler : IMessageHandler<OrderPlacedMessage>
    {
        private readonly ProcessOrderCommandHandler _processOrderHandler;
        
        public async Task HandleAsync(OrderPlacedMessage message)
        {
            var command = new ProcessOrderCommand(new OrderId(message.OrderId));
            
            // Drive application
            await _processOrderHandler.HandleAsync(command);
        }
    }
}
```

### Secondary Adapters (Driven)

```csharp
// Infrastructure/Adapters/Persistence/SqlOrderRepository.cs (Secondary Adapter)
namespace MyApp.Infrastructure.Adapters.Persistence
{
    /// <summary>
    /// Secondary adapter - implements IOrderRepository port.
    /// Driven by the application core.
    /// </summary>
    public sealed class SqlOrderRepository : IOrderRepository
    {
        private readonly MyDbContext _dbContext;
        
        public async Task<Option<Order>> GetByIdAsync(OrderId id)
        {
            // Driven by application - called when core needs data
            var entity = await _dbContext.Orders
                .Include(o => o.Items)
                .FirstOrDefaultAsync(o => o.Id == id.Value);
            
            if (entity == null)
                return Option<Order>.None;
            
            var order = MapToDomain(entity);
            return Option<Order>.Some(order);
        }
        
        public async Task SaveAsync(Order order)
        {
            // Driven by application - called when core needs to persist
            var entity = MapToEntity(order);
            _dbContext.Orders.Update(entity);
        }
    }
}

// Infrastructure/Adapters/Email/SendGridEmailAdapter.cs (Secondary Adapter)
namespace MyApp.Infrastructure.Adapters.Email
{
    /// <summary>
    /// Secondary adapter - implements IEmailService port.
    /// Driven by the application core.
    /// </summary>
    public sealed class SendGridEmailAdapter : IEmailService
    {
        private readonly ISendGridClient _sendGridClient;
        
        public async Task SendOrderConfirmationAsync(OrderId orderId, Email recipientEmail)
        {
            // Driven by application - called when core needs to send email
            var message = new SendGridMessage
            {
                Subject = "Order Confirmation",
                PlainTextContent = $"Your order {orderId} has been confirmed.",
                To = new List<EmailAddress> { new(recipientEmail.Value) }
            };
            
            await _sendGridClient.SendEmailAsync(message);
        }
    }
}

// Infrastructure/Adapters/Payment/StripePaymentAdapter.cs (Secondary Adapter)
namespace MyApp.Infrastructure.Adapters.Payment
{
    /// <summary>
    /// Secondary adapter - implements IPaymentGateway port.
    /// </summary>
    public sealed class StripePaymentAdapter : IPaymentGateway
    {
        private readonly IStripeClient _stripeClient;
        
        public async Task<Result<PaymentId, PaymentError>> ProcessPaymentAsync(
            Money amount,
            PaymentMethod paymentMethod)
        {
            // Driven by application - called when core needs to process payment
            var chargeOptions = new ChargeCreateOptions
            {
                Amount = (long)(amount.Amount * 100),
                Currency = amount.Currency.Code.ToLower(),
                Source = paymentMethod.Token
            };
            
            var charge = await _stripeClient.Charges.CreateAsync(chargeOptions);
            
            return Result<PaymentId, PaymentError>.Success(
                new PaymentId(charge.Id));
        }
    }
}
```

## Comparison

| Aspect | Primary Adapter | Secondary Adapter |
|--------|-----------------|-------------------|
| **Direction** | Drives application | Driven by application |
| **Initiates** | Actions | Nothing |
| **Depends on** | Application core | Application core |
| **Implements** | Entry points | Infrastructure ports |
| **Examples** | HTTP, CLI, Messages | Database, Email, APIs |
| **Layer** | Presentation | Infrastructure |

## Benefits

1. **Clear Boundaries**: Understand direction of dependencies
2. **Testability**: Mock secondary adapters, invoke primary adapters
3. **Flexibility**: Swap adapters without changing core
4. **Multiple Interfaces**: Support HTTP, CLI, messages using same core

## See Also

- [Port-Adapter Separation](./port-adapter-separation.md) — hexagonal architecture
- [Hexagonal Ports](./hexagonal-ports.md) — defining ports
- [Clean Architecture](./clean-architecture.md) — layer dependencies
