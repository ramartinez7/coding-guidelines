# Use Cases / Application Services (Orchestrating Domain Logic)

> Business workflows scattered across controllers and services make the system's capabilities unclear—encapsulate each use case as a distinct class that orchestrates domain logic.

## Problem

When business workflows are implemented directly in controllers or scattered across multiple service classes, it becomes difficult to understand what the system does. Testing requires mocking infrastructure, and business rules get duplicated across entry points (HTTP, CLI, background jobs).

## Example

### ❌ Before (Logic in Controller)

```csharp
[ApiController]
[Route("api/users")]
public class UsersController : ControllerBase
{
    private readonly IUserRepository _userRepository;
    private readonly IEmailService _emailService;
    private readonly ILogger<UsersController> _logger;
    
    [HttpPost("register")]
    public async Task<IActionResult> Register(RegisterUserRequest request)
    {
        // Validation logic in controller
        if (string.IsNullOrEmpty(request.Email))
            return BadRequest("Email is required");
        
        if (request.Password.Length < 8)
            return BadRequest("Password must be at least 8 characters");
        
        // Check if user exists
        var existingUser = await _userRepository.GetByEmailAsync(request.Email);
        if (existingUser != null)
            return Conflict("User already exists");
        
        // Create user
        var user = new User
        {
            Id = Guid.NewGuid(),
            Email = request.Email,
            PasswordHash = BCrypt.HashPassword(request.Password),
            IsEmailConfirmed = false,
            CreatedAt = DateTime.UtcNow
        };
        
        // Generate confirmation token
        var token = Guid.NewGuid().ToString();
        user.EmailConfirmationToken = token;
        user.EmailConfirmationTokenExpiresAt = DateTime.UtcNow.AddHours(24);
        
        // Save user
        await _userRepository.AddAsync(user);
        
        // Send email
        try
        {
            await _emailService.SendConfirmationEmailAsync(user.Email, token);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to send confirmation email");
            // Continue anyway - user is created
        }
        
        _logger.LogInformation("User {UserId} registered", user.Id);
        
        return CreatedAtAction(nameof(GetUser), new { id = user.Id }, new { UserId = user.Id });
    }
    
    // Same logic would need to be duplicated for CLI, background jobs, etc.
}
```

**Problems:**
- Business logic mixed with HTTP concerns
- Hard to test without HTTP context
- Can't reuse from CLI, background jobs, or other entry points
- Unclear what the system's capabilities are
- Difficult to add cross-cutting concerns (logging, transactions, authorization)

### ✅ After (Explicit Use Case)

