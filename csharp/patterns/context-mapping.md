# Context Mapping (Strategic Domain Boundaries and Integration)

> Shared models between domains create coupling and confusion—use context maps to define bounded context relationships and translation strategies.

## Problem

When multiple bounded contexts share the same domain model or integrate without clear boundaries, changes in one context break others, and the same term can mean different things causing confusion.

## Example

### ❌ Before

```csharp
// Shared model across all contexts—causes tight coupling
public class Customer
{
    public Guid Id { get; set; }
    public string Name { get; set; }
    public string Email { get; set; }
    
    // Sales context properties
    public decimal CreditLimit { get; set; }
    public string SalesRepId { get; set; }
    
    // Shipping context properties
    public string ShippingAddress { get; set; }
    public string PreferredCarrier { get; set; }
    
    // Support context properties
    public int TicketCount { get; set; }
    public DateTime LastContactDate { get; set; }
    
    // Billing context properties
    public string PaymentMethod { get; set; }
    public decimal OutstandingBalance { get; set; }
}

// All contexts depend on the same model
public class SalesService
{
    public async Task<Customer> GetCustomer(Guid id)
    {
        // Returns customer with ALL properties even though
        // sales only needs Name, Email, CreditLimit, SalesRepId
        return await _db.Customers.FindAsync(id);
    }
}
```

**Problems:**
- Single model serves multiple contexts with different needs
- Changes in one context break others
- God object with properties from all contexts
- Unclear which properties belong to which context
- Can't evolve contexts independently

### ✅ After

