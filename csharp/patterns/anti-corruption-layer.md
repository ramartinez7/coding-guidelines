# Anti-Corruption Layer (Protecting Domain Boundaries)

> External systems with incompatible models pollute your domain—use an anti-corruption layer to translate and isolate foreign concepts.

## Problem

When your domain directly depends on external APIs, third-party libraries, or legacy systems, their models and concepts leak into your clean domain code. Changes in external systems break your domain, and testing becomes difficult.

## Example

### ❌ Before

```csharp
// External payment gateway models directly in domain
using Stripe;

public class OrderService
{
    private readonly StripeClient _stripeClient;
    
    public async Task<Order> ProcessOrderAsync(Order order)
    {
        // Domain logic tightly coupled to Stripe
        var options = new ChargeCreateOptions
        {
            Amount = (long)(order.Total * 100),  // Stripe uses cents
            Currency = "usd",
            Description = $"Order {order.Id}",
            Source = order.PaymentToken,  // Domain model has Stripe concept
            Metadata = new Dictionary<string, string>
            {
                { "order_id", order.Id.ToString() }
            }
        };
        
        var service = new ChargeService(_stripeClient);
        var charge = await service.CreateAsync(options);
        
        // Stripe's model in our domain
        order.StripeChargeId = charge.Id;
        order.StripeStatus = charge.Status;
        order.StripeReceiptUrl = charge.ReceiptUrl;
        
        return order;
    }
}

public class Order
{
    public OrderId Id { get; set; }
    public decimal Total { get; set; }
    public string PaymentToken { get; set; }  // Stripe concept
    
    // Domain polluted with Stripe-specific fields
    public string? StripeChargeId { get; set; }
    public string? StripeStatus { get; set; }
    public string? StripeReceiptUrl { get; set; }
}
```

**Problems:**
- Domain depends on Stripe library
- Switching payment providers requires domain changes
- Testing requires Stripe SDK mocks
- Stripe concepts leak into domain (`ChargeId`, `ReceiptUrl`)
- Domain tied to Stripe's representation (cents, status strings)

### ✅ After