```csharp
// ============================================
// APPLICATION LAYER: Use Case
// ============================================
namespace MyApp.Application.UseCases.RegisterUser
{
    /// <summary>
    /// Command represents the input for this use case.
    /// Pure data - no behavior, no dependencies.
    /// </summary>
    public sealed record RegisterUserCommand(
        Email Email,
        Password Password,
        string FirstName,
        string LastName);
    
    /// <summary>
    /// Use case orchestrates domain logic and infrastructure.
    /// Single responsibility: register a new user.
    /// Testable without HTTP, database, or external services.
    /// </summary>
    public sealed class RegisterUserUseCase
    {
        private readonly IUserRepository _userRepository;
        private readonly IEmailService _emailService;
        private readonly IUnitOfWork _unitOfWork;
        private readonly ILogger<RegisterUserUseCase> _logger;
        
        public RegisterUserUseCase(
            IUserRepository userRepository,
            IEmailService emailService,
            IUnitOfWork unitOfWork,
            ILogger<RegisterUserUseCase> logger)
        {
            _userRepository = userRepository;
            _emailService = emailService;
            _unitOfWork = unitOfWork;
            _logger = logger;
        }
        
        public async Task<Result<UserId, RegisterUserError>> ExecuteAsync(
            RegisterUserCommand command)
        {
            // Check if user already exists
            var existingUser = await _userRepository.GetByEmailAsync(command.Email);
            if (existingUser.IsSome)
                return Result<UserId, RegisterUserError>.Failure(
                    RegisterUserError.UserAlreadyExists(command.Email));
            
            // Create user (domain logic)
            var user = User.Register(
                command.Email,
                command.Password,
                command.FirstName,
                command.LastName);
            
            // Generate confirmation token (domain logic)
            var token = user.GenerateEmailConfirmationToken(
                expiresIn: TimeSpan.FromHours(24));
            
            // Persist user
            await _userRepository.AddAsync(user);
            await _unitOfWork.CommitAsync();
            
            // Send confirmation email (side effect)
            try
            {
                await _emailService.SendConfirmationEmailAsync(
                    user.Email,
                    token);
                
                _logger.LogInformation(
                    "User {UserId} registered successfully. Confirmation email sent to {Email}",
                    user.Id,
                    user.Email);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex,
                    "Failed to send confirmation email to {Email}. User {UserId} was created.",
                    user.Email,
                    user.Id);
                
                // User is still created - email failure is not critical
                // Could raise a domain event here for retry mechanism
            }
            
            return Result<UserId, RegisterUserError>.Success(user.Id);
        }
    }
    
    /// <summary>
    /// Use case specific errors.
    /// </summary>
    public abstract record RegisterUserError
    {
        public sealed record UserAlreadyExists(Email Email) : RegisterUserError;
        public sealed record InvalidEmail(string Reason) : RegisterUserError;
        public sealed record InvalidPassword(string Reason) : RegisterUserError;
    }
}

// ============================================
// WEB API: Thin Controller
// ============================================
namespace MyApp.WebApi.Controllers
{
    [ApiController]
    [Route("api/users")]
    public class UsersController : ControllerBase
    {
        private readonly RegisterUserUseCase _registerUserUseCase;
        
        public UsersController(RegisterUserUseCase registerUserUseCase)
        {
            _registerUserUseCase = registerUserUseCase;
        }
        
        /// <summary>
        /// Controller is thin - just translates HTTP to use case.
        /// </summary>
        [HttpPost("register")]
        public async Task<IActionResult> Register([FromBody] RegisterUserRequest request)
        {
            // Map HTTP request to command
            var emailResult = Email.Create(request.Email);
            var passwordResult = Password.Create(request.Password);
            
            if (emailResult.IsFailure)
                return BadRequest(new { Error = emailResult.Error });
            
            if (passwordResult.IsFailure)
                return BadRequest(new { Error = passwordResult.Error });
            
            var command = new RegisterUserCommand(
                emailResult.Value,
                passwordResult.Value,
                request.FirstName,
                request.LastName);
            
            // Execute use case
            var result = await _registerUserUseCase.ExecuteAsync(command);
            
            // Map result to HTTP response
            return result.Match(
                onSuccess: userId => CreatedAtAction(
                    nameof(GetUser),
                    new { id = userId.Value },
                    new RegisterUserResponse(userId.Value)),
                onFailure: error => error switch
                {
                    RegisterUserError.UserAlreadyExists e => 
                        Conflict(new { Error = $"User with email {e.Email} already exists" }),
                    RegisterUserError.InvalidEmail e => 
                        BadRequest(new { Error = e.Reason }),
                    RegisterUserError.InvalidPassword e => 
                        BadRequest(new { Error = e.Reason }),
                    _ => StatusCode(500, new { Error = "An unexpected error occurred" })
                });
        }
    }
    
    public sealed record RegisterUserRequest(
        string Email,
        string Password,
        string FirstName,
        string LastName);
    
    public sealed record RegisterUserResponse(Guid UserId);
}

// ============================================
// CLI: Can reuse the same use case
// ============================================
namespace MyApp.Cli.Commands
{
    public class RegisterUserCliCommand
    {
        private readonly RegisterUserUseCase _registerUserUseCase;
        
        public async Task ExecuteAsync(string email, string password, string firstName, string lastName)
        {
            var emailResult = Email.Create(email);
            var passwordResult = Password.Create(password);
            
            if (emailResult.IsFailure || passwordResult.IsFailure)
            {
                Console.WriteLine("Invalid input");
                return;
            }
            
            var command = new RegisterUserCommand(
                emailResult.Value,
                passwordResult.Value,
                firstName,
                lastName);
            
            // Same use case, different entry point
            var result = await _registerUserUseCase.ExecuteAsync(command);
            
            result.Match(
                onSuccess: userId => Console.WriteLine($"User {userId} registered successfully"),
                onFailure: error => Console.WriteLine($"Registration failed: {error}"));
        }
    }
}
```

## Query vs Command Use Cases

### Command Use Cases (Write Operations)

