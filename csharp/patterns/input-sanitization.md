# Input Sanitization (Trusted Types)

> Validating and sanitizing input at multiple points in the codebase—use trusted types to validate once at the boundary.

## Problem

When untrusted input flows through the system, developers must remember to sanitize it before use. This leads to either over-sanitization (validating repeatedly) or under-sanitization (forgetting to validate), both of which cause bugs. There's no compile-time proof that dangerous data has been made safe.

## Example

### ❌ Before

```csharp
public class UserController
{
    public IActionResult UpdateProfile(string username, string bio)
    {
        // Is this input sanitized? We don't know.
        var user = _userService.GetUser(username);
        _userService.UpdateBio(user, bio);
        return Ok();
    }
}

public class UserService
{
    public void UpdateBio(User user, string bio)
    {
        // Should I sanitize here? Was it already done?
        // Sanitizing again is safe but wasteful...
        var sanitized = SanitizeHtml(bio);
        user.Bio = sanitized;
        _repository.Save(user);
    }
    
    public void SendEmail(string email, string body)
    {
        // Forgot to sanitize! 💥 XSS vulnerability
        _emailService.Send(email, body);
    }
}
```

**Problems:**
- No way to know if input is safe
- Duplicate sanitization wastes CPU
- Forgotten sanitization creates vulnerabilities
- Every function must defensively sanitize

### ✅ After

```csharp
// Marker interface for validated inputs
public interface ITrustedInput { }

/// <summary>
/// HTML content that has been sanitized and is safe to render.
/// Cannot be constructed from raw strings—must go through sanitization.
/// </summary>
public sealed record SanitizedHtml(string Value) : ITrustedInput
{
    // Private constructor prevents direct instantiation
    private SanitizedHtml(string value) : this() => Value = value;
    
    public static SanitizedHtml Sanitize(string rawHtml)
    {
        // Use a library like HtmlSanitizer
        var sanitizer = new HtmlSanitizer();
        var clean = sanitizer.Sanitize(rawHtml);
        return new SanitizedHtml(clean);
    }
    
    public static SanitizedHtml Empty => new(string.Empty);
    
    // Implicit conversion for convenience (one-way only)
    public static implicit operator string(SanitizedHtml html) => html.Value;
}

/// <summary>
/// SQL parameter that has been properly escaped.
/// </summary>
public sealed record SqlParameter(string Value) : ITrustedInput
{
    private SqlParameter(string value) : this() => Value = value;
    
    public static SqlParameter Create(string raw)
    {
        // Escape SQL special characters
        var escaped = raw
            .Replace("'", "''")
            .Replace(";", "");
        return new SqlParameter(escaped);
    }
}

/// <summary>
/// Email address that has been validated.
/// </summary>
public sealed record ValidatedEmail(string Value) : ITrustedInput
{
    private ValidatedEmail(string value) : this() => Value = value;
    
    public static Result<ValidatedEmail, string> Create(string email)
    {
        if (string.IsNullOrWhiteSpace(email))
            return Result<ValidatedEmail, string>.Failure("Email cannot be empty");
            
        if (!email.Contains('@'))
            return Result<ValidatedEmail, string>.Failure("Email must contain @");
            
        // More sophisticated validation...
        var regex = new Regex(@"^[^@\s]+@[^@\s]+\.[^@\s]+$");
        if (!regex.IsMatch(email))
            return Result<ValidatedEmail, string>.Failure("Invalid email format");
            
        return Result<ValidatedEmail, string>.Success(new ValidatedEmail(email));
    }
}

public class UserController
{
    public IActionResult UpdateProfile(string username, string bio)
    {
        // Sanitize at the boundary
        var sanitizedBio = SanitizedHtml.Sanitize(bio);
        
        var user = _userService.GetUser(username);
        _userService.UpdateBio(user, sanitizedBio);
        return Ok();
    }
}

public class UserService
{
    // Type signature proves input is safe
    public void UpdateBio(User user, SanitizedHtml bio)
    {
        // No sanitization needed—type guarantees it's safe
        user.Bio = bio;  // Implicit conversion to string
        _repository.Save(user);
    }
    
    // Impossible to call with untrusted input
    public Result<Unit, string> SendEmail(ValidatedEmail email, string body)
    {
        // Type system enforces validation happened
        _emailService.Send(email.Value, body);
        return Result<Unit, string>.Success(Unit.Value);
    }
}

// Usage
var emailResult = ValidatedEmail.Create(userInput);
emailResult.Match(
    onSuccess: email => _userService.SendEmail(email, body),
    onFailure: error => Console.WriteLine($"Invalid email: {error}")
);
```

## Advanced Patterns

### Layered Sanitization

Different contexts require different sanitization:

