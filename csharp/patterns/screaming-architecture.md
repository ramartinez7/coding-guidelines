# Screaming Architecture (Project Structure Reveals Intent)

> Generic folder structures like "Controllers," "Services," "Models" hide what the system does—organize by features so the architecture screams the business domain.

## Problem

Traditional layered folder structures organize code by technical role (Controllers, Services, Repositories, Models). When you open the project, you see technical infrastructure, not the business domain. Finding all code related to a feature requires navigating multiple folders across different layers.

## The Principle

> "A good architecture allows major decisions to be deferred. It allows you to defer the decision about which database, which framework, which delivery mechanism to use." — Robert C. Martin

When someone opens your codebase, **the architecture should scream what the application does, not what framework it uses.**

## Example

### ❌ Before (Technical Organization)

```
MyApp.Api/
├── Controllers/
│   ├── OrdersController.cs
│   ├── ProductsController.cs
│   ├── CustomersController.cs
│   ├── PaymentsController.cs
│   └── ShipmentsController.cs
├── Services/
│   ├── OrderService.cs
│   ├── ProductService.cs
│   ├── CustomerService.cs
│   ├── PaymentService.cs
│   └── ShipmentService.cs
├── Repositories/
│   ├── OrderRepository.cs
│   ├── ProductRepository.cs
│   ├── CustomerRepository.cs
│   ├── PaymentRepository.cs
│   └── ShipmentRepository.cs
├── Models/
│   ├── Order.cs
│   ├── Product.cs
│   ├── Customer.cs
│   ├── Payment.cs
│   └── Shipment.cs
└── DTOs/
    ├── OrderDto.cs
    ├── ProductDto.cs
    └── CustomerDto.cs
```

**Problems:**
- Can't tell what the system does by looking at folders
- Related code scattered across multiple folders
- Changes to a feature touch many folders
- Hard to see dependencies between features
- Accidental coupling between unrelated features

**Question: What does this application do?**
Answer: "It has controllers, services, and models." (Not helpful!)

### ✅ After (Feature Organization / Screaming Architecture)

#### Option 1: Vertical Slice Architecture

```
MyApp/
├── Features/
│   ├── Orders/
│   │   ├── PlaceOrder/
│   │   │   ├── PlaceOrderCommand.cs
│   │   │   ├── PlaceOrderUseCase.cs
│   │   │   ├── PlaceOrderController.cs
│   │   │   ├── PlaceOrderValidator.cs
│   │   │   └── PlaceOrderTests.cs
│   │   ├── CancelOrder/
│   │   │   ├── CancelOrderCommand.cs
│   │   │   ├── CancelOrderUseCase.cs
│   │   │   ├── CancelOrderController.cs
│   │   │   └── CancelOrderTests.cs
│   │   ├── GetOrderDetails/
│   │   │   ├── GetOrderDetailsQuery.cs
│   │   │   ├── GetOrderDetailsQueryHandler.cs
│   │   │   ├── GetOrderDetailsController.cs
│   │   │   └── OrderDetailsDto.cs
│   │   └── Domain/
│   │       ├── Order.cs
│   │       ├── OrderItem.cs
│   │       ├── OrderStatus.cs
│   │       └── IOrderRepository.cs
│   ├── Products/
│   │   ├── CreateProduct/
│   │   │   ├── CreateProductCommand.cs
│   │   │   ├── CreateProductUseCase.cs
│   │   │   └── CreateProductController.cs
│   │   ├── UpdateProductPrice/
│   │   │   ├── UpdateProductPriceCommand.cs
│   │   │   └── UpdateProductPriceUseCase.cs
│   │   └── Domain/
│   │       ├── Product.cs
│   │       ├── ProductId.cs
│   │       └── IProductRepository.cs
│   ├── Customers/
│   │   ├── RegisterCustomer/
│   │   │   ├── RegisterCustomerCommand.cs
│   │   │   ├── RegisterCustomerUseCase.cs
│   │   │   └── RegisterCustomerController.cs
│   │   ├── UpdateCustomerProfile/
│   │   │   └── ...
│   │   └── Domain/
│   │       ├── Customer.cs
│   │       └── CustomerRepository.cs
│   └── Payments/
│       ├── ProcessPayment/
│       │   ├── ProcessPaymentCommand.cs
│       │   ├── ProcessPaymentUseCase.cs
│       │   └── ProcessPaymentController.cs
│       └── Domain/
│           ├── Payment.cs
│           └── IPaymentGateway.cs
├── Infrastructure/
│   ├── Persistence/
│   │   ├── OrderRepository.cs
│   │   ├── ProductRepository.cs
│   │   └── MyDbContext.cs
│   └── ExternalServices/
│       ├── StripePaymentGateway.cs
│       └── SendGridEmailService.cs
└── Shared/
    ├── Result.cs
    ├── Option.cs
    └── IUnitOfWork.cs
```