```csharp
// ============================================
// DOMAIN LAYER (Clean, no external dependencies)
// ============================================
namespace Orders.Domain
{
    /// <summary>
    /// Pure domain model with no knowledge of external systems.
    /// </summary>
    public sealed class Order
    {
        private readonly List<OrderItem> _items = new();
        
        public OrderId Id { get; }
        public CustomerId CustomerId { get; }
        public Money Total { get; private set; }
        public OrderStatus Status { get; private set; }
        
        // Domain concept: payment confirmation
        public Option<PaymentConfirmation> PaymentConfirmation { get; private set; }
        
        public Result<Unit, string> ConfirmPayment(PaymentConfirmation confirmation)
        {
            if (Status != OrderStatus.PaymentPending)
                return Result<Unit, string>.Failure("Order not awaiting payment");
            
            PaymentConfirmation = Option<PaymentConfirmation>.Some(confirmation);
            Status = OrderStatus.PaymentConfirmed;
            
            return Result<Unit, string>.Success(Unit.Value);
        }
    }
    
    /// <summary>
    /// Domain concept: proof that payment was successful.
    /// No knowledge of which provider processed it.
    /// </summary>
    public sealed record PaymentConfirmation(
        TransactionId TransactionId,
        Money Amount,
        DateTimeOffset ConfirmedAt,
        Option<ReceiptUrl> Receipt);
    
    /// <summary>
    /// Domain service interface—implementation details hidden.
    /// </summary>
    public interface IPaymentProcessor
    {
        Task<Result<PaymentConfirmation, PaymentError>> ProcessPaymentAsync(
            PaymentRequest request);
    }
    
    public sealed record PaymentRequest(
        Money Amount,
        PaymentMethod Method,
        OrderId OrderId);
    
    public sealed record PaymentError(string Code, string Message);
}

// ============================================
// ANTI-CORRUPTION LAYER (Translation)
// ============================================
namespace Orders.Infrastructure.Payment
{
    using Stripe;
    using Orders.Domain;
    
    /// <summary>
    /// Anti-Corruption Layer: Translates between domain and Stripe.
    /// Isolates domain from Stripe's model and concepts.
    /// </summary>
    public sealed class StripePaymentProcessor : IPaymentProcessor
    {
        private readonly IStripeClient _stripeClient;
        
        public StripePaymentProcessor(IStripeClient stripeClient)
        {
            _stripeClient = stripeClient;
        }
        
        public async Task<Result<PaymentConfirmation, PaymentError>> ProcessPaymentAsync(
            PaymentRequest request)
        {
            try
            {
                // Translate domain model to Stripe model
                var chargeOptions = TranslateToStripeCharge(request);
                
                var service = new ChargeService(_stripeClient);
                var charge = await service.CreateAsync(chargeOptions);
                
                // Translate Stripe model back to domain
                return charge.Status switch
                {
                    "succeeded" => Result<PaymentConfirmation, PaymentError>.Success(
                        TranslateToDomainConfirmation(charge)),
                    
                    "pending" => Result<PaymentConfirmation, PaymentError>.Failure(
                        new PaymentError("PAYMENT_PENDING", "Payment is still processing")),
                    
                    "failed" => Result<PaymentConfirmation, PaymentError>.Failure(
                        new PaymentError("PAYMENT_FAILED", charge.FailureMessage ?? "Payment failed")),
                    
                    _ => Result<PaymentConfirmation, PaymentError>.Failure(
                        new PaymentError("UNKNOWN_STATUS", $"Unknown status: {charge.Status}"))
                };
            }
            catch (StripeException ex)
            {
                // Translate Stripe exceptions to domain errors
                return Result<PaymentConfirmation, PaymentError>.Failure(
                    TranslateStripeError(ex));
            }
        }
        
        /// <summary>
        /// Translation: Domain → Stripe
        /// </summary>
        private ChargeCreateOptions TranslateToStripeCharge(PaymentRequest request)
        {
            return new ChargeCreateOptions
            {
                // Stripe uses cents, domain uses decimal
                Amount = (long)(request.Amount.Amount * 100),
                Currency = TranslateCurrency(request.Amount.Currency),
                Description = $"Order {request.OrderId}",
                Source = ExtractStripeToken(request.Method),
                Metadata = new Dictionary<string, string>
                {
                    { "order_id", request.OrderId.Value.ToString() },
                    { "processed_at", DateTimeOffset.UtcNow.ToString("O") }
                }
            };
        }
        
        /// <summary>
        /// Translation: Stripe → Domain
        /// </summary>
        private PaymentConfirmation TranslateToDomainConfirmation(Charge charge)
        {
            var transactionId = new TransactionId(charge.Id);
            var amount = new Money(charge.Amount / 100m, ParseCurrency(charge.Currency));
            var confirmedAt = DateTimeOffset.FromUnixTimeSeconds(charge.Created);
            var receipt = string.IsNullOrEmpty(charge.ReceiptUrl)
                ? Option<ReceiptUrl>.None
                : Option<ReceiptUrl>.Some(new ReceiptUrl(charge.ReceiptUrl));
            
            return new PaymentConfirmation(transactionId, amount, confirmedAt, receipt);
        }
        
        private PaymentError TranslateStripeError(StripeException ex)
        {
            return ex.StripeError.Type switch
            {
                "card_error" => new PaymentError(
                    "CARD_DECLINED",
                    ex.StripeError.Message ?? "Card was declined"),
                
                "invalid_request_error" => new PaymentError(
                    "INVALID_REQUEST",
                    "Invalid payment request"),
                
                "api_error" => new PaymentError(
                    "PROVIDER_ERROR",
                    "Payment provider is unavailable"),
                
                _ => new PaymentError("UNKNOWN_ERROR", ex.Message)
            };
        }
        
        private string TranslateCurrency(Currency currency)
        {
            return currency switch
            {
                Currency.USD => "usd",
                Currency.EUR => "eur",
                Currency.GBP => "gbp",
                _ => throw new ArgumentOutOfRangeException(nameof(currency))
            };
        }
        
        private Currency ParseCurrency(string stripeCurrency)
        {
            return stripeCurrency.ToLowerInvariant() switch
            {
                "usd" => Currency.USD,
                "eur" => Currency.EUR,
                "gbp" => Currency.GBP,
                _ => throw new ArgumentException($"Unknown currency: {stripeCurrency}")
            };
        }
        
        private string ExtractStripeToken(PaymentMethod method)
        {
            // Extract Stripe-specific token from domain payment method
            return method switch
            {
                CreditCardPayment cc => cc.Token,
                _ => throw new NotSupportedException($"Payment method not supported: {method.GetType()}")
            };
        }
    }
}

// ============================================
// TESTING (Easy now—no Stripe dependency)
// ============================================
namespace Orders.Tests
{
    public class FakePaymentProcessor : IPaymentProcessor
    {
        public PaymentConfirmation? ConfirmationToReturn { get; set; }
        public PaymentError? ErrorToReturn { get; set; }
        
        public Task<Result<PaymentConfirmation, PaymentError>> ProcessPaymentAsync(
            PaymentRequest request)
        {
            if (ErrorToReturn != null)
                return Task.FromResult(
                    Result<PaymentConfirmation, PaymentError>.Failure(ErrorToReturn));
            
            var confirmation = ConfirmationToReturn ?? new PaymentConfirmation(
                new TransactionId($"TEST_{Guid.NewGuid()}"),
                request.Amount,
                DateTimeOffset.UtcNow,
                Option<ReceiptUrl>.None);
            
            return Task.FromResult(
                Result<PaymentConfirmation, PaymentError>.Success(confirmation));
        }
    }
    
    [Fact]
    public async Task ProcessOrder_SuccessfulPayment_ConfirmsOrder()
    {
        // No Stripe mocking needed!
        var processor = new FakePaymentProcessor
        {
            ConfirmationToReturn = new PaymentConfirmation(
                new TransactionId("test_123"),
                Money.USD(100),
                DateTimeOffset.UtcNow,
                Option<ReceiptUrl>.None)
        };
        
        var service = new OrderService(processor);
        var order = Order.Create(customerId, items);
        
        var result = await service.ProcessOrderAsync(order);
        
        Assert.True(result.IsSuccess);
        Assert.Equal(OrderStatus.PaymentConfirmed, order.Status);
    }
}
```

