# Boundary Enforcement (DTO vs. Domain)

> Using the same class for API serialization and domain logic—separate types and use mappers as the barrier.

## Problem

When the same class serves both as a JSON DTO and a domain entity, you get:
- **Over-posting vulnerabilities**: Attackers can set fields they shouldn't (like `IsAdmin`)
- **Validation pollution**: Domain invariants mixed with serialization attributes
- **Coupling**: API changes break domain logic, domain changes break API contracts
- **Impossible states**: DTOs allow invalid combinations that domain types should prevent

## Example

### ❌ Before

```csharp
// One class does everything—API, persistence, domain logic
public class User
{
    [JsonPropertyName("id")]
    public Guid Id { get; set; }
    
    [Required]
    [JsonPropertyName("email")]
    public string Email { get; set; }
    
    [JsonPropertyName("name")]
    public string Name { get; set; }
    
    [JsonPropertyName("role")]
    public string Role { get; set; }  // Can attacker POST this?
    
    [JsonIgnore]
    public string PasswordHash { get; set; }
    
    [JsonPropertyName("isAdmin")]
    public bool IsAdmin { get; set; }  // Over-posting vulnerability!
    
    // Domain logic mixed with serialization concerns
    public void ChangeEmail(string newEmail)
    {
        if (!newEmail.Contains("@"))
            throw new ValidationException("Invalid email");
        Email = newEmail;
    }
}

// Controller directly binds to domain type
[HttpPost]
public IActionResult CreateUser([FromBody] User user)
{
    // Attacker POSTs: { "email": "attacker@evil.com", "isAdmin": true }
    // IsAdmin gets bound! 💥
    _repository.Save(user);
    return Ok(user);  // Might expose PasswordHash if JsonIgnore fails
}

[HttpPut("{id}")]
public IActionResult UpdateUser(Guid id, [FromBody] User user)
{
    // What fields can they update? ALL of them!
    var existing = _repository.Find(id);
    existing.Email = user.Email;
    existing.Name = user.Name;
    existing.Role = user.Role;      // Should they be able to change this?
    existing.IsAdmin = user.IsAdmin; // Definitely not!
    _repository.Save(existing);
    return Ok(existing);
}
```

### ✅ After

```csharp
// ═══════════════════════════════════════════════════════════════
// API Layer: DTOs for serialization only
// ═══════════════════════════════════════════════════════════════

// Request DTO: only fields the client CAN send
public sealed record CreateUserRequest(
    [property: Required] string Email,
    [property: Required] string Name,
    [property: Required] string Password
);

// Request DTO: only fields the client CAN update
public sealed record UpdateUserRequest(
    string? Name,
    string? Email
);

// Response DTO: only fields the client SHOULD see
public sealed record UserResponse(
    Guid Id,
    string Email,
    string Name,
    string Role,
    DateTime CreatedAt
);
// Note: no IsAdmin, no PasswordHash

// ═══════════════════════════════════════════════════════════════
// Domain Layer: types with invariants and behavior
// ═══════════════════════════════════════════════════════════════

public sealed class User
{
    public UserId Id { get; }
    public Email Email { get; private set; }
    public UserName Name { get; private set; }
    public Role Role { get; private set; }
    public PasswordHash PasswordHash { get; private set; }
    public DateTime CreatedAt { get; }
    
    private User(UserId id, Email email, UserName name, PasswordHash hash)
    {
        Id = id;
        Email = email;
        Name = name;
        Role = Role.User;  // Default, not settable from outside
        PasswordHash = hash;
        CreatedAt = DateTime.UtcNow;
    }
    
    public static Result<User, CreateUserError> Create(
        Email email, 
        UserName name, 
        Password password)
    {
        // Domain validation
        if (email.IsBanned())
            return Result.Failure(new BannedEmailDomain(email));
        
        var hash = PasswordHash.Create(password);
        return Result.Success(new User(UserId.New(), email, name, hash));
    }
    
    public void ChangeName(UserName newName) => Name = newName;
    
    public Result<Unit, ChangeEmailError> ChangeEmail(Email newEmail)
    {
        if (newEmail == Email)
            return Result.Success(Unit.Value);
        
        // Domain rules for email change
        Email = newEmail;
        return Result.Success(Unit.Value);
    }
    
    // Role changes require authorization—not a simple setter
    public void PromoteTo(Role newRole, Capability<PromoteUser> capability)
    {
        Role = newRole;
    }
}

// ═══════════════════════════════════════════════════════════════
// Mapper: the barrier between layers
// ═══════════════════════════════════════════════════════════════

public static class UserMapper
{
    public static UserResponse ToResponse(User user) => new(
        Id: user.Id.Value,
        Email: user.Email.Value,
        Name: user.Name.Value,
        Role: user.Role.ToString(),
        CreatedAt: user.CreatedAt
    );
    
    // No ToEntity—domain objects are created through their own methods
}

// ═══════════════════════════════════════════════════════════════
// Controller: uses DTOs, delegates to domain
// ═══════════════════════════════════════════════════════════════

[HttpPost]
public IActionResult CreateUser([FromBody] CreateUserRequest request)
{
    // Parse primitives into domain types (validation happens here)
    var emailResult = Email.Create(request.Email);
    var nameResult = UserName.Create(request.Name);
    var passwordResult = Password.Create(request.Password);
    
    // Combine results
    return (emailResult, nameResult, passwordResult) switch
    {
        ({ IsSuccess: true }, { IsSuccess: true }, { IsSuccess: true }) =>
            CreateUserInternal(emailResult.Value, nameResult.Value, passwordResult.Value),
        _ => BadRequest(/* collect errors */)
    };
}

private IActionResult CreateUserInternal(Email email, UserName name, Password password)
{
    return User.Create(email, name, password).Match(
        onSuccess: user =>
        {
            _repository.Save(user);
            return Ok(UserMapper.ToResponse(user));  // Response DTO, not entity
        },
        onFailure: error => BadRequest(error)
    );
}

[HttpPut("{id}")]
public IActionResult UpdateUser(Guid id, [FromBody] UpdateUserRequest request)
{
    var user = _repository.Find(new UserId(id));
    if (user is null)
        return NotFound();
    
    // Only update what's in the DTO, using domain methods
    if (request.Name is not null)
    {
        var nameResult = UserName.Create(request.Name);
        if (nameResult.IsFailure)
            return BadRequest(nameResult.Error);
        user.ChangeName(nameResult.Value);
    }
    
    if (request.Email is not null)
    {
        var emailResult = Email.Create(request.Email);
        if (emailResult.IsFailure)
            return BadRequest(emailResult.Error);
        
        var changeResult = user.ChangeEmail(emailResult.Value);
        if (changeResult.IsFailure)
            return BadRequest(changeResult.Error);
    }
    
    // Note: no way to change Role or IsAdmin through this endpoint!
    
    _repository.Save(user);
    return Ok(UserMapper.ToResponse(user));
}
```

