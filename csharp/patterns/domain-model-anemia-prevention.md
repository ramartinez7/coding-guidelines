# Domain Model Anemia Prevention (Avoiding Anemic Domain Models)

> Entities without behavior push logic into services—add behavior to entities to prevent anemic domain models.

## Problem

Domain models with only getters/setters and no behavior result in business logic scattered across service classes.

## Example

### ❌ Anemic Model

```csharp
public class Product
{
    public Guid Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
    public int Stock { get; set; }
}

public class ProductService
{
    public bool ReserveStock(Product product, int quantity)
    {
        if (product.Stock < quantity) return false;
        product.Stock -= quantity;
        return true;
    }
}
```

### ✅ Rich Model

```csharp
public sealed class Product
{
    public ProductId Id { get; }
    public ProductName Name { get; }
    public Money Price { get; private set; }
    private int _stock;
    
    public Result<Unit, string> ReserveStock(int quantity)
    {
        if (quantity <= 0)
            return Result<Unit, string>.Failure("Quantity must be positive");
        
        if (_stock < quantity)
            return Result<Unit, string>.Failure($"Insufficient stock. Available: {_stock}");
        
        _stock -= quantity;
        return Result<Unit, string>.Success(Unit.Value);
    }
}
```

## See Also

- [Rich Domain Models](./rich-domain-models.md)
- [Tell, Don't Ask](./tell-dont-ask.md)