## Multiple External Systems

```csharp
// Anti-Corruption Layer for legacy system
public sealed class LegacyCustomerTranslator
{
    private readonly LegacyCustomerService _legacyService;
    
    public async Task<Option<Customer>> GetCustomerAsync(CustomerId id)
    {
        // Legacy uses string IDs and different field names
        var legacyCustomer = await _legacyService.GetCustomerByCode(id.Value.ToString());
        
        if (legacyCustomer == null)
            return Option<Customer>.None;
        
        // Translate legacy model to domain
        var email = EmailAddress.Parse(legacyCustomer.EmailAddr);
        var customer = Customer.Create(
            id,
            legacyCustomer.FullName,
            email,
            TranslateLegacyStatus(legacyCustomer.StatusCode));
        
        return Option<Customer>.Some(customer);
    }
    
    private CustomerStatus TranslateLegacyStatus(int statusCode)
    {
        return statusCode switch
        {
            1 => CustomerStatus.Active,
            2 => CustomerStatus.Suspended,
            3 => CustomerStatus.Closed,
            _ => throw new ArgumentException($"Unknown status code: {statusCode}")
        };
    }
}

// Anti-Corruption Layer for shipping service
public sealed class ShipStationAdapter
{
    private readonly IShipStationClient _client;
    
    public async Task<Result<ShipmentTracking, string>> CreateShipmentAsync(
        ShipmentRequest request)
    {
        // Translate domain to ShipStation API model
        var shipStationOrder = new ShipStationOrderCreate
        {
            OrderNumber = request.OrderId.Value.ToString(),
            OrderDate = request.OrderDate.ToString("yyyy-MM-dd"),
            ShipTo = new ShipStationAddress
            {
                Name = request.ShippingAddress.RecipientName,
                Street1 = request.ShippingAddress.Street,
                City = request.ShippingAddress.City,
                State = request.ShippingAddress.State,
                PostalCode = request.ShippingAddress.PostalCode,
                Country = request.ShippingAddress.CountryCode
            },
            Items = request.Items.Select(item => new ShipStationOrderItem
            {
                Sku = item.Sku.Value,
                Quantity = item.Quantity,
                Name = item.ProductName
            }).ToList()
        };
        
        try
        {
            var response = await _client.CreateOrderAsync(shipStationOrder);
            
            // Translate ShipStation response to domain
            var tracking = new ShipmentTracking(
                new TrackingNumber(response.TrackingNumber),
                new CarrierCode(response.CarrierCode),
                DateTimeOffset.Parse(response.ShipDate));
            
            return Result<ShipmentTracking, string>.Success(tracking);
        }
        catch (ShipStationException ex)
        {
            return Result<ShipmentTracking, string>.Failure(
                $"Shipping service error: {ex.Message}");
        }
    }
}
```

## API Gateway Pattern

