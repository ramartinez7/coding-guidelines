# Persistence Ignorance

> Domain entities should have no knowledge of how they're persisted—no ORM attributes, no database concerns, keeping the domain pure and persistence swappable.

## Problem

When domain entities are polluted with persistence attributes ([Table], [Column], [Key]), the domain becomes tightly coupled to the database technology, making it hard to test and impossible to swap persistence mechanisms.

## Core Concept

**Persistence Ignorance** means:
- Domain entities have NO persistence attributes
- No `virtual` properties for lazy loading
- No parameterless constructors for ORMs
- Infrastructure layer handles all mapping

## Example

### ❌ Before (Persistence Leaks into Domain)

```csharp
// ❌ Domain entity knows about Entity Framework
[Table("Orders")]
public class Order
{
    [Key]
    [Column("order_id")]
    public Guid Id { get; set; }
    
    [Column("customer_id")]
    public Guid CustomerId { get; set; }
    
    [Column("total_amount")]
    public decimal Total { get; set; }
    
    // ❌ Virtual for EF lazy loading
    public virtual Customer Customer { get; set; }
    
    // ❌ Virtual for EF lazy loading
    public virtual ICollection<OrderItem> Items { get; set; }
    
    // ❌ Parameterless constructor for EF
    public Order() { }
}
```

### ✅ After (Persistence Ignorant Domain)

```csharp
// Domain/Entities/Order.cs - Pure domain entity
namespace MyApp.Domain.Entities
{
    /// <summary>
    /// Pure domain entity - no knowledge of persistence.
    /// </summary>
    public sealed class Order
    {
        private readonly List<OrderItem> _items = new();
        
        public OrderId Id { get; }
        public CustomerId CustomerId { get; }
        public IReadOnlyList<OrderItem> Items => _items.AsReadOnly();
        public Money Total { get; private set; }
        public OrderStatus Status { get; private set; }
        
        // ✅ Private constructor enforces factory method
        private Order(OrderId id, CustomerId customerId)
        {
            Id = id;
            CustomerId = customerId;
            Total = Money.Zero;
            Status = OrderStatus.Draft;
        }
        
        public static Order Create(CustomerId customerId)
        {
            return new Order(OrderId.New(), customerId);
        }
        
        public Result<Unit, string> AddItem(Product product, int quantity)
        {
            if (quantity <= 0)
                return Result<Unit, string>.Failure("Quantity must be positive");
            
            var item = new OrderItem(product.Id, quantity, product.Price);
            _items.Add(item);
            RecalculateTotal();
            
            return Result<Unit, string>.Success(Unit.Value);
        }
        
        private void RecalculateTotal()
        {
            Total = _items
                .Select(i => i.Total)
                .Aggregate(Money.Zero, (sum, price) => sum + price);
        }
    }
}

// Infrastructure/Persistence/Entities/OrderEntity.cs - Persistence model
namespace MyApp.Infrastructure.Persistence.Entities
{
    /// <summary>
    /// Persistence entity - separate from domain.
    /// Contains all ORM attributes.
    /// </summary>
    [Table("Orders")]
    public class OrderEntity
    {
        [Key]
        [Column("order_id")]
        public Guid Id { get; set; }
        
        [Column("customer_id")]
        public Guid CustomerId { get; set; }
        
        [Column("total_amount")]
        public decimal TotalAmount { get; set; }
        
        [Column("total_currency")]
        public string TotalCurrency { get; set; } = string.Empty;
        
        [Column("status")]
        public string Status { get; set; } = string.Empty;
        
        [Column("created_at")]
        public DateTime CreatedAt { get; set; }
        
        // EF navigation properties
        public virtual ICollection<OrderItemEntity> Items { get; set; } = new List<OrderItemEntity>();
        
        // Parameterless constructor for EF
        public OrderEntity() { }
    }
}

// Infrastructure/Persistence/Repositories/OrderRepository.cs
namespace MyApp.Infrastructure.Persistence.Repositories
{
    /// <summary>
    /// Repository translates between domain and persistence models.
    /// </summary>
    public sealed class OrderRepository : IOrderRepository
    {
        private readonly MyDbContext _dbContext;
        
        public OrderRepository(MyDbContext dbContext)
        {
            _dbContext = dbContext;
        }
        
        public async Task<Option<Order>> GetByIdAsync(OrderId id)
        {
            var entity = await _dbContext.Orders
                .Include(o => o.Items)
                .FirstOrDefaultAsync(o => o.Id == id.Value);
            
            if (entity == null)
                return Option<Order>.None;
            
            // ✅ Map persistence model to domain model
            var domainOrder = MapToDomain(entity);
            return Option<Order>.Some(domainOrder);
        }
        
        public async Task SaveAsync(Order order)
        {
            var entity = await _dbContext.Orders.FindAsync(order.Id.Value);
            
            if (entity == null)
            {
                // ✅ Map domain model to persistence model
                entity = MapToEntity(order);
                _dbContext.Orders.Add(entity);
            }
            else
            {
                // Update existing entity
                entity.TotalAmount = order.Total.Amount;
                entity.TotalCurrency = order.Total.Currency.Code;
                entity.Status = order.Status.ToString();
            }
        }
        
        private Order MapToDomain(OrderEntity entity)
        {
            // Use reflection or a mapping library to reconstruct domain object
            // This is complex but keeps domain pure
            // Alternatively, use a dedicated mapper like AutoMapper
            
            var order = (Order)FormatterServices.GetUninitializedObject(typeof(Order));
            
            // Set private fields using reflection
            var idField = typeof(Order).GetField("<Id>k__BackingField", 
                BindingFlags.NonPublic | BindingFlags.Instance);
            idField!.SetValue(order, new OrderId(entity.Id));
            
            // ... map other fields
            
            return order;
        }
        
        private OrderEntity MapToEntity(Order order)
        {
            return new OrderEntity
            {
                Id = order.Id.Value,
                CustomerId = order.CustomerId.Value,
                TotalAmount = order.Total.Amount,
                TotalCurrency = order.Total.Currency.Code,
                Status = order.Status.ToString(),
                CreatedAt = DateTime.UtcNow,
                Items = order.Items.Select(i => new OrderItemEntity
                {
                    ProductId = i.ProductId.Value,
                    Quantity = i.Quantity,
                    Price = i.Price.Amount
                }).ToList()
            };
        }
    }
}

// Infrastructure/Persistence/MyDbContext.cs
namespace MyApp.Infrastructure.Persistence
{
    public class MyDbContext : DbContext
    {
        // ✅ DbContext knows about persistence entities, not domain entities
        public DbSet<OrderEntity> Orders { get; set; }
        public DbSet<OrderItemEntity> OrderItems { get; set; }
        
        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            // ✅ All EF configuration in infrastructure layer
            modelBuilder.Entity<OrderEntity>(entity =>
            {
                entity.HasKey(e => e.Id);
                entity.Property(e => e.TotalAmount).HasPrecision(18, 2);
                entity.HasMany(e => e.Items)
                      .WithOne()
                      .HasForeignKey("OrderId");
            });
        }
    }
}
```