```csharp
// ============================================
// SALES CONTEXT (Bounded Context #1)
// ============================================
namespace Sales.Domain
{
    /// <summary>
    /// Customer in Sales context: focused on sales activities.
    /// </summary>
    public sealed class Customer
    {
        public CustomerId Id { get; }
        public CustomerName Name { get; }
        public EmailAddress Email { get; }
        public Money CreditLimit { get; private set; }
        public SalesRepId AssignedSalesRep { get; private set; }
        
        // Sales-specific behavior
        public Result<Unit, string> UpdateCreditLimit(Money newLimit)
        {
            if (newLimit.Amount < 0)
                return Result<Unit, string>.Failure("Credit limit cannot be negative");
            
            CreditLimit = newLimit;
            return Result<Unit, string>.Success(Unit.Value);
        }
    }
}

// ============================================
// SHIPPING CONTEXT (Bounded Context #2)
// ============================================
namespace Shipping.Domain
{
    /// <summary>
    /// Customer in Shipping context: focused on shipment delivery.
    /// Different from Sales.Customer—can evolve independently.
    /// </summary>
    public sealed class ShippingRecipient
    {
        public RecipientId Id { get; }  // Different ID type!
        public RecipientName Name { get; }
        public Address ShippingAddress { get; private set; }
        public Option<PreferredCarrier> PreferredCarrier { get; private set; }
        public IReadOnlyList<DeliveryInstruction> SpecialInstructions { get; }
        
        // Shipping-specific behavior
        public Result<Unit, string> UpdateShippingAddress(Address newAddress)
        {
            ShippingAddress = newAddress;
            return Result<Unit, string>.Success(Unit.Value);
        }
    }
}

// ============================================
// SUPPORT CONTEXT (Bounded Context #3)
// ============================================
namespace Support.Domain
{
    /// <summary>
    /// Customer in Support context: focused on support interactions.
    /// </summary>
    public sealed class SupportContact
    {
        public ContactId Id { get; }
        public ContactName Name { get; }
        public EmailAddress Email { get; }
        public PhoneNumber Phone { get; }
        public SupportTier Tier { get; private set; }
        public IReadOnlyList<TicketId> TicketHistory { get; }
        public DateTimeOffset LastContactedAt { get; private set; }
        
        // Support-specific behavior
        public Ticket CreateTicket(TicketDescription description, Priority priority)
        {
            LastContactedAt = DateTimeOffset.UtcNow;
            return Ticket.Create(TicketId.New(), Id, description, priority);
        }
    }
}

// ============================================
// BILLING CONTEXT (Bounded Context #4)
// ============================================
namespace Billing.Domain
{
    /// <summary>
    /// Customer in Billing context: focused on payments and invoices.
    /// </summary>
    public sealed class BillingAccount
    {
        public AccountId Id { get; }
        public AccountName Name { get; }
        public PaymentMethod DefaultPaymentMethod { get; private set; }
        public Money OutstandingBalance { get; private set; }
        public IReadOnlyList<Invoice> Invoices { get; }
        
        // Billing-specific behavior
        public Result<Unit, string> RecordPayment(Money amount, PaymentMethod method)
        {
            if (amount.Amount <= 0)
                return Result<Unit, string>.Failure("Payment amount must be positive");
            
            OutstandingBalance = Money.FromAmount(
                OutstandingBalance.Amount - amount.Amount,
                OutstandingBalance.Currency);
            
            return Result<Unit, string>.Success(Unit.Value);
        }
    }
}

// ============================================
// CONTEXT MAPPING: Anti-Corruption Layer
// ============================================
namespace Sales.Integration
{
    /// <summary>
    /// Translates between Sales context and Shipping context.
    /// Prevents Shipping concepts from polluting Sales domain.
    /// </summary>
    public sealed class ShippingContextAdapter
    {
        private readonly IShippingService _shippingService;
        
        public ShippingContextAdapter(IShippingService shippingService)
        {
            _shippingService = shippingService;
        }
        
        public async Task<Result<ShipmentStatus, string>> GetShipmentStatus(
            Sales.Domain.CustomerId customerId,
            OrderId orderId)
        {
            // Translate Sales ID to Shipping ID
            var recipientId = MapCustomerIdToRecipientId(customerId);
            
            // Call Shipping context
            var shippingResult = await _shippingService.GetStatusAsync(recipientId, orderId);
            
            if (!shippingResult.IsSuccess)
                return Result<ShipmentStatus, string>.Failure(shippingResult.Error);
            
            // Translate Shipping domain concept to Sales domain concept
            var status = MapShippingStatusToSalesStatus(shippingResult.Value);
            
            return Result<ShipmentStatus, string>.Success(status);
        }
        
        private Shipping.Domain.RecipientId MapCustomerIdToRecipientId(
            Sales.Domain.CustomerId customerId)
        {
            // Translation logic between contexts
            return new Shipping.Domain.RecipientId(customerId.Value);
        }
        
        private ShipmentStatus MapShippingStatusToSalesStatus(
            Shipping.Domain.DeliveryStatus shippingStatus)
        {
            // Translate concepts from Shipping vocabulary to Sales vocabulary
            return shippingStatus switch
            {
                Shipping.Domain.DeliveryStatus.Pending => ShipmentStatus.Processing,
                Shipping.Domain.DeliveryStatus.InTransit => ShipmentStatus.Shipped,
                Shipping.Domain.DeliveryStatus.Delivered => ShipmentStatus.Delivered,
                _ => ShipmentStatus.Unknown
            };
        }
    }
    
    // Sales context's own status type (not shared with Shipping)
    public enum ShipmentStatus
    {
        Processing,
        Shipped,
        Delivered,
        Unknown
    }
}

// ============================================
// CONTEXT MAPPING: Published Language
// ============================================
namespace Integration.Contracts
{
    /// <summary>
    /// Published language: Well-defined contract for external integration.
    /// Multiple contexts can consume this without coupling to each other.
    /// </summary>
    public sealed record CustomerCreatedEvent
    {
        [JsonProperty("customer_id")]
        public string CustomerId { get; init; }
        
        [JsonProperty("customer_name")]
        public string CustomerName { get; init; }
        
        [JsonProperty("email")]
        public string Email { get; init; }
        
        [JsonProperty("created_at")]
        public string CreatedAt { get; init; }
    }
    
    /// <summary>
    /// Each context translates from published language to its own domain model.
    /// </summary>
    public sealed class SalesCustomerEventHandler : IEventHandler<CustomerCreatedEvent>
    {
        private readonly ISalesCustomerRepository _repo;
        
        public async Task HandleAsync(CustomerCreatedEvent evt)
        {
            // Translate published language to Sales domain
            var customerId = Sales.Domain.CustomerId.Parse(evt.CustomerId);
            var name = Sales.Domain.CustomerName.Create(evt.CustomerName).Value;
            var email = EmailAddress.Create(evt.Email).Value;
            
            var customer = Sales.Domain.Customer.Create(
                customerId,
                name,
                email,
                Money.Zero);  // Default credit limit in Sales context
            
            await _repo.SaveAsync(customer);
        }
    }
    
    public sealed class ShippingRecipientEventHandler : IEventHandler<CustomerCreatedEvent>
    {
        private readonly IShippingRecipientRepository _repo;
        
        public async Task HandleAsync(CustomerCreatedEvent evt)
        {
            // Translate published language to Shipping domain
            var recipientId = Shipping.Domain.RecipientId.Parse(evt.CustomerId);
            var name = Shipping.Domain.RecipientName.Create(evt.CustomerName).Value;
            
            var recipient = Shipping.Domain.ShippingRecipient.Create(
                recipientId,
                name,
                Address.Default);  // Default address in Shipping context
            
            await _repo.SaveAsync(recipient);
        }
    }
}

// ============================================
// CONTEXT MAPPING: Shared Kernel
// ============================================
namespace Shared.Kernel
{
    /// <summary>
    /// Shared kernel: Common types used across multiple bounded contexts.
    /// Use sparingly—only for truly universal concepts.
    /// Changes here affect all contexts.
    /// </summary>
    
    // Universal value objects
    public sealed record EmailAddress
    {
        public string Value { get; }
        
        private EmailAddress(string value) => Value = value;
        
        public static Result<EmailAddress, string> Create(string email)
        {
            if (string.IsNullOrWhiteSpace(email))
                return Result<EmailAddress, string>.Failure("Email required");
            
            if (!email.Contains("@"))
                return Result<EmailAddress, string>.Failure("Invalid email");
            
            return Result<EmailAddress, string>.Success(
                new EmailAddress(email.ToLowerInvariant()));
        }
    }
    
    public sealed record Money
    {
        public decimal Amount { get; }
        public Currency Currency { get; }
        
        public static Money Zero => new Money(0, Currency.USD);
        
        // Shared implementation used by multiple contexts
    }
}

// ============================================
// CONTEXT MAP RELATIONSHIPS
// ============================================

/// <summary>
/// Context Map describes relationships between bounded contexts:
/// 
/// 1. PARTNERSHIP: Sales ←→ Billing
///    - Teams coordinate closely
///    - Share some concepts through shared kernel
///    - Mutual dependency, evolve together
/// 
/// 2. CUSTOMER-SUPPLIER: Sales → Shipping
///    - Sales is upstream (customer)
///    - Shipping is downstream (supplier)
///    - Sales defines the contract
///    - Shipping conforms to Sales needs
/// 
/// 3. ANTI-CORRUPTION LAYER: Sales → Support
///    - Sales protects itself from Support changes
///    - Adapter translates Support concepts to Sales
///    - Sales remains independent
/// 
/// 4. PUBLISHED LANGUAGE: All Contexts → Integration Events
///    - Well-defined event schema
///    - Each context translates to/from published language
///    - No direct coupling between contexts
/// 
/// 5. SEPARATE WAYS: Sales and HumanResources
///    - No integration
///    - Completely independent
///    - No shared concepts
/// </summary>
public static class ContextMap { }
```