## The Three Layers

```
┌─────────────────────────────────────────────────────────────────┐
│  API Layer (DTOs)                                               │
│  • JSON serialization attributes                                │
│  • Input validation (Required, StringLength, etc.)              │
│  • Shape matches wire format                                    │
│  • No behavior, just data                                       │
├─────────────────────────────────────────────────────────────────┤
│  Mapper / Translator                                            │
│  • Converts DTO → Domain types (with validation)               │
│  • Converts Domain → Response DTO (with filtering)             │
│  • Single point of translation                                  │
├─────────────────────────────────────────────────────────────────┤
│  Domain Layer (Entities, Value Objects)                         │
│  • Business invariants                                          │
│  • Behavior and rules                                           │
│  • No serialization concerns                                    │
│  • Private/internal constructors                                │
└─────────────────────────────────────────────────────────────────┘
```

## Over-Posting Prevention

**Over-posting** occurs when an attacker sends fields you didn't intend to bind:

```json
// You expected:
{ "name": "Alice", "email": "alice@example.com" }

// Attacker sends:
{ "name": "Alice", "email": "alice@example.com", "isAdmin": true, "role": "SuperAdmin" }
```

With separate DTOs, extra fields are simply ignored—they don't exist on the DTO type.

```csharp
// UpdateUserRequest only has Name and Email
// isAdmin and role fields are silently dropped
public sealed record UpdateUserRequest(string? Name, string? Email);
```

## Request vs Response DTOs

Don't reuse the same DTO for both:

```csharp
// ❌ Same DTO for input and output
public record UserDto(Guid Id, string Email, string Name, DateTime CreatedAt);

// Problem: clients can't send Id or CreatedAt on create, but receive them on read

// ✅ Separate DTOs for each operation
public record CreateUserRequest(string Email, string Name, string Password);
public record UpdateUserRequest(string? Name, string? Email);
public record UserResponse(Guid Id, string Email, string Name, DateTime CreatedAt);
```

## Why It's a Problem

- **Security**: Over-posting can escalate privileges or corrupt data
- **Coupling**: API changes force domain changes and vice versa
- **Leaky abstractions**: Internal fields (PasswordHash) can leak to clients
- **Validation confusion**: Is this annotation for JSON or domain rules?
- **Testing difficulty**: Can't test domain logic without serialization

## Symptoms

- `[JsonIgnore]` scattered through domain entities
- The same class has `[Required]` (API) and business validation methods
- Security reviews flag "mass assignment" vulnerabilities
- API changes require domain model changes
- Sensitive data accidentally exposed in API responses

## Guidelines

1. **Separate assemblies**: Put DTOs in API assembly, domain in Domain assembly
2. **One DTO per operation**: `CreateXRequest`, `UpdateXRequest`, `XResponse`
3. **Explicit mapping**: No `AutoMapper` magic—write the mapping code
4. **Domain creates domain**: DTOs don't construct entities; entities have factory methods
5. **Response filtering**: Only include fields the client should see

## Benefits

- **Security by default**: Over-posting is impossible—fields don't exist
- **Independent evolution**: Change API format without touching domain
- **Clear contracts**: DTOs document exactly what the API accepts/returns
- **Testable layers**: Test domain without HTTP, test API without business logic
- **Explicit mapping**: No surprises about what gets included

## See Also

- [Primitive Obsession](./primitive-obsession.md) — domain uses value objects, DTOs use primitives
- [Static Factory Methods](./static-factory-methods.md) — domain objects control their creation
- [Assemblies vs. Namespaces](./assembly-isolation.md) — enforce separation with assemblies
- [Ghost States](./ghost-states.md) — DTOs can have invalid combinations, domain types can't
