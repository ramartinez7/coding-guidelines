# Domain Events (Decoupling Side Effects)

> "God methods" with core action plus 10 side effects—return domain events as types to decouple and announce changes.

## Problem

It's common to see methods that do a core action (Register User) and then perform 10 side effects (Send Email, Update Cache, Notify Sales, etc.). This violates the Single Responsibility Principle, creates tight coupling, and makes the core logic impossible to test without mocking the entire world.

## Example

### ❌ Before

```csharp
public class UserService
{
    private readonly IUserRepository _userRepo;
    private readonly IEmailService _emailService;
    private readonly ICrmService _crmService;
    private readonly ICacheService _cacheService;
    private readonly IAnalyticsService _analyticsService;
    private readonly ISlackNotifier _slackNotifier;

    public UserService(
        IUserRepository userRepo,
        IEmailService emailService,
        ICrmService crmService,
        ICacheService cacheService,
        IAnalyticsService analyticsService,
        ISlackNotifier slackNotifier)
    {
        _userRepo = userRepo;
        _emailService = emailService;
        _crmService = crmService;
        _cacheService = cacheService;
        _analyticsService = analyticsService;
        _slackNotifier = slackNotifier;
    }

    public async Task<User> RegisterAsync(string email, string name)
    {
        // Core action
        var user = new User(email, name);
        await _userRepo.AddAsync(user);

        // Side effect 1: Send welcome email
        await _emailService.SendWelcomeEmailAsync(user.Email, user.Name);

        // Side effect 2: Create CRM contact
        await _crmService.CreateContactAsync(user.Email, user.Name);

        // Side effect 3: Invalidate cache
        await _cacheService.InvalidateAsync("user-count");

        // Side effect 4: Track analytics
        await _analyticsService.TrackAsync("user_registered", new { user.Id });

        // Side effect 5: Notify sales team
        await _slackNotifier.NotifyChannelAsync("#sales", $"New user: {user.Email}");

        return user;
    }
}

// Testing nightmare: must mock 6 dependencies
[Fact]
public async Task RegisterAsync_CreatesUser()
{
    var userRepo = new Mock<IUserRepository>();
    var emailService = new Mock<IEmailService>();
    var crmService = new Mock<ICrmService>();
    var cacheService = new Mock<ICacheService>();
    var analyticsService = new Mock<IAnalyticsService>();
    var slackNotifier = new Mock<ISlackNotifier>();

    var service = new UserService(
        userRepo.Object,
        emailService.Object,
        crmService.Object,
        cacheService.Object,
        analyticsService.Object,
        slackNotifier.Object);

    // Just want to test user creation, but must satisfy all dependencies
    await service.RegisterAsync("test@example.com", "Test User");

    userRepo.Verify(r => r.AddAsync(It.IsAny<User>()), Times.Once);
}
```

**Problems:**
- `UserService` knows about Email, CRM, Cache, Analytics, and Slack
- Adding a new side effect requires modifying `RegisterAsync`
- Testing the core logic requires mocking 6 dependencies
- Side effects execute synchronously, blocking the response
- If Slack is down, user registration fails

### ✅ After

```csharp
// Domain event as a type
public sealed record UserRegisteredEvent(
    UserId UserId,
    EmailAddress Email,
    string Name,
    DateTimeOffset RegisteredAt) : IDomainEvent;

// Marker interface for domain events
public interface IDomainEvent
{
    DateTimeOffset OccurredAt => DateTimeOffset.UtcNow;
}

// Service does ONE thing: creates user and announces it
public class UserService
{
    private readonly IUserRepository _userRepo;

    public UserService(IUserRepository userRepo)
    {
        _userRepo = userRepo;
    }

    public async Task<(User User, UserRegisteredEvent Event)> RegisterAsync(
        EmailAddress email, 
        string name)
    {
        var user = new User(email, name);
        await _userRepo.AddAsync(user);

        var domainEvent = new UserRegisteredEvent(
            user.Id,
            user.Email,
            user.Name,
            DateTimeOffset.UtcNow);

        return (user, domainEvent);
    }
}

// Side effects are separate handlers
public sealed class SendWelcomeEmailHandler : IEventHandler<UserRegisteredEvent>
{
    private readonly IEmailService _emailService;

    public SendWelcomeEmailHandler(IEmailService emailService)
        => _emailService = emailService;

    public async Task HandleAsync(UserRegisteredEvent evt)
    {
        await _emailService.SendWelcomeEmailAsync(evt.Email, evt.Name);
    }
}

public sealed class CreateCrmContactHandler : IEventHandler<UserRegisteredEvent>
{
    private readonly ICrmService _crmService;

    public CreateCrmContactHandler(ICrmService crmService)
        => _crmService = crmService;

    public async Task HandleAsync(UserRegisteredEvent evt)
    {
        await _crmService.CreateContactAsync(evt.Email, evt.Name);
    }
}

public sealed class NotifySalesHandler : IEventHandler<UserRegisteredEvent>
{
    private readonly ISlackNotifier _slackNotifier;

    public NotifySalesHandler(ISlackNotifier slackNotifier)
        => _slackNotifier = slackNotifier;

    public async Task HandleAsync(UserRegisteredEvent evt)
    {
        await _slackNotifier.NotifyChannelAsync("#sales", $"New user: {evt.Email}");
    }
}
```

**Testing is trivial:**