**Question: What does this application do?**
Answer: "It manages orders, products, customers, and payments. I can see it can place orders, cancel orders, register customers, process payments..." (Much better!)

#### Option 2: Clean Architecture with Feature Folders

```
MyApp/
├── Domain/                              ← Core business logic
│   ├── Orders/
│   │   ├── Order.cs
│   │   ├── OrderItem.cs
│   │   ├── OrderStatus.cs
│   │   ├── IOrderRepository.cs
│   │   └── Events/
│   │       ├── OrderPlacedEvent.cs
│   │       └── OrderCancelledEvent.cs
│   ├── Products/
│   │   ├── Product.cs
│   │   ├── ProductId.cs
│   │   ├── IProductRepository.cs
│   │   └── ProductOutOfStockException.cs
│   ├── Customers/
│   │   ├── Customer.cs
│   │   ├── CustomerId.cs
│   │   └── ICustomerRepository.cs
│   └── SharedKernel/
│       ├── Money.cs
│       ├── Email.cs
│       └── Result.cs
│
├── Application/                         ← Use cases / Business workflows
│   ├── Orders/
│   │   ├── PlaceOrder/
│   │   │   ├── PlaceOrderCommand.cs
│   │   │   ├── PlaceOrderUseCase.cs
│   │   │   └── PlaceOrderValidator.cs
│   │   ├── CancelOrder/
│   │   │   ├── CancelOrderCommand.cs
│   │   │   └── CancelOrderUseCase.cs
│   │   └── GetOrderDetails/
│   │       ├── GetOrderDetailsQuery.cs
│   │       ├── GetOrderDetailsQueryHandler.cs
│   │       └── OrderDetailsDto.cs
│   ├── Products/
│   │   ├── CreateProduct/
│   │   │   └── ...
│   │   └── UpdateProductPrice/
│   │       └── ...
│   ├── Customers/
│   │   ├── RegisterCustomer/
│   │   │   └── ...
│   │   └── UpdateCustomerProfile/
│   │       └── ...
│   └── Common/
│       ├── IUseCase.cs
│       └── IQuery.cs
│
├── Infrastructure/                      ← External concerns
│   ├── Persistence/
│   │   ├── Orders/
│   │   │   ├── OrderRepository.cs
│   │   │   └── OrderConfiguration.cs
│   │   ├── Products/
│   │   │   ├── ProductRepository.cs
│   │   │   └── ProductConfiguration.cs
│   │   └── MyDbContext.cs
│   ├── ExternalServices/
│   │   ├── Payments/
│   │   │   └── StripePaymentGateway.cs
│   │   └── Email/
│   │       └── SendGridEmailService.cs
│   └── DependencyInjection.cs
│
└── WebApi/                              ← HTTP interface
    ├── Orders/
    │   ├── OrdersController.cs
    │   ├── PlaceOrderRequest.cs
    │   └── OrderResponse.cs
    ├── Products/
    │   └── ProductsController.cs
    ├── Customers/
    │   └── CustomersController.cs
    └── Program.cs
```

## Real-World Example: E-Commerce Platform

### ❌ Before (Framework-Centric)

```
ECommerce.Api/
├── Controllers/
│   ├── AuthController.cs
│   ├── OrdersController.cs
│   ├── CartController.cs
│   ├── ProductsController.cs
│   ├── ReviewsController.cs
│   └── WishlistController.cs
├── Services/
│   └── ... (20+ service classes)
├── Models/
│   └── ... (30+ model classes)
└── Data/
    └── ApplicationDbContext.cs
```

When you open this project, you see: "This is an ASP.NET API."

### ✅ After (Domain-Centric / Screaming Architecture)

