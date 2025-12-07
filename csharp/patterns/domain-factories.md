# Domain Factories (Complex Object Creation)

> Complex object creation with invariants and multiple dependencies—use factories to encapsulate construction logic and ensure valid aggregates.

## Problem

When creating complex domain objects requires multiple steps, validation, or coordination of related objects, constructors become unwieldy. Business rules around object creation get scattered or bypassed.

## Example

### ❌ Before

```csharp
// Complex construction with scattered validation
public class Order
{
    public Order(Guid id, Guid customerId, string shippingStreet, string shippingCity)
    {
        Id = id;
        CustomerId = customerId;
        ShippingStreet = shippingStreet;
        ShippingCity = shippingCity;
        Items = new List<OrderItem>();
    }
    
    public Guid Id { get; set; }
    public Guid CustomerId { get; set; }
    public string ShippingStreet { get; set; }
    public string ShippingCity { get; set; }
    public List<OrderItem> Items { get; set; }
    public OrderStatus Status { get; set; }
}

// Client code: Complex construction with many steps
var order = new Order(
    Guid.NewGuid(),
    customerId,
    "123 Main St",
    "Boston");

// Validation scattered across codebase
if (string.IsNullOrEmpty(order.ShippingStreet))
    throw new Exception("Street required");

// Adding items with separate validation
var item1 = new OrderItem { ProductId = product1.Id, Quantity = 2, Price = 10.00m };
if (item1.Quantity <= 0)
    throw new Exception("Invalid quantity");
order.Items.Add(item1);

// Status initialization forgotten
// order.Status = OrderStatus.Draft;  // Oops, forgot to set!

// Business rules not enforced during construction
```

**Problems:**
- Construction logic scattered
- Validation can be bypassed
- Easy to create invalid objects
- Business rules not enforced
- No single place for creation logic

### ✅ After