```csharp
/// <summary>
/// Command use case: Modifies system state.
/// Returns success/failure, not data.
/// </summary>
public sealed class CancelOrderUseCase
{
    private readonly IOrderRepository _orderRepository;
    private readonly IPaymentGateway _paymentGateway;
    private readonly IEventDispatcher _eventDispatcher;
    
    public async Task<Result<Unit, CancelOrderError>> ExecuteAsync(
        CancelOrderCommand command)
    {
        // Load aggregate
        var orderResult = await _orderRepository.GetByIdAsync(command.OrderId);
        
        return await orderResult.Match(
            onSome: async order =>
            {
                // Execute domain logic
                var cancelResult = order.Cancel(command.Reason);
                if (cancelResult.IsFailure)
                    return Result<Unit, CancelOrderError>.Failure(
                        CancelOrderError.CannotCancel(cancelResult.Error));
                
                // Refund payment if needed
                if (order.WasPaid)
                {
                    var refundResult = await _paymentGateway.RefundAsync(order.PaymentId);
                    if (refundResult.IsFailure)
                        return Result<Unit, CancelOrderError>.Failure(
                            CancelOrderError.RefundFailed(refundResult.Error));
                }
                
                // Persist changes
                await _orderRepository.SaveAsync(order);
                
                // Publish event
                await _eventDispatcher.DispatchAsync(
                    new OrderCancelledEvent(order.Id, command.Reason));
                
                return Result<Unit, CancelOrderError>.Success(Unit.Value);
            },
            onNone: () => Task.FromResult(
                Result<Unit, CancelOrderError>.Failure(
                    CancelOrderError.OrderNotFound(command.OrderId))));
    }
}
```

### Query Use Cases (Read Operations)

```csharp
/// <summary>
/// Query use case: Reads data without modifying state.
/// Returns data directly, not wrapped in Result.
/// Can bypass domain and go straight to database for performance.
/// </summary>
public sealed class GetOrderDetailsQueryHandler
{
    private readonly IOrderQueryRepository _queryRepository;
    
    public async Task<Option<OrderDetailsDto>> HandleAsync(GetOrderDetailsQuery query)
    {
        // Query repository can return DTOs directly - no domain object needed
        var dto = await _queryRepository.GetOrderDetailsAsync(query.OrderId);
        
        return dto == null
            ? Option<OrderDetailsDto>.None
            : Option<OrderDetailsDto>.Some(dto);
    }
}

public sealed record GetOrderDetailsQuery(OrderId OrderId);

public sealed record OrderDetailsDto(
    Guid OrderId,
    Guid CustomerId,
    string CustomerName,
    List<OrderItemDto> Items,
    decimal Total,
    string Status,
    DateTimeOffset PlacedAt);
```

## Testing Use Cases

```csharp
public class RegisterUserUseCaseTests
{
    [Fact]
    public async Task ExecuteAsync_WithNewUser_CreatesUserAndSendsEmail()
    {
        // Arrange - No infrastructure mocking needed
        var userRepository = new InMemoryUserRepository();
        var emailService = new FakeEmailService();
        var unitOfWork = new FakeUnitOfWork();
        var logger = new NullLogger<RegisterUserUseCase>();
        
        var useCase = new RegisterUserUseCase(
            userRepository,
            emailService,
            unitOfWork,
            logger);
        
        var command = new RegisterUserCommand(
            Email.Create("test@example.com").Value,
            Password.Create("SecurePass123!").Value,
            "John",
            "Doe");
        
        // Act
        var result = await useCase.ExecuteAsync(command);
        
        // Assert
        Assert.True(result.IsSuccess);
        
        var savedUser = await userRepository.GetByEmailAsync(command.Email);
        Assert.True(savedUser.IsSome);
        
        Assert.Single(emailService.SentEmails);
        Assert.Equal(command.Email.Value, emailService.SentEmails[0].To);
    }
    
    [Fact]
    public async Task ExecuteAsync_WithExistingEmail_ReturnsError()
    {
        // Arrange
        var userRepository = new InMemoryUserRepository();
        var existingUser = User.Register(
            Email.Create("test@example.com").Value,
            Password.Create("SecurePass123!").Value,
            "Existing",
            "User");
        await userRepository.AddAsync(existingUser);
        
        var useCase = new RegisterUserUseCase(
            userRepository,
            new FakeEmailService(),
            new FakeUnitOfWork(),
            new NullLogger<RegisterUserUseCase>());
        
        var command = new RegisterUserCommand(
            Email.Create("test@example.com").Value,
            Password.Create("DifferentPass123!").Value,
            "John",
            "Doe");
        
        // Act
        var result = await useCase.ExecuteAsync(command);
        
        // Assert
        Assert.True(result.IsFailure);
        Assert.IsType<RegisterUserError.UserAlreadyExists>(result.Error);
    }
}
```

## Cross-Cutting Concerns with Decorators