```
ECommerce/
├── Catalog/                             ← Browsing and discovering products
│   ├── UseCases/
│   │   ├── SearchProducts/
│   │   ├── GetProductDetails/
│   │   └── BrowseCategories/
│   └── Domain/
│       ├── Product.cs
│       ├── Category.cs
│       └── ProductReview.cs
│
├── ShoppingCart/                        ← Selecting items for purchase
│   ├── UseCases/
│   │   ├── AddItemToCart/
│   │   ├── RemoveItemFromCart/
│   │   └── GetCart/
│   └── Domain/
│       ├── Cart.cs
│       └── CartItem.cs
│
├── Ordering/                            ← Placing and managing orders
│   ├── UseCases/
│   │   ├── PlaceOrder/
│   │   ├── CancelOrder/
│   │   ├── GetOrderHistory/
│   │   └── TrackShipment/
│   └── Domain/
│       ├── Order.cs
│       ├── OrderItem.cs
│       └── ShipmentTracking.cs
│
├── Identity/                            ← User authentication and authorization
│   ├── UseCases/
│   │   ├── RegisterUser/
│   │   ├── Login/
│   │   ├── ChangePassword/
│   │   └── ResetPassword/
│   └── Domain/
│       ├── User.cs
│       └── RefreshToken.cs
│
├── Reviews/                             ← Product reviews and ratings
│   ├── UseCases/
│   │   ├── SubmitReview/
│   │   ├── EditReview/
│   │   └── GetProductReviews/
│   └── Domain/
│       └── Review.cs
│
└── Wishlist/                            ← Saving products for later
    ├── UseCases/
    │   ├── AddToWishlist/
    │   ├── RemoveFromWishlist/
    │   └── GetWishlist/
    └── Domain/
        └── WishlistItem.cs
```

When you open this project, you see: "This is an e-commerce platform with catalog, shopping cart, ordering, user identity, reviews, and wishlist features."

## Benefits of Feature Organization

### 1. High Cohesion

```csharp
// All code related to "Place Order" is in one place
Features/Orders/PlaceOrder/
├── PlaceOrderCommand.cs          // Input
├── PlaceOrderUseCase.cs          // Logic
├── PlaceOrderController.cs       // HTTP interface
├── PlaceOrderValidator.cs        // Validation
├── PlaceOrderRequest.cs          // HTTP DTO
└── PlaceOrderTests.cs            // Tests

// vs scattered across:
// Controllers/OrdersController.cs
// Services/OrderService.cs
// Validators/PlaceOrderValidator.cs
// DTOs/PlaceOrderRequest.cs
// Tests/OrderServiceTests.cs
```

### 2. Clear Dependencies

```csharp
// Easy to see: PlaceOrder depends on ProcessPayment
Features/
├── Orders/
│   └── PlaceOrder/
│       └── PlaceOrderUseCase.cs
│           // uses: Features.Payments.ProcessPayment.IPaymentProcessor
└── Payments/
    └── ProcessPayment/
        └── IPaymentProcessor.cs

// vs hidden dependency:
// Services/OrderService.cs uses Services/PaymentService.cs
// Can't tell from folder structure
```

### 3. Team Ownership

```csharp
// Team Checkout owns everything in Features/Orders/
// Team Catalog owns everything in Features/Products/
// Clear boundaries, no stepping on each other's toes

Features/
├── Orders/         ← Team Checkout
├── Products/       ← Team Catalog
├── Customers/      ← Team Identity
└── Payments/       ← Team Payments
```

### 4. Safe Refactoring

```csharp
// Change PlaceOrder feature? All code is in one folder
// Delete CancelOrder feature? Delete one folder
// No hunting through Controllers, Services, Models, Repositories

// Before: Delete "Cancel Order" feature
Controllers/OrdersController.cs         // Delete CancelOrder action
Services/OrderService.cs                // Delete CancelOrder method
Repositories/OrderRepository.cs         // Maybe delete some methods?
Models/CancelOrderRequest.cs           // Delete DTO
Tests/OrderServiceTests.cs             // Delete tests
// Did I find everything? Not sure...

// After: Delete "Cancel Order" feature
Features/Orders/CancelOrder/           // Delete folder
// Done. Everything was in one place.
```

## Handling Shared Code

### Shared Kernel

```csharp
// Truly shared primitives used everywhere
Shared/
├── Result.cs
├── Option.cs
├── Money.cs
├── Email.cs
└── IUnitOfWork.cs
```

### Feature-to-Feature Dependencies

```csharp
// ✅ Explicit: Import from another feature's public interface
namespace Features.Orders.PlaceOrder
{
    using Features.Payments.ProcessPayment;  // Explicit dependency
    
    public class PlaceOrderUseCase
    {
        private readonly IPaymentProcessor _paymentProcessor;  // From Payments feature
    }
}

// ❌ Avoid: Each feature accessing other features' internals
// If Orders needs access to Customer domain model, consider:
// 1. Is Orders really a subdomain of Customer?
// 2. Should Orders only store CustomerId, not full Customer?
// 3. Should there be a SharedKernel with Customer types?
```

## When to Use Each Style

