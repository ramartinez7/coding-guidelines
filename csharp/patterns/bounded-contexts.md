# Bounded Contexts (Strategic Domain Boundaries)

> Large models with conflicting meanings for the same term—divide the system into bounded contexts where each term has one precise definition.

## Problem

In large systems, the same word can mean different things in different parts of the business. A "Product" in the catalog is different from a "Product" in the shopping cart or warehouse. Trying to use one model everywhere creates confusion, coupling, and compromises.

## Example

### ❌ Before

```csharp
// One "Product" class trying to serve everyone
public class Product
{
    // Catalog context needs
    public Guid Id { get; set; }
    public string Name { get; set; }
    public string Description { get; set; }
    public List<string> Images { get; set; }
    public decimal Price { get; set; }
    public int ReviewCount { get; set; }
    public double AverageRating { get; set; }
    
    // Shopping cart context needs
    public bool IsInStock { get; set; }
    public int SelectedQuantity { get; set; }
    
    // Warehouse context needs
    public string WarehouseLocation { get; set; }
    public int QuantityOnHand { get; set; }
    public int QuantityReserved { get; set; }
    public DateTime? LastRestockedAt { get; set; }
    
    // Pricing context needs
    public decimal CostPrice { get; set; }
    public decimal SuggestedRetailPrice { get; set; }
    public List<PriceHistory> PriceHistory { get; set; }
    
    // God object with responsibilities from multiple contexts
}

// Services become tightly coupled
public class ProductService
{
    // Handles concerns from all contexts
    public void UpdateProductPrice(Guid productId, decimal newPrice) { }
    public void AddToCart(Guid productId, int quantity) { }
    public void ReserveStock(Guid productId, int quantity) { }
    public void UpdateWarehouseLocation(Guid productId, string location) { }
}
```

**Problems:**
- One Product class with conflicting responsibilities
- Changes in one context affect all others
- Different teams step on each other's toes
- Impossible to optimize for any single use case
- Unclear which properties are relevant where

### ✅ After

```csharp
// ============================================
// CATALOG CONTEXT
// Concern: Browsing and discovering products
// ============================================
namespace Catalog
{
    /// <summary>
    /// Product in the catalog context: focused on customer discovery and browsing.
    /// </summary>
    public sealed class Product
    {
        public ProductId Id { get; }
        public string Name { get; private set; }
        public RichText Description { get; private set; }
        public List<ImageUrl> Images { get; private set; }
        public Money DisplayPrice { get; private set; }
        public ProductReviews Reviews { get; private set; }
        
        public void UpdateDescription(RichText description)
        {
            Description = description;
        }
        
        public void SetPrice(Money price)
        {
            DisplayPrice = price;
        }
    }
    
    public sealed record ProductReviews(int Count, double AverageRating);
}

// ============================================
// SHOPPING CART CONTEXT
// Concern: Selecting items for purchase
// ============================================
namespace ShoppingCart
{
    /// <summary>
    /// Product in shopping cart context: lightweight, focused on selection.
    /// Only what's needed for cart operations.
    /// </summary>
    public sealed record CartProduct(
        ProductId Id,
        string Name,
        Money UnitPrice,
        StockStatus Availability);
    
    public sealed class Cart
    {
        private readonly List<CartItem> _items = new();
        
        public CartId Id { get; }
        public CustomerId CustomerId { get; }
        public IReadOnlyList<CartItem> Items => _items.AsReadOnly();
        
        public Result<Unit, string> AddItem(CartProduct product, int quantity)
        {
            if (!product.Availability.IsAvailable)
                return Result<Unit, string>.Failure("Product not available");
            
            var item = new CartItem(product.Id, product.Name, quantity, product.UnitPrice);
            _items.Add(item);
            
            return Result<Unit, string>.Success(Unit.Value);
        }
    }
    
    public sealed record CartItem(
        ProductId ProductId,
        string Name,
        int Quantity,
        Money UnitPrice);
}

// ============================================
// WAREHOUSE CONTEXT
// Concern: Physical inventory management
// ============================================
namespace Warehouse
{
    /// <summary>
    /// Product in warehouse context: focused on physical location and stock levels.
    /// Different model, different concerns.
    /// </summary>
    public sealed class InventoryItem
    {
        public Sku Sku { get; }  // Note: Uses SKU, not ProductId
        public WarehouseLocation Location { get; private set; }
        public int QuantityOnHand { get; private set; }
        public int QuantityReserved { get; private set; }
        public DateTimeOffset? LastRestockedAt { get; private set; }
        
        public int AvailableQuantity => QuantityOnHand - QuantityReserved;
        
        public Result<Reservation, string> ReserveStock(int quantity)
        {
            if (quantity <= 0)
                return Result<Reservation, string>.Failure("Quantity must be positive");
            
            if (AvailableQuantity < quantity)
                return Result<Reservation, string>.Failure("Insufficient stock");
            
            QuantityReserved += quantity;
            
            return Result<Reservation, string>.Success(
                new Reservation(ReservationId.New(), Sku, quantity));
        }
        
        public void Restock(int quantity, DateTimeOffset restockedAt)
        {
            QuantityOnHand += quantity;
            LastRestockedAt = restockedAt;
        }
    }
    
    public sealed record Reservation(ReservationId Id, Sku Sku, int Quantity);
}

// ============================================
// PRICING CONTEXT
// Concern: Price determination and history
// ============================================
namespace Pricing
{
    /// <summary>
    /// Product in pricing context: focused on cost analysis and price strategy.
    /// </summary>
    public sealed class PricedProduct
    {
        private readonly List<PriceChangeEvent> _priceHistory = new();
        
        public ProductId Id { get; }
        public Money CostPrice { get; private set; }
        public Money CurrentRetailPrice { get; private set; }
        public Money SuggestedRetailPrice { get; private set; }
        public IReadOnlyList<PriceChangeEvent> PriceHistory => _priceHistory.AsReadOnly();
        
        public decimal ProfitMargin => 
            (CurrentRetailPrice.Amount - CostPrice.Amount) / CostPrice.Amount;
        
        public Result<Unit, string> ChangePrice(Money newPrice, string reason)
        {
            if (newPrice.Amount < CostPrice.Amount)
                return Result<Unit, string>.Failure("Price cannot be below cost");
            
            var priceChange = new PriceChangeEvent(
                ChangedAt: DateTimeOffset.UtcNow,
                OldPrice: CurrentRetailPrice,
                NewPrice: newPrice,
                Reason: reason);
            
            _priceHistory.Add(priceChange);
            CurrentRetailPrice = newPrice;
            
            return Result<Unit, string>.Success(Unit.Value);
        }
    }
    
    public sealed record PriceChangeEvent(
        DateTimeOffset ChangedAt,
        Money OldPrice,
        Money NewPrice,
        string Reason);
}
```