```csharp
/// <summary>
/// Decorator adds transaction management to any use case.
/// </summary>
public sealed class TransactionalUseCaseDecorator<TCommand, TResult, TError> : IUseCase<TCommand, TResult, TError>
{
    private readonly IUseCase<TCommand, TResult, TError> _inner;
    private readonly IUnitOfWork _unitOfWork;
    
    public async Task<Result<TResult, TError>> ExecuteAsync(TCommand command)
    {
        using var transaction = await _unitOfWork.BeginTransactionAsync();
        
        try
        {
            var result = await _inner.ExecuteAsync(command);
            
            if (result.IsSuccess)
                await transaction.CommitAsync();
            else
                await transaction.RollbackAsync();
            
            return result;
        }
        catch
        {
            await transaction.RollbackAsync();
            throw;
        }
    }
}

/// <summary>
/// Decorator adds logging to any use case.
/// </summary>
public sealed class LoggingUseCaseDecorator<TCommand, TResult, TError> : IUseCase<TCommand, TResult, TError>
{
    private readonly IUseCase<TCommand, TResult, TError> _inner;
    private readonly ILogger _logger;
    
    public async Task<Result<TResult, TError>> ExecuteAsync(TCommand command)
    {
        var stopwatch = Stopwatch.StartNew();
        _logger.LogInformation("Executing {UseCase} with command: {@Command}",
            _inner.GetType().Name,
            command);
        
        try
        {
            var result = await _inner.ExecuteAsync(command);
            
            stopwatch.Stop();
            _logger.LogInformation("Executed {UseCase} in {ElapsedMs}ms. Success: {Success}",
                _inner.GetType().Name,
                stopwatch.ElapsedMilliseconds,
                result.IsSuccess);
            
            return result;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error executing {UseCase}", _inner.GetType().Name);
            throw;
        }
    }
}

/// <summary>
/// Decorator adds authorization to any use case.
/// </summary>
public sealed class AuthorizingUseCaseDecorator<TCommand, TResult, TError> : IUseCase<TCommand, TResult, TError>
{
    private readonly IUseCase<TCommand, TResult, TError> _inner;
    private readonly IAuthorizationService _authService;
    private readonly ICurrentUser _currentUser;
    
    public async Task<Result<TResult, TError>> ExecuteAsync(TCommand command)
    {
        var authResult = await _authService.AuthorizeAsync(
            _currentUser,
            command,
            _inner.GetType().Name);
        
        if (!authResult.Succeeded)
            throw new UnauthorizedAccessException("User not authorized for this operation");
        
        return await _inner.ExecuteAsync(command);
    }
}

// Register decorators in DI
services.Decorate<IUseCase<RegisterUserCommand, UserId, RegisterUserError>>(
    (inner, provider) => new LoggingUseCaseDecorator<RegisterUserCommand, UserId, RegisterUserError>(
        inner,
        provider.GetRequiredService<ILogger>()));
```

## Why It's a Problem

1. **Scattered logic**: Business workflows duplicated across controllers, CLI, background jobs
2. **Hard to test**: Testing requires HTTP context, database, external services
3. **Unclear capabilities**: What can this system do? Need to read all controllers
4. **No reusability**: Can't invoke business logic outside HTTP context
5. **Cross-cutting concerns**: Logging, transactions, authorization scattered everywhere

## Symptoms

- Controllers with hundreds of lines of logic
- Same business workflow implemented multiple times
- Tests that require HTTP mocking
- Difficulty adding new entry points (gRPC, CLI, message handlers)
- Inconsistent error handling across controllers

## Benefits

- **Clear capabilities**: One use case = one system capability
- **Testable**: Test business logic without infrastructure
- **Reusable**: Same use case from HTTP, CLI, background jobs, gRPC
- **Cross-cutting concerns**: Add logging, transactions, authorization via decorators
- **Maintainable**: Each use case is self-contained and focused

## Naming Conventions

Use case names should describe the **user goal** or **business capability**:

### ✅ Good Names (Business-Focused)
- `RegisterUserUseCase`
- `PlaceOrderUseCase`
- `CancelSubscriptionUseCase`
- `ApproveLeaveRequestUseCase`
- `TransferFundsUseCase`

### ❌ Bad Names (Technical-Focused)
- `CreateUserService`
- `OrderManager`
- `SubscriptionHandler`
- `DataProcessor`

## See Also

- [Clean Architecture](./clean-architecture.md) — overall architecture pattern
- [Domain Services](./domain-services.md) — domain-level services vs application services
- [Domain Events](./domain-events.md) — decoupling side effects
- [Repository Pattern](./repository-pattern.md) — data access abstraction
- [SOLID Principles](./solid-principles.md) — Single Responsibility Principle
- [Dependency Injection](./dependency-injection.md) — wiring up use cases