### Use Vertical Slice (Option 1) when:
- Building new applications (greenfield)
- Feature-focused teams
- Microservices (each feature could become a service)
- Rapid feature development

### Use Clean Architecture Folders (Option 2) when:
- Existing DDD-based codebase
- Want clear layer boundaries
- Multiple bounded contexts in one solution
- Large team with specialized roles

### Use Hybrid when:
- Large applications with multiple modules
- Some features share domain models
- Gradual migration from traditional architecture

```
MyApp/
├── Modules/
│   ├── Catalog/                    ← Bounded Context
│   │   ├── Domain/
│   │   ├── Application/
│   │   └── Infrastructure/
│   ├── ShoppingCart/               ← Bounded Context
│   │   ├── Domain/
│   │   ├── Application/
│   │   └── Infrastructure/
│   └── Ordering/                   ← Bounded Context
│       ├── Domain/
│       ├── Application/
│       └── Infrastructure/
└── Shared/
    └── SharedKernel/
```

## Migration Strategy

### Step 1: Identify Features

```csharp
// List all use cases / user stories
- Register User
- Login
- Reset Password
- Place Order
- Cancel Order
- Add Product to Cart
- Remove Product from Cart
// ...
```

### Step 2: Create Feature Folders

```csharp
// Create folders for each feature
Features/
├── Identity/
│   ├── RegisterUser/
│   ├── Login/
│   └── ResetPassword/
├── Orders/
│   ├── PlaceOrder/
│   └── CancelOrder/
└── ShoppingCart/
    ├── AddItemToCart/
    └── RemoveItemFromCart/
```

### Step 3: Move Code Feature-by-Feature

```csharp
// Pick one feature, move all related code
// Don't move everything at once

// Move PlaceOrder:
Controllers/OrdersController.cs#PlaceOrder()  → Features/Orders/PlaceOrder/PlaceOrderController.cs
Services/OrderService.cs#PlaceOrder()         → Features/Orders/PlaceOrder/PlaceOrderUseCase.cs
Models/PlaceOrderRequest.cs                   → Features/Orders/PlaceOrder/PlaceOrderRequest.cs
// etc.
```

### Step 4: Refactor Incrementally

```csharp
// After moving, refactor to improve structure
// Extract shared code to Shared/
// Break dependencies between features
// Add tests
```

## Why It's a Problem

1. **Hidden intent**: Can't tell what system does from folder structure
2. **Scattered code**: Related code spread across multiple folders
3. **Unclear dependencies**: Can't see which features depend on others
4. **Difficult refactoring**: Changing a feature touches many folders
5. **Team conflicts**: Multiple teams modifying same folders

## Symptoms

- Controllers with 20+ actions
- Services with 30+ methods
- Can't describe what the system does without reading all code
- Need to touch 5+ folders to add one feature
- Difficulty assigning features to teams

## Benefits

- **Clear intent**: Architecture reveals business domain
- **High cohesion**: Related code stays together
- **Team ownership**: Clear feature boundaries
- **Easy refactoring**: Change or delete entire features easily
- **Maintainability**: New developers understand system quickly

## Common Objections

### "But we need Controllers, Services, Repositories!"

You can still have those — just organized by feature:

```csharp
Features/Orders/PlaceOrder/
├── PlaceOrderController.cs     ← Controller
├── PlaceOrderUseCase.cs        ← Service
└── IOrderRepository.cs         ← Repository interface

Infrastructure/Persistence/Orders/
└── OrderRepository.cs          ← Repository implementation
```

### "Won't we duplicate code?"

Only if features truly need the same logic. More often, "duplication" is accidental coupling.

```csharp
// ❌ Shared "UserService" doing too much
Services/UserService.cs
- RegisterUser()
- Login()
- UpdateProfile()
- ChangePassword()
- DeleteAccount()

// ✅ Each feature owns its logic
Features/Identity/RegisterUser/RegisterUserUseCase.cs
Features/Identity/Login/LoginUseCase.cs
Features/Profile/UpdateProfile/UpdateProfileUseCase.cs
// Each can evolve independently
```

True shared logic goes in Shared/:

```csharp
Shared/
├── Email/
│   └── IEmailService.cs
└── Security/
    └── IPasswordHasher.cs
```

## See Also

- [Clean Architecture](./clean-architecture.md) — layer organization
- [Use Cases](./use-cases.md) — organizing by business capabilities
- [Bounded Contexts](./bounded-contexts.md) — strategic domain boundaries
- [Ubiquitous Language](./ubiquitous-language.md) — using domain terms in structure
- [SOLID Principles](./solid-principles.md) — Single Responsibility at folder level