```csharp
[Fact]
public async Task RegisterAsync_ReturnsUserRegisteredEvent()
{
    var userRepo = new Mock<IUserRepository>();
    var service = new UserService(userRepo.Object);

    var (user, evt) = await service.RegisterAsync(
        EmailAddress.Parse("test@example.com"),
        "Test User");

    Assert.Equal(user.Id, evt.UserId);
    Assert.Equal("test@example.com", evt.Email.Value);
}

// Each handler is tested in isolation
[Fact]
public async Task SendWelcomeEmailHandler_SendsEmail()
{
    var emailService = new Mock<IEmailService>();
    var handler = new SendWelcomeEmailHandler(emailService.Object);

    await handler.HandleAsync(new UserRegisteredEvent(
        UserId.New(),
        EmailAddress.Parse("test@example.com"),
        "Test User",
        DateTimeOffset.UtcNow));

    emailService.Verify(e => e.SendWelcomeEmailAsync(
        It.IsAny<EmailAddress>(), "Test User"), Times.Once);
}
```

## Event Dispatcher

```csharp
public interface IEventHandler<TEvent> where TEvent : IDomainEvent
{
    Task HandleAsync(TEvent evt, CancellationToken ct = default);
}

public interface IEventDispatcher
{
    Task DispatchAsync<TEvent>(TEvent evt, CancellationToken ct = default) 
        where TEvent : IDomainEvent;
}

public sealed class EventDispatcher : IEventDispatcher
{
    private readonly IServiceProvider _serviceProvider;

    public EventDispatcher(IServiceProvider serviceProvider)
        => _serviceProvider = serviceProvider;

    public async Task DispatchAsync<TEvent>(TEvent evt, CancellationToken ct = default)
        where TEvent : IDomainEvent
    {
        var handlers = _serviceProvider.GetServices<IEventHandler<TEvent>>();

        foreach (var handler in handlers)
        {
            await handler.HandleAsync(evt, ct);
        }
    }
}
```

**Registration:**

```csharp
builder.Services.AddScoped<IEventDispatcher, EventDispatcher>();
builder.Services.AddScoped<IEventHandler<UserRegisteredEvent>, SendWelcomeEmailHandler>();
builder.Services.AddScoped<IEventHandler<UserRegisteredEvent>, CreateCrmContactHandler>();
builder.Services.AddScoped<IEventHandler<UserRegisteredEvent>, NotifySalesHandler>();
```

## Why It's a Problem

1. **SRP violation**: `UserService` knows too much about Email, CRM, Slack, etc.

2. **Testing nightmare**: Core logic requires mocking every side effect dependency.

3. **Rigid design**: Adding a side effect means modifying the core service.

4. **Synchronous blocking**: All side effects must complete before returning.

5. **Cascading failures**: If Slack is down, user registration fails.

## Symptoms

- Services with 5+ constructor dependencies
- Methods longer than 50 lines due to side effects
- Tests that require extensive mocking setup
- Comments like `// TODO: move this to a background job`
- Changes to side effects requiring modification of core services

## Event Sourcing Lite

For entities with rich domain logic, accumulate events internally:

```csharp
public abstract class AggregateRoot
{
    private readonly List<IDomainEvent> _events = new();
    public IReadOnlyList<IDomainEvent> DomainEvents => _events;

    protected void RaiseEvent(IDomainEvent evt) => _events.Add(evt);
    public void ClearEvents() => _events.Clear();
}

public class User : AggregateRoot
{
    public UserId Id { get; }
    public EmailAddress Email { get; private set; }
    public string Name { get; private set; }

    private User(UserId id, EmailAddress email, string name)
    {
        Id = id;
        Email = email;
        Name = name;
    }

    public static User Register(EmailAddress email, string name)
    {
        var user = new User(UserId.New(), email, name);
        user.RaiseEvent(new UserRegisteredEvent(user.Id, email, name, DateTimeOffset.UtcNow));
        return user;
    }

    public void ChangeEmail(EmailAddress newEmail)
    {
        var oldEmail = Email;
        Email = newEmail;
        RaiseEvent(new UserEmailChangedEvent(Id, oldEmail, newEmail, DateTimeOffset.UtcNow));
    }
}

// Repository dispatches events after save
public class UserRepository : IUserRepository
{
    private readonly DbContext _db;
    private readonly IEventDispatcher _dispatcher;

    public async Task AddAsync(User user)
    {
        _db.Users.Add(user);
        await _db.SaveChangesAsync();

        foreach (var evt in user.DomainEvents)
        {
            await _dispatcher.DispatchAsync(evt);
        }
        user.ClearEvents();
    }
}
```

## Async Event Handling

For non-critical side effects, dispatch to a queue:

```csharp
public sealed class QueuedEventDispatcher : IEventDispatcher
{
    private readonly IMessageQueue _queue;

    public async Task DispatchAsync<TEvent>(TEvent evt, CancellationToken ct = default)
        where TEvent : IDomainEvent
    {
        await _queue.EnqueueAsync(evt, ct);
    }
}

// Background worker processes events
public class EventProcessorWorker : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        await foreach (var evt in _queue.DequeueAsync(ct))
        {
            var handlers = _serviceProvider.GetServices<IEventHandler<IDomainEvent>>();
            foreach (var handler in handlers)
            {
                await handler.HandleAsync(evt, ct);
            }
        }
    }
}
```

## Benefits

- **Single responsibility**: Services do one thing and announce it
- **Easy testing**: Core logic tested without mocking side effects
- **Open/Closed**: Add handlers without modifying core services
- **Async ready**: Events can be queued for background processing
- **Resilience**: Side effect failures don't break core operations
- **Discoverability**: Event types document what happens in the system

## See Also

- [Honest Functions](./honest-functions.md)
- [Specification Pattern (Logic as Types)](./specification-pattern.md)
- [Boundary Enforcement (DTO vs. Domain)](./dto-domain-boundary.md)
