# Infrastructure Layer Organization

> Organize infrastructure code by technology (Persistence, Email, ExternalApis) or by feature—keeping adapters cohesive and making it easy to locate and swap implementations.

## Problem

When infrastructure code is disorganized, it becomes hard to find implementations, understand which technologies are being used, and swap adapters.

## Core Concepts

**Two Organizational Strategies:**

1. **By Technology** (Traditional): Group by technical concern
2. **By Feature** (Vertical Slices): Group by business capability

Choose based on team size and architecture style.

## Example 1: Organization by Technology

```
Infrastructure/
├── Persistence/                    ← Database concerns
│   ├── Entities/                   ← EF entities
│   │   ├── OrderEntity.cs
│   │   ├── ProductEntity.cs
│   │   └── CustomerEntity.cs
│   ├── Configuration/              ← EF configuration
│   │   ├── OrderConfiguration.cs
│   │   └── ProductConfiguration.cs
│   ├── Repositories/               ← Repository implementations
│   │   ├── OrderRepository.cs
│   │   ├── ProductRepository.cs
│   │   └── CustomerRepository.cs
│   ├── Migrations/                 ← Database migrations
│   │   └── 20240101_InitialCreate.cs
│   ├── MyDbContext.cs
│   └── UnitOfWork.cs
│
├── Email/                          ← Email concerns
│   ├── SendGridEmailAdapter.cs
│   ├── SmtpEmailAdapter.cs
│   └── ConsoleEmailAdapter.cs
│
├── ExternalApis/                   ← External service integrations
│   ├── Payment/
│   │   ├── StripePaymentAdapter.cs
│   │   └── PayPalPaymentAdapter.cs
│   ├── Inventory/
│   │   └── RestInventoryAdapter.cs
│   └── Shipping/
│       └── FedExShippingAdapter.cs
│
├── FileStorage/                    ← File storage concerns
│   ├── S3FileStorageAdapter.cs
│   ├── AzureBlobStorageAdapter.cs
│   └── LocalFileStorageAdapter.cs
│
├── MessageQueues/                  ← Message queue concerns
│   ├── RabbitMqPublisher.cs
│   └── AzureServiceBusPublisher.cs
│
├── Caching/                        ← Caching concerns
│   ├── RedisCacheAdapter.cs
│   └── MemoryCacheAdapter.cs
│
└── DependencyInjection/            ← DI configuration
    └── InfrastructureModule.cs
```

### Example Implementation

```csharp
// Infrastructure/Persistence/Repositories/OrderRepository.cs
namespace MyApp.Infrastructure.Persistence.Repositories
{
    /// <summary>
    /// Repository implementation using EF Core.
    /// </summary>
    public sealed class OrderRepository : IOrderRepository
    {
        private readonly MyDbContext _dbContext;
        
        public async Task<Option<Order>> GetByIdAsync(OrderId id)
        {
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
            var entity = MapToEntity(order);
            _dbContext.Orders.Update(entity);
        }
    }
}

// Infrastructure/Email/SendGridEmailAdapter.cs
namespace MyApp.Infrastructure.Email
{
    public sealed class SendGridEmailAdapter : IEmailService
    {
        private readonly ISendGridClient _client;
        
        public async Task SendOrderConfirmationAsync(OrderId orderId, Email recipient)
        {
            var message = new SendGridMessage
            {
                Subject = "Order Confirmation",
                To = new List<EmailAddress> { new(recipient.Value) }
            };
            
            await _client.SendEmailAsync(message);
        }
    }
}

// Infrastructure/ExternalApis/Payment/StripePaymentAdapter.cs
namespace MyApp.Infrastructure.ExternalApis.Payment
{
    public sealed class StripePaymentAdapter : IPaymentGateway
    {
        private readonly IStripeClient _client;
        
        public async Task<Result<PaymentId, PaymentError>> ProcessPaymentAsync(
            Money amount,
            PaymentMethod paymentMethod)
        {
            var chargeOptions = new ChargeCreateOptions
            {
                Amount = (long)(amount.Amount * 100),
                Currency = amount.Currency.Code.ToLower()
            };
            
            var charge = await _client.Charges.CreateAsync(chargeOptions);
            return Result<PaymentId, PaymentError>.Success(new PaymentId(charge.Id));
        }
    }
}
```

## Example 2: Organization by Feature (Vertical Slices)

```
Infrastructure/
├── Orders/                         ← All infrastructure for orders
│   ├── OrderRepository.cs
│   ├── OrderEntity.cs
│   ├── OrderConfiguration.cs
│   └── OrderReadModel.cs
│
├── Products/                       ← All infrastructure for products
│   ├── ProductRepository.cs
│   ├── ProductEntity.cs
│   └── ProductConfiguration.cs
│
├── Customers/                      ← All infrastructure for customers
│   ├── CustomerRepository.cs
│   ├── CustomerEntity.cs
│   └── CustomerConfiguration.cs
│
├── Notifications/                  ← Cross-cutting notification services
│   ├── EmailService.cs
│   └── SmsService.cs
│
└── Shared/                         ← Shared infrastructure
    ├── DbContext.cs
    ├── UnitOfWork.cs
    └── DependencyInjection/
        └── InfrastructureModule.cs
```