```csharp
// Text safe for HTML display
public sealed record HtmlSafeText(string Value) : ITrustedInput
{
    public static HtmlSafeText Encode(string raw)
    {
        var encoded = System.Net.WebUtility.HtmlEncode(raw);
        return new HtmlSafeText(encoded);
    }
}

// Text safe for JavaScript context
public sealed record JsSafeText(string Value) : ITrustedInput
{
    public static JsSafeText Encode(string raw)
    {
        var encoded = System.Text.Json.JsonSerializer.Serialize(raw);
        return new JsSafeText(encoded);
    }
}

// Text safe for URL parameter
public sealed record UrlSafeText(string Value) : ITrustedInput
{
    public static UrlSafeText Encode(string raw)
    {
        var encoded = System.Net.WebUtility.UrlEncode(raw);
        return new UrlSafeText(encoded);
    }
}

// Rendering in different contexts
public string RenderUserProfile(User user)
{
    var htmlSafe = HtmlSafeText.Encode(user.Bio);
    var jsSafe = JsSafeText.Encode(user.Bio);
    var urlSafe = UrlSafeText.Encode(user.Username);
    
    return $@"
        <div class='bio'>{htmlSafe}</div>
        <script>var bio = {jsSafe};</script>
        <a href='/user?name={urlSafe}'>Profile</a>
    ";
}
```

### Combining with Result Types

```csharp
public sealed record ValidatedUsername(string Value) : ITrustedInput
{
    private ValidatedUsername(string value) : this() => Value = value;
    
    public static Result<ValidatedUsername, ValidationError> Create(string raw)
    {
        if (string.IsNullOrWhiteSpace(raw))
            return Result<ValidatedUsername, ValidationError>.Failure(
                new ValidationError("Username", "Cannot be empty"));
                
        if (raw.Length < 3)
            return Result<ValidatedUsername, ValidationError>.Failure(
                new ValidationError("Username", "Must be at least 3 characters"));
                
        if (raw.Length > 20)
            return Result<ValidatedUsername, ValidationError>.Failure(
                new ValidationError("Username", "Cannot exceed 20 characters"));
                
        if (!Regex.IsMatch(raw, "^[a-zA-Z0-9_]+$"))
            return Result<ValidatedUsername, ValidationError>.Failure(
                new ValidationError("Username", "Can only contain letters, numbers, and underscores"));
                
        return Result<ValidatedUsername, ValidationError>.Success(new ValidatedUsername(raw));
    }
}

public sealed record ValidationError(string Field, string Message);

// API usage
public IActionResult CreateUser(CreateUserDto dto)
{
    var usernameResult = ValidatedUsername.Create(dto.Username);
    var emailResult = ValidatedEmail.Create(dto.Email);
    
    // Combine multiple validations
    return (usernameResult, emailResult) switch
    {
        (Success: true, Success: true) => 
            _userService.CreateUser(usernameResult.Value, emailResult.Value),
            
        _ => BadRequest(new 
        { 
            errors = new[] { usernameResult.Error, emailResult.Error }
                .Where(e => e != null)
        })
    };
}
```

## Why It's a Problem

1. **Scattered validation**: Each function must remember to sanitize, leading to duplication or gaps
2. **No proof of safety**: Looking at a `string` parameter, you can't tell if it's been sanitized
3. **Over-sanitization**: Multiple layers sanitize the same input, wasting CPU
4. **Under-sanitization**: Forgotten validation creates XSS, SQL injection, or other vulnerabilities
5. **Testing complexity**: Must test both "sanitized" and "not sanitized" paths

## Symptoms

- `SanitizeInput()` or `ValidateEmail()` calls scattered throughout the codebase
- Comments like `// Must be sanitized before use`
- Security bugs found in penetration testing
- Defensive validation: "just in case it wasn't validated upstream"
- Regex patterns duplicated across multiple files

## Benefits

- **Compile-time enforcement**: Cannot pass raw strings where trusted types are required
- **Single sanitization point**: Validate once at the boundary
- **Self-documenting**: `UpdateBio(SanitizedHtml bio)` makes requirements explicit
- **No forgotten validation**: Type system prevents unsafe usage
- **Testable**: Unit test the sanitization logic in isolation

## Trade-offs

- **More types**: Each sanitization strategy needs its own type
- **Boundary ceremony**: Must convert at API boundaries
- **Cannot "unsanitize"**: Once wrapped, the raw input is inaccessible (by design)

## Integration with ASP.NET Core

```csharp
// Model binder for automatic conversion
public class SanitizedHtmlBinder : IModelBinder
{
    public Task BindModelAsync(ModelBindingContext bindingContext)
    {
        var value = bindingContext.ValueProvider.GetValue(bindingContext.ModelName).FirstValue;
        
        if (value == null)
        {
            bindingContext.Result = ModelBindingResult.Success(null);
            return Task.CompletedTask;
        }
        
        var sanitized = SanitizedHtml.Sanitize(value);
        bindingContext.Result = ModelBindingResult.Success(sanitized);
        return Task.CompletedTask;
    }
}

// Register in Startup
services.AddControllers(options =>
{
    options.ModelBinderProviders.Insert(0, new SanitizedHtmlBinderProvider());
});

// Now controller actions automatically sanitize
public IActionResult UpdateBio([FromBody] UpdateBioRequest request)
{
    // request.Bio is already SanitizedHtml
    _userService.UpdateBio(request.Bio);
    return Ok();
}

public record UpdateBioRequest(SanitizedHtml Bio);
```

## See Also

- [Capability Security](./capability-security.md) — authorization tokens
- [Honest Functions](./honest-functions.md) — making requirements explicit
- [Static Factory Methods](./static-factory-methods.md) — controlled construction
- [DTO vs. Domain Boundary](./dto-domain-boundary.md) — boundary enforcement