## Context Mapping

```csharp
// Translation between contexts using explicit mappers
namespace Catalog
{
    public interface IProductRepository
    {
        Task<Option<Product>> GetByIdAsync(ProductId id);
    }
}

namespace ShoppingCart
{
    /// <summary>
    /// Anti-Corruption Layer: Translates from Catalog to ShoppingCart context.
    /// </summary>
    public class CartProductTranslator
    {
        private readonly Catalog.IProductRepository _catalogRepository;
        private readonly Warehouse.IInventoryQuery _inventoryQuery;
        
        public async Task<Option<CartProduct>> TranslateAsync(ProductId productId)
        {
            var catalogProductOpt = await _catalogRepository.GetByIdAsync(productId);
            
            return await catalogProductOpt.MatchAsync(
                onSome: async catalogProduct =>
                {
                    // Check availability from warehouse context
                    var availability = await _inventoryQuery.CheckAvailability(
                        Sku.FromProductId(productId));
                    
                    // Create cart-specific model
                    var cartProduct = new CartProduct(
                        catalogProduct.Id,
                        catalogProduct.Name,
                        catalogProduct.DisplayPrice,
                        availability);
                    
                    return Option<CartProduct>.Some(cartProduct);
                },
                onNone: () => Task.FromResult(Option<CartProduct>.None));
        }
    }
}
```

## Context Integration Patterns

### Shared Kernel

```csharp
// Core assembly shared by multiple contexts
namespace Core
{
    // Shared value objects used across contexts
    public sealed record ProductId(Guid Value)
    {
        public static ProductId New() => new(Guid.NewGuid());
    }
    
    public sealed record Money(decimal Amount, Currency Currency);
    
    public enum Currency { USD, EUR, GBP }
}
```

### Customer-Supplier

```csharp
// Catalog context is upstream supplier
namespace Catalog
{
    public interface IProductPublisher
    {
        Task PublishProductCreatedAsync(ProductCreatedEvent evt);
        Task PublishPriceChangedAsync(PriceChangedEvent evt);
    }
}

// Pricing context is downstream customer
namespace Pricing
{
    /// <summary>
    /// Subscribes to events from Catalog context.
    /// </summary>
    public class ProductEventHandler
    {
        public async Task HandleProductCreated(ProductCreatedEvent evt)
        {
            // Create corresponding pricing record
            var pricedProduct = new PricedProduct(
                evt.ProductId,
                evt.InitialCostPrice,
                evt.InitialRetailPrice);
            
            await _repository.SaveAsync(pricedProduct);
        }
    }
}
```

