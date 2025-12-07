# CQRS (Command Query Responsibility Segregation)

> Separate read and write models to optimize each independently—commands change state, queries return data, never both.

## Problem

Using the same model for reads and writes creates conflicting requirements. Writes need strong consistency and validation. Reads need fast queries with denormalized data. Trying to serve both use cases with one model leads to complex, slow code.

## Command-Query Separation

```
Traditional Model                    CQRS
      
┌──────────────┐                   ┌──────────────┐
│    Single    │                   │   Command    │
│    Model     │                   │    Model     │
│              │                   │ (Optimized   │
│ - Create     │                   │  for writes) │
│ - Update     │                   └──────┬───────┘
│ - Delete     │                          │
│ - Query      │                          │ Events
│ - Report     │                          │
│              │                          ▼
│ Conflicting  │                   ┌──────────────┐
│ requirements │                   │   Query      │
└──────────────┘                   │   Model      │
                                   │ (Optimized   │
                                   │  for reads)  │
                                   └──────────────┘
```

## Example

### ❌ Single Model for Reads and Writes

```csharp
public class Product
{
    public Guid Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
    public int StockQuantity { get; set; }
    public string Category { get; set; }
    public string Description { get; set; }
    
    // Navigation properties needed for queries
    public List<Review> Reviews { get; set; }
    public List<OrderItem> OrderItems { get; set; }
    
    // Computed properties for queries
    public decimal AverageRating => Reviews.Average(r => r.Rating);
    public int TotalSales => OrderItems.Sum(o => o.Quantity);
}

public class ProductService
{
    private readonly DbContext _context;
    
    public async Task UpdateStock(Guid productId, int quantity)
    {
        // Write operation loads unnecessary data
        var product = await _context.Products
            .Include(p => p.Reviews)        // Not needed for write!
            .Include(p => p.OrderItems)     // Not needed for write!
            .FirstAsync(p => p.Id == productId);
        
        product.StockQuantity = quantity;
        await _context.SaveChangesAsync();
    }
    
    public async Task<ProductDto> GetProduct(Guid productId)
    {
        // Read operation does expensive joins
        var product = await _context.Products
            .Include(p => p.Reviews)
            .Include(p => p.OrderItems)
            .FirstAsync(p => p.Id == productId);
        
        // Calculate on every query (slow!)
        return new ProductDto
        {
            Id = product.Id,
            Name = product.Name,
            Price = product.Price,
            AverageRating = product.AverageRating,
            TotalSales = product.TotalSales
        };
    }
}
```

### ✅ CQRS: Separate Read and Write Models