```csharp
// ============================================
// DOMAIN ENTITIES (With private constructors)
// ============================================
namespace Orders.Domain
{
    /// <summary>
    /// Aggregate root with private constructor.
    /// Can only be created through factory.
    /// </summary>
    public sealed class Order
    {
        private readonly List<OrderLine> _lines = new();
        
        public OrderId Id { get; }
        public CustomerId CustomerId { get; }
        public Address ShippingAddress { get; }
        public OrderStatus Status { get; private set; }
        public DateTimeOffset CreatedAt { get; }
        public IReadOnlyList<OrderLine> Lines => _lines.AsReadOnly();
        
        // Private constructor—can only create through factory
        private Order(
            OrderId id,
            CustomerId customerId,
            Address shippingAddress,
            DateTimeOffset createdAt,
            List<OrderLine> initialLines)
        {
            Id = id;
            CustomerId = customerId;
            ShippingAddress = shippingAddress;
            CreatedAt = createdAt;
            Status = OrderStatus.Draft;
            _lines = initialLines;
        }
        
        // Factory method for simple cases
        public static Result<Order, string> Create(
            CustomerId customerId,
            Address shippingAddress)
        {
            if (shippingAddress == null)
                return Result<Order, string>.Failure("Shipping address required");
            
            var order = new Order(
                OrderId.New(),
                customerId,
                shippingAddress,
                DateTimeOffset.UtcNow,
                new List<OrderLine>());
            
            return Result<Order, string>.Success(order);
        }
        
        // For complex creation, use OrderFactory
        internal static Order CreateFromFactory(
            OrderId id,
            CustomerId customerId,
            Address shippingAddress,
            DateTimeOffset createdAt,
            List<OrderLine> initialLines)
        {
            return new Order(id, customerId, shippingAddress, createdAt, initialLines);
        }
        
        public void AddLine(OrderLine line)
        {
            _lines.Add(line);
        }
    }
    
    public sealed record OrderLine(ProductId ProductId, int Quantity, Money UnitPrice)
    {
        public Money LineTotal => UnitPrice * Quantity;
    }
}

// ============================================
// DOMAIN FACTORY (Complex creation logic)
// ============================================
namespace Orders.Domain.Factories
{
    /// <summary>
    /// Factory: Encapsulates complex order creation with validation and business rules.
    /// </summary>
    public sealed class OrderFactory
    {
        private readonly ICustomerRepository _customerRepo;
        private readonly IProductRepository _productRepo;
        private readonly IShippingValidator _shippingValidator;
        
        public OrderFactory(
            ICustomerRepository customerRepo,
            IProductRepository productRepo,
            IShippingValidator shippingValidator)
        {
            _customerRepo = customerRepo;
            _productRepo = productRepo;
            _shippingValidator = shippingValidator;
        }
        
        /// <summary>
        /// Creates a new order with full validation and business rules.
        /// </summary>
        public async Task<Result<Order, OrderCreationError>> CreateOrder(
            CreateOrderRequest request)
        {
            // Validate customer exists and is active
            var customerOpt = await _customerRepo.GetByIdAsync(request.CustomerId);
            if (customerOpt.IsNone)
                return Result<Order, OrderCreationError>.Failure(
                    OrderCreationError.CustomerNotFound(request.CustomerId));
            
            var customer = customerOpt.Value;
            if (!customer.IsActive)
                return Result<Order, OrderCreationError>.Failure(
                    OrderCreationError.CustomerNotActive(request.CustomerId));
            
            // Validate shipping address
            var addressValidation = _shippingValidator.Validate(
                request.ShippingAddress,
                customer.Country);
            
            if (!addressValidation.IsSuccess)
                return Result<Order, OrderCreationError>.Failure(
                    OrderCreationError.InvalidAddress(addressValidation.Error));
            
            // Validate and load products
            var linesResult = await ValidateAndCreateLines(request.Items);
            if (!linesResult.IsSuccess)
                return Result<Order, OrderCreationError>.Failure(linesResult.Error);
            
            var lines = linesResult.Value;
            
            // Business rule: minimum order value
            var subtotal = lines.Sum(line => line.LineTotal.Amount);
            if (subtotal < 10m)
                return Result<Order, OrderCreationError>.Failure(
                    OrderCreationError.BelowMinimumOrderValue(Money.USD(subtotal)));
            
            // Create order through internal factory method
            var order = Order.CreateFromFactory(
                OrderId.New(),
                request.CustomerId,
                request.ShippingAddress,
                DateTimeOffset.UtcNow,
                lines);
            
            return Result<Order, OrderCreationError>.Success(order);
        }
        
        private async Task<Result<List<OrderLine>, OrderCreationError>> ValidateAndCreateLines(
            List<OrderItemRequest> items)
        {
            if (items.Count == 0)
                return Result<List<OrderLine>, OrderCreationError>.Failure(
                    OrderCreationError.EmptyOrder());
            
            var lines = new List<OrderLine>();
            
            foreach (var item in items)
            {
                // Validate quantity
                if (item.Quantity <= 0)
                    return Result<List<OrderLine>, OrderCreationError>.Failure(
                        OrderCreationError.InvalidQuantity(item.ProductId, item.Quantity));
                
                // Validate product exists
                var productOpt = await _productRepo.GetByIdAsync(item.ProductId);
                if (productOpt.IsNone)
                    return Result<List<OrderLine>, OrderCreationError>.Failure(
                        OrderCreationError.ProductNotFound(item.ProductId));
                
                var product = productOpt.Value;
                
                // Business rule: product must be active
                if (!product.IsActive)
                    return Result<List<OrderLine>, OrderCreationError>.Failure(
                        OrderCreationError.ProductNotAvailable(item.ProductId));
                
                // Create order line with validated data
                var line = new OrderLine(
                    product.Id,
                    item.Quantity,
                    product.CurrentPrice);
                
                lines.Add(line);
            }
            
            return Result<List<OrderLine>, OrderCreationError>.Success(lines);
        }
    }
    
    /// <summary>
    /// Request for creating an order.
    /// </summary>
    public sealed record CreateOrderRequest(
        CustomerId CustomerId,
        Address ShippingAddress,
        List<OrderItemRequest> Items);
    
    public sealed record OrderItemRequest(ProductId ProductId, int Quantity);
    
    /// <summary>
    /// Domain-specific errors for order creation.
    /// </summary>
    public abstract record OrderCreationError(string Message)
    {
        public sealed record CustomerNotFoundError(CustomerId CustomerId)
            : OrderCreationError($"Customer {CustomerId} not found");
        
        public sealed record CustomerNotActiveError(CustomerId CustomerId)
            : OrderCreationError($"Customer {CustomerId} is not active");
        
        public sealed record InvalidAddressError(string Reason)
            : OrderCreationError($"Invalid address: {Reason}");
        
        public sealed record ProductNotFoundError(ProductId ProductId)
            : OrderCreationError($"Product {ProductId} not found");
        
        public sealed record ProductNotAvailableError(ProductId ProductId)
            : OrderCreationError($"Product {ProductId} is not available");
        
        public sealed record InvalidQuantityError(ProductId ProductId, int Quantity)
            : OrderCreationError($"Invalid quantity {Quantity} for product {ProductId}");
        
        public sealed record BelowMinimumError(Money Amount)
            : OrderCreationError($"Order total {Amount} is below minimum");
        
        public sealed record EmptyOrderError()
            : OrderCreationError("Order must contain at least one item");
        
        public static OrderCreationError CustomerNotFound(CustomerId id)
            => new CustomerNotFoundError(id);
        
        public static OrderCreationError CustomerNotActive(CustomerId id)
            => new CustomerNotActiveError(id);
        
        public static OrderCreationError InvalidAddress(string reason)
            => new InvalidAddressError(reason);
        
        public static OrderCreationError ProductNotFound(ProductId id)
            => new ProductNotFoundError(id);
        
        public static OrderCreationError ProductNotAvailable(ProductId id)
            => new ProductNotAvailableError(id);
        
        public static OrderCreationError InvalidQuantity(ProductId id, int quantity)
            => new InvalidQuantityError(id, quantity);
        
        public static OrderCreationError BelowMinimumOrderValue(Money amount)
            => new BelowMinimumError(amount);
        
        public static OrderCreationError EmptyOrder()
            => new EmptyOrderError();
    }
}
```