### Published Language

```csharp
// Shared event contracts between contexts
namespace Events
{
    public sealed record ProductCreatedEvent(
        ProductId ProductId,
        string Name,
        Money InitialCostPrice,
        Money InitialRetailPrice,
        DateTimeOffset CreatedAt) : IDomainEvent;
    
    public sealed record OrderPlacedEvent(
        OrderId OrderId,
        CustomerId CustomerId,
        List<OrderLine> Items,
        DateTimeOffset PlacedAt) : IDomainEvent;
}
```

## Assembly Organization

```
Solution Structure:
├── Core/                          (Shared Kernel)
│   ├── ProductId.cs
│   ├── Money.cs
│   └── Result.cs
├── Catalog/
│   ├── Catalog.Domain/           (Catalog bounded context)
│   │   ├── Product.cs
│   │   └── Category.cs
│   ├── Catalog.Infrastructure/
│   │   └── ProductRepository.cs
│   └── Catalog.Api/
│       └── ProductsController.cs
├── ShoppingCart/
│   ├── ShoppingCart.Domain/      (Cart bounded context)
│   │   ├── Cart.cs
│   │   └── CartItem.cs
│   ├── ShoppingCart.Infrastructure/
│   │   └── CartRepository.cs
│   └── ShoppingCart.Api/
│       └── CartsController.cs
├── Warehouse/
│   ├── Warehouse.Domain/         (Warehouse bounded context)
│   │   ├── InventoryItem.cs
│   │   └── Reservation.cs
│   └── Warehouse.Infrastructure/
│       └── InventoryRepository.cs
└── Pricing/
    ├── Pricing.Domain/           (Pricing bounded context)
    │   ├── PricedProduct.cs
    │   └── PriceHistory.cs
    └── Pricing.Infrastructure/
        └── PricingRepository.cs
```

## Benefits of Bounded Contexts

```csharp
// Before: One team's change breaks another team
public class Product
{
    public decimal Price { get; set; }  // Which price? Display? Cost? Discount?
}

// After: Each context owns its model
namespace Catalog
{
    public class Product
    {
        public Money DisplayPrice { get; set; }  // Clear: customer-facing price
    }
}

namespace Pricing
{
    public class PricedProduct
    {
        public Money CostPrice { get; set; }     // Clear: internal cost
        public Money RetailPrice { get; set; }   // Clear: standard price
    }
}
```

## Context Communication

```csharp
// Synchronous: Direct API call between contexts
public class OrderService
{
    private readonly Catalog.IProductRepository _catalogRepo;
    private readonly Warehouse.IInventoryService _inventoryService;
    
    public async Task<Result<Order, string>> PlaceOrder(PlaceOrderCommand command)
    {
        // Query catalog context for product info
        var productOpt = await _catalogRepo.GetByIdAsync(command.ProductId);
        
        // Command warehouse context to reserve stock
        var reservation = await _inventoryService.ReserveStock(
            Sku.FromProductId(command.ProductId),
            command.Quantity);
        
        return reservation.Match(
            onSuccess: reserved => CreateOrder(command, reserved),
            onFailure: error => Result<Order, string>.Failure(error));
    }
}

// Asynchronous: Event-driven communication
public class OrderEventHandler
{
    public async Task Handle(OrderPlacedEvent evt)
    {
        // Warehouse context reacts to event from Orders context
        foreach (var line in evt.Items)
        {
            await _inventoryService.FulfillReservation(
                Sku.FromProductId(line.ProductId),
                line.Quantity);
        }
    }
}
```

## Why It's a Problem

1. **Conflicting meanings**: Same term means different things
2. **God objects**: One model trying to serve everyone
3. **Tight coupling**: Changes ripple across unrelated features
4. **Team conflicts**: Multiple teams modifying same code
5. **Compromised models**: No model optimized for its purpose

## Symptoms

- Class with 20+ properties serving multiple use cases
- Confusion about which properties are relevant where
- Multiple teams waiting on each other for changes
- Difficulty understanding which part of code handles what
- Comments like "only used by X team"

## Benefits

- **Clear boundaries**: Each context has precise, unambiguous terms
- **Team autonomy**: Teams own their context without blocking others
- **Optimized models**: Each model tailored for its specific purpose
- **Reduced coupling**: Changes in one context don't affect others
- **Parallel development**: Teams work independently

## See Also

- [Anti-Corruption Layer](./anti-corruption-layer.md) — protecting context boundaries
- [Ubiquitous Language](./ubiquitous-language.md) — language within a context
- [Domain Events](./domain-events.md) — communication between contexts
- [DTO vs Domain Boundary](./dto-domain-boundary.md) — separating API from domain