```csharp
// ============================================
// COMMAND SIDE (Writes)
// ============================================

/// <summary>
/// Commands represent intent to change state.
/// </summary>
public abstract record ProductCommand
{
    public sealed record CreateProduct(
        string Name,
        decimal Price,
        int InitialStock,
        string Category,
        string Description) : ProductCommand;
    
    public sealed record UpdateStock(
        Guid ProductId,
        int Quantity) : ProductCommand;
    
    public sealed record UpdatePrice(
        Guid ProductId,
        decimal NewPrice) : ProductCommand;
}

/// <summary>
/// Write model optimized for validation and business logic.
/// </summary>
public class Product
{
    public Guid Id { get; private set; }
    public string Name { get; private set; }
    public decimal Price { get; private set; }
    public int StockQuantity { get; private set; }
    public string Category { get; private set; }
    public string Description { get; private set; }
    
    private readonly List<IDomainEvent> _events = new();
    
    public static Product Create(
        string name,
        decimal price,
        int initialStock,
        string category,
        string description)
    {
        // Validation
        if (string.IsNullOrWhiteSpace(name))
            throw new ArgumentException("Name is required");
        
        if (price <= 0)
            throw new ArgumentException("Price must be positive");
        
        var product = new Product
        {
            Id = Guid.NewGuid(),
            Name = name,
            Price = price,
            StockQuantity = initialStock,
            Category = category,
            Description = description
        };
        
        product.AddEvent(new ProductCreatedEvent(product.Id, name, price));
        
        return product;
    }
    
    public void UpdateStock(int quantity)
    {
        if (quantity < 0)
            throw new ArgumentException("Stock cannot be negative");
        
        StockQuantity = quantity;
        
        AddEvent(new StockUpdatedEvent(Id, quantity));
    }
    
    public void UpdatePrice(decimal newPrice)
    {
        if (newPrice <= 0)
            throw new ArgumentException("Price must be positive");
        
        Price = newPrice;
        
        AddEvent(new PriceUpdatedEvent(Id, newPrice));
    }
    
    private void AddEvent(IDomainEvent @event)
    {
        _events.Add(@event);
    }
    
    public IReadOnlyList<IDomainEvent> GetEvents() => _events.AsReadOnly();
    
    public void ClearEvents() => _events.Clear();
}

/// <summary>
/// Command handler processes commands.
/// </summary>
public class ProductCommandHandler
{
    private readonly IProductRepository _repository;
    private readonly IEventBus _eventBus;
    
    public async Task<Result<Guid, CommandError>> HandleAsync(
        ProductCommand.CreateProduct command)
    {
        var product = Product.Create(
            command.Name,
            command.Price,
            command.InitialStock,
            command.Category,
            command.Description);
        
        await _repository.SaveAsync(product);
        
        // Publish events for read model to consume
        foreach (var @event in product.GetEvents())
        {
            await _eventBus.PublishAsync(@event);
        }
        
        return Result<Guid, CommandError>.Success(product.Id);
    }
    
    public async Task<Result<Unit, CommandError>> HandleAsync(
        ProductCommand.UpdateStock command)
    {
        var product = await _repository.GetByIdAsync(command.ProductId);
        
        if (product == null)
            return Result<Unit, CommandError>.Failure(
                new CommandError.NotFound("Product not found"));
        
        product.UpdateStock(command.Quantity);
        
        await _repository.SaveAsync(product);
        
        foreach (var @event in product.GetEvents())
        {
            await _eventBus.PublishAsync(@event);
        }
        
        return Result<Unit, CommandError>.Success(Unit.Value);
    }
}

// ============================================
// QUERY SIDE (Reads)
// ============================================

/// <summary>
/// Queries represent requests for data.
/// </summary>
public abstract record ProductQuery
{
    public sealed record GetProduct(Guid ProductId) : ProductQuery;
    
    public sealed record GetProductsByCategory(string Category) : ProductQuery;
    
    public sealed record SearchProducts(
        string SearchTerm,
        int PageNumber,
        int PageSize) : ProductQuery;
}

/// <summary>
/// Read model optimized for queries (denormalized).
/// </summary>
public class ProductReadModel
{
    public Guid Id { get; set; }
    public string Name { get; set; } = "";
    public decimal Price { get; set; }
    public int StockQuantity { get; set; }
    public string Category { get; set; } = "";
    public string Description { get; set; } = "";
    
    // Pre-calculated fields (no joins needed!)
    public decimal AverageRating { get; set; }
    public int ReviewCount { get; set; }
    public int TotalSales { get; set; }
    public DateTimeOffset LastUpdated { get; set; }
    
    // Denormalized data
    public List<string> Tags { get; set; } = new();
    public string CategoryPath { get; set; } = "";  // e.g., "Electronics > Computers > Laptops"
}

/// <summary>
/// Query handler reads from denormalized read model.
/// </summary>
public class ProductQueryHandler
{
    private readonly IReadModelRepository<ProductReadModel> _readRepository;
    
    public async Task<Result<ProductReadModel, QueryError>> HandleAsync(
        ProductQuery.GetProduct query)
    {
        // Simple query, no joins, pre-calculated data
        var product = await _readRepository.GetByIdAsync(query.ProductId);
        
        if (product == null)
            return Result<ProductReadModel, QueryError>.Failure(
                new QueryError.NotFound("Product not found"));
        
        return Result<ProductReadModel, QueryError>.Success(product);
    }
    
    public async Task<Result<List<ProductReadModel>, QueryError>> HandleAsync(
        ProductQuery.GetProductsByCategory query)
    {
        // Fast indexed query
        var products = await _readRepository.WhereAsync(
            p => p.Category == query.Category);
        
        return Result<List<ProductReadModel>, QueryError>.Success(products);
    }
    
    public async Task<Result<List<ProductReadModel>, QueryError>> HandleAsync(
        ProductQuery.SearchProducts query)
    {
        // Full-text search on denormalized data
        var products = await _readRepository.FullTextSearchAsync(
            query.SearchTerm,
            query.PageNumber,
            query.PageSize);
        
        return Result<List<ProductReadModel>, QueryError>.Success(products);
    }
}

// ============================================
// PROJECTION (Event Handlers Update Read Model)
// ============================================

/// <summary>
/// Projection updates read model when write model changes.
/// </summary>
public class ProductProjection
{
    private readonly IReadModelRepository<ProductReadModel> _readRepository;
    
    public async Task HandleAsync(ProductCreatedEvent @event)
    {
        var readModel = new ProductReadModel
        {
            Id = @event.ProductId,
            Name = @event.Name,
            Price = @event.Price,
            StockQuantity = 0,
            Category = "",
            Description = "",
            AverageRating = 0,
            ReviewCount = 0,
            TotalSales = 0,
            LastUpdated = DateTimeOffset.UtcNow
        };
        
        await _readRepository.InsertAsync(readModel);
    }
    
    public async Task HandleAsync(StockUpdatedEvent @event)
    {
        var readModel = await _readRepository.GetByIdAsync(@event.ProductId);
        
        if (readModel != null)
        {
            readModel.StockQuantity = @event.NewQuantity;
            readModel.LastUpdated = DateTimeOffset.UtcNow;
            
            await _readRepository.UpdateAsync(readModel);
        }
    }
    
    public async Task HandleAsync(PriceUpdatedEvent @event)
    {
        var readModel = await _readRepository.GetByIdAsync(@event.ProductId);
        
        if (readModel != null)
        {
            readModel.Price = @event.NewPrice;
            readModel.LastUpdated = DateTimeOffset.UtcNow;
            
            await _readRepository.UpdateAsync(readModel);
        }
    }
    
    public async Task HandleAsync(ReviewAddedEvent @event)
    {
        var readModel = await _readRepository.GetByIdAsync(@event.ProductId);
        
        if (readModel != null)
        {
            // Update pre-calculated fields
            readModel.ReviewCount++;
            readModel.AverageRating = 
                (readModel.AverageRating * (readModel.ReviewCount - 1) + @event.Rating) 
                / readModel.ReviewCount;
            readModel.LastUpdated = DateTimeOffset.UtcNow;
            
            await _readRepository.UpdateAsync(readModel);
        }
    }
}
```