## Reconstitution Factory

```csharp
/// <summary>
/// Factory for reconstituting entities from persistence.
/// Bypasses validation since data is already validated.
/// </summary>
public sealed class OrderReconstitutionFactory
{
    /// <summary>
    /// Reconstitutes an order from database data.
    /// No validation—assumes data is already valid.
    /// </summary>
    public Order Reconstitute(OrderData data)
    {
        var lines = data.Lines
            .Select(lineData => new OrderLine(
                new ProductId(lineData.ProductId),
                lineData.Quantity,
                new Money(lineData.UnitPrice, lineData.Currency)))
            .ToList();
        
        return Order.CreateFromFactory(
            new OrderId(data.Id),
            new CustomerId(data.CustomerId),
            new Address(data.ShippingStreet, data.ShippingCity, data.ShippingZip, data.ShippingCountry),
            data.CreatedAt,
            lines);
    }
}

// Data transfer object from database
public record OrderData(
    Guid Id,
    Guid CustomerId,
    string ShippingStreet,
    string ShippingCity,
    string ShippingZip,
    string ShippingCountry,
    DateTimeOffset CreatedAt,
    List<OrderLineData> Lines);

public record OrderLineData(
    Guid ProductId,
    int Quantity,
    decimal UnitPrice,
    Currency Currency);
```

## Abstract Factory Pattern

```csharp
/// <summary>
/// Abstract factory for creating different types of accounts.
/// </summary>
public interface IAccountFactory
{
    Task<Result<Account, string>> CreateAccount(AccountType type, CustomerId customerId);
}

public sealed class BankAccountFactory : IAccountFactory
{
    private readonly ICustomerRepository _customerRepo;
    private readonly ICreditCheckService _creditCheck;
    
    public async Task<Result<Account, string>> CreateAccount(
        AccountType type,
        CustomerId customerId)
    {
        var customerOpt = await _customerRepo.GetByIdAsync(customerId);
        if (customerOpt.IsNone)
            return Result<Account, string>.Failure("Customer not found");
        
        var customer = customerOpt.Value;
        
        return type switch
        {
            AccountType.Checking => CreateCheckingAccount(customer),
            AccountType.Savings => CreateSavingsAccount(customer),
            AccountType.Credit => await CreateCreditAccount(customer),
            _ => Result<Account, string>.Failure($"Unknown account type: {type}")
        };
    }
    
    private Result<Account, string> CreateCheckingAccount(Customer customer)
    {
        var account = CheckingAccount.Create(
            AccountId.New(),
            customer.Id,
            Money.Zero,
            overdraftLimit: Money.USD(100));
        
        return Result<Account, string>.Success(account);
    }
    
    private Result<Account, string> CreateSavingsAccount(Customer customer)
    {
        var account = SavingsAccount.Create(
            AccountId.New(),
            customer.Id,
            Money.Zero,
            interestRate: 0.02m);
        
        return Result<Account, string>.Success(account);
    }
    
    private async Task<Result<Account, string>> CreateCreditAccount(Customer customer)
    {
        // Credit account requires additional validation
        var creditCheck = await _creditCheck.CheckCredit(customer);
        
        if (!creditCheck.IsApproved)
            return Result<Account, string>.Failure("Credit check failed");
        
        var account = CreditAccount.Create(
            AccountId.New(),
            customer.Id,
            Money.Zero,
            creditLimit: Money.USD(creditCheck.ApprovedLimit));
        
        return Result<Account, string>.Success(account);
    }
}
```