## Context Relationship Patterns

```csharp
// Pattern 1: Customer-Supplier (Upstream-Downstream)
namespace Sales.Domain  // Upstream
{
    public interface ISalesCustomerExport
    {
        Task<CustomerSummary> GetCustomerSummaryAsync(CustomerId id);
    }
    
    public record CustomerSummary(
        string CustomerId,
        string Name,
        string Email,
        decimal CreditLimit);
}

namespace Shipping.Integration  // Downstream
{
    // Downstream conforms to upstream contract
    public class SalesCustomerAdapter
    {
        private readonly ISalesCustomerExport _sales;
        
        public async Task<ShippingRecipient> GetRecipientFromSales(string customerId)
        {
            var summary = await _sales.GetCustomerSummaryAsync(
                CustomerId.Parse(customerId));
            
            // Translate Sales concept to Shipping concept
            return ShippingRecipient.Create(
                RecipientId.Parse(summary.CustomerId),
                RecipientName.Create(summary.Name).Value,
                Address.Default);
        }
    }
}

// Pattern 2: Conformist (Downstream accepts upstream model)
namespace Billing.Integration
{
    // Billing accepts Sales model directly (conformist relationship)
    public class SalesBillingAdapter
    {
        public BillingAccount CreateFromSalesCustomer(
            Sales.Domain.Customer salesCustomer)
        {
            // Directly use Sales concepts, becoming dependent
            return BillingAccount.Create(
                AccountId.Parse(salesCustomer.Id.Value.ToString()),
                AccountName.Create(salesCustomer.Name.Value).Value,
                PaymentMethod.Default,
                Money.Zero);
        }
    }
}
```

## Why It's a Problem

1. **Tight coupling**: Changes in one context break others
2. **Confused models**: Same term means different things
3. **God objects**: Single model serves multiple contexts
4. **Evolution blocked**: Can't change without affecting others

## Symptoms

- Shared domain models across all contexts
- God objects with properties from multiple domains
- Changes in one area breaking unrelated areas
- Same term meaning different things in different places
- Difficulty understanding which team owns which model

## Benefits

- **Independence**: Contexts evolve separately
- **Clarity**: Each context has its own model
- **Flexibility**: Change one context without affecting others
- **Explicit integration**: Clear translation points
- **Team autonomy**: Teams own their contexts

## See Also

- [Bounded Contexts](./bounded-contexts.md) — defining context boundaries
- [Anti-Corruption Layer](./anti-corruption-layer.md) — protecting domain boundaries
- [Ubiquitous Language](./ubiquitous-language.md) — language within context
- [Domain Events](./domain-events.md) — integration through events
- [DTO vs Domain Boundary](./dto-domain-boundary.md) — separating external models