## API Layer with CQRS

```csharp
[ApiController]
[Route("api/products")]
public class ProductsController : ControllerBase
{
    private readonly ICommandHandler<ProductCommand> _commandHandler;
    private readonly IQueryHandler<ProductQuery> _queryHandler;
    
    // Commands go to write model
    [HttpPost]
    public async Task<IActionResult> CreateProduct(CreateProductDto dto)
    {
        var command = new ProductCommand.CreateProduct(
            dto.Name,
            dto.Price,
            dto.InitialStock,
            dto.Category,
            dto.Description);
        
        var result = await _commandHandler.HandleAsync(command);
        
        return result.Match<IActionResult>(
            onSuccess: productId => CreatedAtAction(
                nameof(GetProduct),
                new { id = productId },
                new { id = productId }),
            onFailure: error => BadRequest(error));
    }
    
    [HttpPut("{id}/stock")]
    public async Task<IActionResult> UpdateStock(Guid id, UpdateStockDto dto)
    {
        var command = new ProductCommand.UpdateStock(id, dto.Quantity);
        
        var result = await _commandHandler.HandleAsync(command);
        
        return result.Match<IActionResult>(
            onSuccess: _ => NoContent(),
            onFailure: error => BadRequest(error));
    }
    
    // Queries go to read model
    [HttpGet("{id}")]
    public async Task<IActionResult> GetProduct(Guid id)
    {
        var query = new ProductQuery.GetProduct(id);
        
        var result = await _queryHandler.HandleAsync(query);
        
        return result.Match<IActionResult>(
            onSuccess: product => Ok(product),
            onFailure: error => NotFound());
    }
    
    [HttpGet("search")]
    public async Task<IActionResult> SearchProducts(
        [FromQuery] string term,
        [FromQuery] int page = 1,
        [FromQuery] int pageSize = 20)
    {
        var query = new ProductQuery.SearchProducts(term, page, pageSize);
        
        var result = await _queryHandler.HandleAsync(query);
        
        return result.Match<IActionResult>(
            onSuccess: products => Ok(products),
            onFailure: error => BadRequest(error));
    }
}
```

## Different Databases for Read and Write

```csharp
// Write side: Relational database for ACID guarantees
public class WriteDatabase
{
    private readonly SqlConnection _connection;
    
    public async Task SaveProductAsync(Product product)
    {
        await using var transaction = await _connection.BeginTransactionAsync();
        
        // Store normalized data
        await _connection.ExecuteAsync(
            "INSERT INTO Products (Id, Name, Price, StockQuantity) " +
            "VALUES (@Id, @Name, @Price, @StockQuantity)",
            product,
            transaction);
        
        await transaction.CommitAsync();
    }
}

// Read side: Document database for fast queries
public class ReadDatabase
{
    private readonly IMongoCollection<ProductReadModel> _collection;
    
    public async Task<ProductReadModel> GetByIdAsync(Guid id)
    {
        // Fast lookup, all data in single document
        return await _collection
            .Find(p => p.Id == id)
            .FirstOrDefaultAsync();
    }
    
    public async Task<List<ProductReadModel>> FullTextSearchAsync(
        string searchTerm,
        int page,
        int pageSize)
    {
        // Full-text search optimized for reads
        return await _collection
            .Find(Builders<ProductReadModel>.Filter.Text(searchTerm))
            .Skip((page - 1) * pageSize)
            .Limit(pageSize)
            .ToListAsync();
    }
}
```

## Why It's a Problem (Not Using CQRS)

1. **Conflicting requirements**: Writes need normalization, reads need denormalization
2. **Poor performance**: Complex joins on every query
3. **Scalability limits**: Can't scale reads independently from writes
4. **Complex code**: Single model serves multiple purposes badly

## Symptoms

- Slow queries with many joins
- Write operations loading unnecessary data
- Cache strategies become complex
- ORM queries getting out of control
- Difficulty optimizing reads without breaking writes

## Benefits

- **Optimized reads**: Denormalized data, no joins, pre-calculated values
- **Optimized writes**: Focused on validation and business logic
- **Independent scaling**: Scale read and write databases separately
- **Flexibility**: Use different databases for different needs
- **Simplicity**: Each model has single responsibility

## See Also

- [Event Sourcing](./event-sourcing.md) — often combined with CQRS
- [Eventual Consistency](./eventual-consistency.md) — read models update asynchronously
- [Event-Driven Architecture](./event-driven-architecture.md) — events connect write and read sides
- [Materialized Views](./materialized-views.md) — pre-computed read models