## Builder-Style Factory

```csharp
/// <summary>
/// Fluent factory for building complex aggregates step-by-step.
/// </summary>
public sealed class OrderBuilder
{
    private CustomerId? _customerId;
    private Address? _shippingAddress;
    private readonly List<OrderLine> _lines = new();
    private Money? _discount;
    
    public OrderBuilder ForCustomer(CustomerId customerId)
    {
        _customerId = customerId;
        return this;
    }
    
    public OrderBuilder ShipTo(Address address)
    {
        _shippingAddress = address;
        return this;
    }
    
    public OrderBuilder AddLine(ProductId productId, int quantity, Money unitPrice)
    {
        _lines.Add(new OrderLine(productId, quantity, unitPrice));
        return this;
    }
    
    public OrderBuilder WithDiscount(Money discount)
    {
        _discount = discount;
        return this;
    }
    
    public Result<Order, string> Build()
    {
        // Validate all required fields are set
        if (_customerId == null)
            return Result<Order, string>.Failure("Customer is required");
        
        if (_shippingAddress == null)
            return Result<Order, string>.Failure("Shipping address is required");
        
        if (_lines.Count == 0)
            return Result<Order, string>.Failure("Order must have at least one line");
        
        // Create order
        var order = Order.CreateFromFactory(
            OrderId.New(),
            _customerId,
            _shippingAddress,
            DateTimeOffset.UtcNow,
            _lines);
        
        return Result<Order, string>.Success(order);
    }
}

// Usage
var orderResult = new OrderBuilder()
    .ForCustomer(customerId)
    .ShipTo(shippingAddress)
    .AddLine(product1, 2, Money.USD(10))
    .AddLine(product2, 1, Money.USD(25))
    .Build();
```

## Factory for Aggregate Hierarchies

```csharp
/// <summary>
/// Factory for creating aggregate with complex child objects.
/// </summary>
public sealed class SubscriptionFactory
{
    private readonly IProductRepository _productRepo;
    private readonly IPricingService _pricingService;
    
    public async Task<Result<Subscription, string>> CreateSubscription(
        CustomerId customerId,
        SubscriptionPlanId planId,
        BillingCycle cycle,
        List<AddOnRequest> addOns)
    {
        // Load and validate subscription plan
        var planOpt = await _productRepo.GetPlanAsync(planId);
        if (planOpt.IsNone)
            return Result<Subscription, string>.Failure("Plan not found");
        
        var plan = planOpt.Value;
        
        // Calculate pricing based on plan and cycle
        var basePrice = await _pricingService.CalculatePrice(plan, cycle);
        
        // Process add-ons
        var addOnFeatures = new List<SubscriptionFeature>();
        foreach (var addOnRequest in addOns)
        {
            var addOnOpt = await _productRepo.GetAddOnAsync(addOnRequest.AddOnId);
            if (addOnOpt.IsNone)
                return Result<Subscription, string>.Failure(
                    $"Add-on {addOnRequest.AddOnId} not found");
            
            var addOn = addOnOpt.Value;
            var feature = new SubscriptionFeature(
                addOn.Id,
                addOn.Name,
                addOn.MonthlyPrice);
            
            addOnFeatures.Add(feature);
        }
        
        // Create subscription aggregate
        var subscription = Subscription.CreateWithFeatures(
            SubscriptionId.New(),
            customerId,
            plan,
            cycle,
            basePrice,
            addOnFeatures,
            DateTimeOffset.UtcNow);
        
        return Result<Subscription, string>.Success(subscription);
    }
}
```

## Why It's a Problem

1. **Complex constructors**: Too many parameters, hard to use
2. **Scattered validation**: Business rules not enforced
3. **Invalid objects**: Easy to bypass validation
4. **No reuse**: Creation logic duplicated

## Symptoms

- Constructors with 5+ parameters
- Public setters to "finish" construction
- Validation scattered across codebase
- Comments explaining construction order
- Helper methods named `Initialize()` or `Setup()`

## Benefits

- **Encapsulated creation**: All construction logic in one place
- **Enforced invariants**: Valid objects guaranteed
- **Reusable**: Complex creation logic centralized
- **Testable**: Factories easy to test

## See Also

- [Static Factory Methods](./static-factory-methods.md) — simple creation patterns
- [Type-Safe Step Builder](./type-safe-builder.md) — compile-time construction validation
- [Aggregate Roots](./aggregate-roots.md) — consistency boundaries
- [Honest Functions](./honest-functions.md) — explicit error handling