## DI Configuration by Technology

```csharp
// Infrastructure/DependencyInjection/InfrastructureModule.cs
namespace MyApp.Infrastructure.DependencyInjection
{
    public static class InfrastructureModule
    {
        public static IServiceCollection AddInfrastructure(
            this IServiceCollection services,
            IConfiguration configuration)
        {
            services.AddPersistence(configuration);
            services.AddEmailServices(configuration);
            services.AddExternalApis(configuration);
            services.AddFileStorage(configuration);
            services.AddCaching(configuration);
            
            return services;
        }
        
        private static IServiceCollection AddPersistence(
            this IServiceCollection services,
            IConfiguration configuration)
        {
            var connectionString = configuration.GetConnectionString("DefaultConnection");
            
            services.AddDbContext<MyDbContext>(options =>
                options.UseSqlServer(connectionString));
            
            services.AddScoped<IOrderRepository, OrderRepository>();
            services.AddScoped<IProductRepository, ProductRepository>();
            services.AddScoped<ICustomerRepository, CustomerRepository>();
            services.AddScoped<IUnitOfWork, EfCoreUnitOfWork>();
            
            return services;
        }
        
        private static IServiceCollection AddEmailServices(
            this IServiceCollection services,
            IConfiguration configuration)
        {
            var provider = configuration.GetValue<string>("Email:Provider");
            
            switch (provider)
            {
                case "SendGrid":
                    services.AddSingleton<ISendGridClient>(sp =>
                        new SendGridClient(configuration["Email:SendGrid:ApiKey"]));
                    services.AddScoped<IEmailService, SendGridEmailAdapter>();
                    break;
                
                case "Smtp":
                    services.AddScoped<IEmailService, SmtpEmailAdapter>();
                    break;
                
                default:
                    services.AddScoped<IEmailService, ConsoleEmailAdapter>();
                    break;
            }
            
            return services;
        }
        
        private static IServiceCollection AddExternalApis(
            this IServiceCollection services,
            IConfiguration configuration)
        {
            // Payment gateway
            services.AddScoped<IPaymentGateway, StripePaymentAdapter>();
            services.AddSingleton<IStripeClient>(sp =>
                new StripeClient(configuration["Stripe:SecretKey"]));
            
            // Inventory service
            services.AddHttpClient<IInventoryService, RestInventoryAdapter>(client =>
            {
                client.BaseAddress = new Uri(configuration["InventoryApi:BaseUrl"]);
            });
            
            return services;
        }
        
        private static IServiceCollection AddFileStorage(
            this IServiceCollection services,
            IConfiguration configuration)
        {
            var provider = configuration.GetValue<string>("FileStorage:Provider");
            
            switch (provider)
            {
                case "S3":
                    services.AddSingleton<IAmazonS3>(sp =>
                        new AmazonS3Client(
                            configuration["AWS:AccessKey"],
                            configuration["AWS:SecretKey"],
                            RegionEndpoint.USEast1));
                    services.AddScoped<IFileStorageService, S3FileStorageAdapter>();
                    break;
                
                case "AzureBlob":
                    services.AddSingleton(sp =>
                        new BlobServiceClient(configuration["Azure:StorageConnectionString"]));
                    services.AddScoped<IFileStorageService, AzureBlobStorageAdapter>();
                    break;
                
                default:
                    services.AddScoped<IFileStorageService, LocalFileStorageAdapter>();
                    break;
            }
            
            return services;
        }
        
        private static IServiceCollection AddCaching(
            this IServiceCollection services,
            IConfiguration configuration)
        {
            var useRedis = configuration.GetValue<bool>("Cache:UseRedis");
            
            if (useRedis)
            {
                services.AddStackExchangeRedisCache(options =>
                {
                    options.Configuration = configuration["Cache:Redis:ConnectionString"];
                });
                services.AddScoped<ICacheService, RedisCacheAdapter>();
            }
            else
            {
                services.AddMemoryCache();
                services.AddScoped<ICacheService, MemoryCacheAdapter>();
            }
            
            return services;
        }
    }
}
```

## Benefits

**By Technology:**
- Easy to swap entire technology (replace all of Persistence/)
- Clear separation of technical concerns
- Easy to see all database code in one place

**By Feature:**
- Cohesive feature implementations
- Easy to remove entire features
- Better for vertical slice architecture

## Guidelines

1. **Separate Concerns**: Each adapter in its own file
2. **Configuration**: Centralize DI configuration in InfrastructureModule
3. **Naming**: Clear naming that indicates technology (SqlOrderRepository, SendGridEmailAdapter)
4. **Dependencies**: Keep infrastructure dependencies isolated to Infrastructure project
5. **Testing**: Test each adapter independently with integration tests

## See Also

- [Clean Architecture](./clean-architecture.md) — layer organization
- [Composition Root](./composition-root.md) — DI configuration
- [Infrastructure Service Adapters](./infrastructure-service-adapters.md) — implementing adapters
- [Port-Adapter Separation](./port-adapter-separation.md) — hexagonal architecture
