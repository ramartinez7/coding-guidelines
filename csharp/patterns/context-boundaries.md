# Context Boundaries (Explicit Domain Context Separation)

> Mixing concepts from different domains creates confusion—establish explicit context boundaries with translation layers.

## Problem

When concepts from different business domains mix in the same codebase without clear boundaries, the same term can mean different things.

## Example

### ❌ Mixed Contexts

```csharp
// "Product" means different things in different contexts
public class Product
{
    // Catalog context: product information
    public string Name { get; set; }
    public string Description { get; set; }
    
    // Inventory context: stock levels
    public int StockLevel { get; set; }
    public string WarehouseLocation { get; set; }
    
    // Pricing context: pricing rules
    public decimal BasePrice { get; set; }
    public decimal DiscountPercentage { get; set; }
    
    // Shipping context: logistics
    public double Weight { get; set; }
    public string PackagingType { get; set; }
}
```

### ✅ Separate Contexts

```csharp
// Catalog Context: Product as catalog item
namespace Catalog.Domain
{
    public sealed class Product
    {
        public ProductId Id { get; }
        public ProductName Name { get; }
        public Description Description { get; }
        public Category Category { get; }
    }
}

// Inventory Context: Product as inventory item
namespace Inventory.Domain
{
    public sealed class InventoryItem
    {
        public SKU Sku { get; }  // Different identity!
        public int QuantityOnHand { get; private set; }
        public WarehouseLocation Location { get; }
        
        public Result<Unit, string> AdjustQuantity(int delta)
        {
            if (QuantityOnHand + delta < 0)
                return Result<Unit, string>.Failure("Cannot have negative inventory");
            
            QuantityOnHand += delta;
            return Result<Unit, string>.Success(Unit.Value);
        }
    }
}

// Pricing Context: Product as priced item
namespace Pricing.Domain
{
    public sealed class PricedProduct
    {
        public ProductId ProductId { get; }
        public Money BasePrice { get; private set; }
        public Option<DiscountRule> ActiveDiscount { get; private set; }
        
        public Money CalculatePrice(CustomerTier tier) =>
            ActiveDiscount.Match(
                onSome: discount => discount.Apply(BasePrice, tier),
                onNone: () => BasePrice);
    }
}

// Context integration through translation
namespace Catalog.Integration
{
    public sealed class InventoryAdapter
    {
        private readonly IInventoryService _inventory;
        
        public async Task<int> GetStockLevel(ProductId catalogProductId)
        {
            // Translate catalog ID to inventory SKU
            var sku = MapProductIdToSKU(catalogProductId);
            
            var item = await _inventory.GetItemAsync(sku);
            return item.QuantityOnHand;
        }
        
        private SKU MapProductIdToSKU(ProductId productId)
        {
            // Translation logic between contexts
            return SKU.Parse(productId.Value.ToString());
        }
    }
}
```

## See Also

- [Bounded Contexts](./bounded-contexts.md)
- [Context Mapping](./context-mapping.md)
- [Anti-Corruption Layer](./anti-corruption-layer.md)