```csharp
/// <summary>
/// Facade over multiple external services.
/// Provides unified interface for domain.
/// </summary>
public sealed class PaymentGateway : IPaymentProcessor
{
    private readonly StripePaymentProcessor _stripe;
    private readonly PayPalPaymentProcessor _paypal;
    private readonly IPaymentMethodClassifier _classifier;
    
    public async Task<Result<PaymentConfirmation, PaymentError>> ProcessPaymentAsync(
        PaymentRequest request)
    {
        // Route to appropriate processor based on payment method
        var processor = _classifier.SelectProcessor(request.Method);
        
        return processor switch
        {
            "stripe" => await _stripe.ProcessPaymentAsync(request),
            "paypal" => await _paypal.ProcessPaymentAsync(request),
            _ => Result<PaymentConfirmation, PaymentError>.Failure(
                new PaymentError("UNSUPPORTED_METHOD", "Payment method not supported"))
        };
    }
}
```

## Caching Layer

```csharp
/// <summary>
/// Anti-Corruption Layer with caching to reduce external calls.
/// </summary>
public sealed class CachedProductCatalogAdapter
{
    private readonly IExternalProductApi _externalApi;
    private readonly IMemoryCache _cache;
    
    public async Task<Option<Product>> GetProductAsync(ProductId id)
    {
        var cacheKey = $"product:{id}";
        
        if (_cache.TryGetValue<Product>(cacheKey, out var cached))
            return Option<Product>.Some(cached);
        
        // Call external API
        var externalProduct = await _externalApi.GetProductAsync(id.Value);
        
        if (externalProduct == null)
            return Option<Product>.None;
        
        // Translate and cache
        var product = TranslateToProduct(externalProduct);
        _cache.Set(cacheKey, product, TimeSpan.FromMinutes(15));
        
        return Option<Product>.Some(product);
    }
    
    private Product TranslateToProduct(ExternalProductDto dto)
    {
        // Defensive translation—external data may be invalid
        var price = dto.Price > 0 
            ? Money.USD(dto.Price) 
            : Money.USD(0);  // Default for invalid data
        
        var name = !string.IsNullOrWhiteSpace(dto.Name)
            ? dto.Name
            : "Unknown Product";  // Safe default
        
        return Product.Create(
            new ProductId(Guid.Parse(dto.Id)),
            name,
            price);
    }
}
```

## Event Translation

```csharp
/// <summary>
/// Translates external events to domain events.
/// </summary>
public sealed class PaymentEventTranslator
{
    private readonly IEventDispatcher _dispatcher;
    
    /// <summary>
    /// Stripe webhook handler.
    /// </summary>
    public async Task HandleStripeWebhook(string json, string signature)
    {
        // Verify and parse Stripe event
        var stripeEvent = EventUtility.ConstructEvent(json, signature, _webhookSecret);
        
        // Translate to domain events
        var domainEvent = stripeEvent.Type switch
        {
            "charge.succeeded" => TranslateChargeSucceeded(stripeEvent.Data.Object as Charge),
            "charge.failed" => TranslateChargeFailed(stripeEvent.Data.Object as Charge),
            "charge.refunded" => TranslateChargeRefunded(stripeEvent.Data.Object as Charge),
            _ => null
        };
        
        if (domainEvent != null)
            await _dispatcher.DispatchAsync(domainEvent);
    }
    
    private PaymentSucceededEvent TranslateChargeSucceeded(Charge charge)
    {
        var orderId = OrderId.Parse(charge.Metadata["order_id"]);
        var amount = Money.USD(charge.Amount / 100m);
        
        return new PaymentSucceededEvent(
            orderId,
            new TransactionId(charge.Id),
            amount,
            DateTimeOffset.FromUnixTimeSeconds(charge.Created));
    }
}
```

## Why It's a Problem

1. **Tight coupling**: Domain depends on external systems
2. **Hard to test**: Requires mocking external SDKs
3. **Brittle**: External changes break domain
4. **Polluted model**: Foreign concepts in domain
5. **Vendor lock-in**: Can't switch providers

## Symptoms

- Domain classes with external library types
- `using` statements for third-party SDKs in domain
- Domain tests requiring external system mocks
- Foreign terminology in domain code
- Difficulty switching providers

## Benefits

- **Isolation**: Domain independent of external systems
- **Testability**: Easy to test without external dependencies
- **Flexibility**: Can swap providers without domain changes
- **Clean model**: Domain uses its own vocabulary
- **Resilience**: External failures don't corrupt domain

## See Also

- [Bounded Contexts](./bounded-contexts.md) — context boundaries
- [DTO vs Domain Boundary](./dto-domain-boundary.md) — API separation
- [Domain Events](./domain-events.md) — decoupled communication
- [Repository Pattern](./repository-pattern.md) — data access isolation