## Alternative: Fluent API Configuration

```csharp
// Infrastructure/Persistence/Configuration/OrderConfiguration.cs
namespace MyApp.Infrastructure.Persistence.Configuration
{
    /// <summary>
    /// EF configuration using Fluent API instead of attributes.
    /// Domain entities stay completely pure.
    /// </summary>
    public class OrderConfiguration : IEntityTypeConfiguration<OrderEntity>
    {
        public void Configure(EntityTypeBuilder<OrderEntity> builder)
        {
            builder.ToTable("Orders");
            
            builder.HasKey(o => o.Id);
            
            builder.Property(o => o.Id)
                .HasColumnName("order_id")
                .IsRequired();
            
            builder.Property(o => o.CustomerId)
                .HasColumnName("customer_id")
                .IsRequired();
            
            builder.Property(o => o.TotalAmount)
                .HasColumnName("total_amount")
                .HasPrecision(18, 2)
                .IsRequired();
            
            builder.Property(o => o.Status)
                .HasColumnName("status")
                .HasConversion<string>()
                .IsRequired();
            
            builder.HasMany(o => o.Items)
                .WithOne()
                .HasForeignKey("OrderId")
                .OnDelete(DeleteBehavior.Cascade);
        }
    }
}
```

## Benefits

1. **Pure Domain**: Domain entities have no infrastructure dependencies
2. **Testable**: Test domain logic without database
3. **Swappable**: Change persistence technology without changing domain
4. **Maintainable**: Clear separation of concerns

## Drawbacks and Solutions

**Problem**: More code required (separate persistence models, mapping)
**Solution**: Use mapping libraries (AutoMapper, Mapster) or reflection-based mappers

**Problem**: Performance overhead from mapping
**Solution**: Only matters for high-throughput scenarios; optimize critical paths

**Problem**: Complexity in repository implementations
**Solution**: Complexity is isolated in infrastructure layer; domain stays simple

## See Also

- [Clean Architecture](./clean-architecture.md) — layered dependencies
- [Repository Pattern](./repository-pattern.md) — data access abstraction
- [Domain Model Isolation](./domain-model-isolation.md) — pure domain logic
- [DTO vs Domain Boundary](./dto-domain-boundary.md) — separation of models
