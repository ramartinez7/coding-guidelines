# Secure Defaults (Fail-Safe Design)

> Types that allow insecure configurations through optional parameters—use secure defaults and make insecure choices explicit and difficult.

## Problem

When security features are opt-in, developers forget to enable them or use insecure defaults for convenience. This creates vulnerabilities. Secure-by-default design makes insecurity require explicit effort.

## Example

### ❌ Before

```csharp
// Insecure defaults—easy to get wrong
public class HttpClientBuilder
{
    public TimeSpan Timeout { get; set; } = TimeSpan.FromMinutes(100); // Too long!
    public bool ValidateCertificates { get; set; } = false;  // 💥 Insecure default!
    public bool AllowAutoRedirect { get; set; } = true;  // 💥 Dangerous!
    public int MaxRedirects { get; set; } = int.MaxValue;  // 💥 Infinite redirects!
    
    public HttpClient Build()
    {
        var handler = new HttpClientHandler
        {
            ServerCertificateCustomValidationCallback =
                ValidateCertificates
                    ? null
                    : (_, _, _, _) => true,  // Accepts any certificate!
            AllowAutoRedirect = AllowAutoRedirect,
            MaxAutomaticRedirections = MaxRedirects
        };
        
        return new HttpClient(handler) { Timeout = Timeout };
    }
}

// Easy to create insecure client—just forgot to set flags
var client = new HttpClientBuilder().Build();  // 💥 All insecure defaults!

// Or with optional parameters—insecure by default
public HttpClient CreateClient(
    string baseUrl,
    bool validateCerts = false,  // 💥 Dangerous default
    bool followRedirects = true,  // 💥 Dangerous default
    int maxRedirects = 50)  // 💥 Still too high
{
    // ...
}
```

**Problems:**
- Security features are opt-in
- Easy to forget to enable security
- Insecure choices are the path of least resistance
- No compile-time enforcement of security
- Copy-paste code inherits insecurity

### ✅ After (Secure Defaults)

```csharp
/// <summary>
/// HTTP client configuration with secure defaults.
/// Insecure options require explicit types.
/// </summary>
public sealed record HttpClientConfig
{
    // Secure defaults—private setters prevent external modification
    public TimeSpan Timeout { get; private init; } = TimeSpan.FromSeconds(30);
    public CertificateValidation CertificateValidation { get; private init; } =
        CertificateValidation.Strict();
    public RedirectPolicy RedirectPolicy { get; private init; } =
        RedirectPolicy.NoRedirects();
    
    private HttpClientConfig() { }
    
    /// <summary>
    /// Create secure HTTP client configuration.
    /// All defaults are secure.
    /// </summary>
    public static HttpClientConfig Secure() => new();
    
    /// <summary>
    /// Override timeout—still secure by default.
    /// </summary>
    public HttpClientConfig WithTimeout(TimeSpan timeout)
    {
        if (timeout > TimeSpan.FromMinutes(5))
            throw new ArgumentException(
                "Timeout should not exceed 5 minutes for security");
        
        return this with { Timeout = timeout };
    }
    
    /// <summary>
    /// Allow redirects—requires explicit policy.
    /// Cannot accidentally enable infinite redirects.
    /// </summary>
    public HttpClientConfig WithRedirects(RedirectPolicy policy)
    {
        return this with { RedirectPolicy = policy };
    }
    
    /// <summary>
    /// Dangerous: Disable certificate validation.
    /// Requires explicit insecure type to make danger obvious.
    /// </summary>
    public HttpClientConfig WithCertificateValidation(CertificateValidation validation)
    {
        return this with { CertificateValidation = validation };
    }
}

/// <summary>
/// Certificate validation policy—strict by default.
/// </summary>
public abstract record CertificateValidation
{
    private CertificateValidation() { }
    
    /// <summary>
    /// Strict validation—checks all certificates.
    /// </summary>
    public sealed record Strict : CertificateValidation
    {
        public static Strict Instance { get; } = new();
    }
    
    /// <summary>
    /// Accept specific certificate thumbprints only.
    /// </summary>
    public sealed record PinnedCertificates(
        IReadOnlySet<string> AllowedThumbprints) : CertificateValidation;
    
    /// <summary>
    /// DANGEROUS: Accept all certificates.
    /// Name makes danger obvious.
    /// </summary>
    public sealed record AcceptAllCertificatesDangerous(
        string Justification) : CertificateValidation;
    
    public static CertificateValidation Strict() => Strict.Instance;
    
    public static CertificateValidation Pinned(params string[] thumbprints) =>
        new PinnedCertificates(thumbprints.ToHashSet());
    
    /// <summary>
    /// Must provide justification to create insecure validation.
    /// </summary>
    public static CertificateValidation AcceptAll(string justification)
    {
        if (string.IsNullOrWhiteSpace(justification))
            throw new ArgumentException(
                "Must provide justification for accepting all certificates");
        
        return new AcceptAllCertificatesDangerous(justification);
    }
}

/// <summary>
/// Redirect policy—no redirects by default.
/// </summary>
public abstract record RedirectPolicy
{
    private RedirectPolicy() { }
    
    /// <summary>
    /// No redirects allowed.
    /// </summary>
    public sealed record NoRedirects : RedirectPolicy
    {
        public static NoRedirects Instance { get; } = new();
    }
    
    /// <summary>
    /// Allow limited redirects to same host only.
    /// </summary>
    public sealed record SameHost(int MaxRedirects) : RedirectPolicy
    {
        public SameHost() : this(3) { }
    }
    
    /// <summary>
    /// Allow limited redirects to any host.
    /// Requires explicit max—cannot be infinite.
    /// </summary>
    public sealed record AnyHost : RedirectPolicy
    {
        public int MaxRedirects { get; }
        
        public AnyHost() : this(3) { }
        
        public AnyHost(int maxRedirects)
        {
            if (maxRedirects < 1 || maxRedirects > 10)
                throw new ArgumentException(
                    "Max redirects must be between 1 and 10");
            
            MaxRedirects = maxRedirects;
        }
    }
    
    public static RedirectPolicy NoRedirects() => NoRedirects.Instance;
    
    public static RedirectPolicy SameHost(int maxRedirects = 3) =>
        new SameHost(maxRedirects);
    
    public static RedirectPolicy AnyHost(int maxRedirects = 3) =>
        new AnyHost(maxRedirects);
}

// Usage: Secure by default
var secureClient = HttpClientConfig.Secure()
    .WithTimeout(TimeSpan.FromSeconds(45))
    .Build();

// Allowing redirects requires explicit policy
var clientWithRedirects = HttpClientConfig.Secure()
    .WithRedirects(RedirectPolicy.SameHost(maxRedirects: 5))
    .Build();

// Disabling cert validation requires justification and scary name
var dangerousClient = HttpClientConfig.Secure()
    .WithCertificateValidation(
        CertificateValidation.AcceptAll("Development environment only"))
    .Build();
```

## Database Connection with Secure Defaults

```csharp
/// <summary>
/// Database connection configuration with secure defaults.
/// </summary>
public sealed record DatabaseConfig
{
    public string ConnectionString { get; private init; }
    public ConnectionSecurity Security { get; private init; } =
        ConnectionSecurity.Encrypted();
    public int MaxPoolSize { get; private init; } = 20;
    public TimeSpan ConnectionTimeout { get; private init; } =
        TimeSpan.FromSeconds(15);
    public bool MultipleActiveResultSets { get; private init; } = false;
    
    private DatabaseConfig(string connectionString)
    {
        ConnectionString = connectionString;
    }
    
    public static DatabaseConfig Create(string server, string database) =>
        new($"Server={server};Database={database}");
    
    public DatabaseConfig WithSecurity(ConnectionSecurity security) =>
        this with { Security = security };
    
    public DatabaseConfig WithMaxPoolSize(int size)
    {
        if (size < 1 || size > 100)
            throw new ArgumentException("Pool size must be between 1 and 100");
        
        return this with { MaxPoolSize = size };
    }
}

/// <summary>
/// Connection security—encrypted by default.
/// </summary>
public abstract record ConnectionSecurity
{
    private ConnectionSecurity() { }
    
    /// <summary>
    /// Encrypted connection with certificate validation.
    /// </summary>
    public sealed record Encrypted(bool ValidateCertificate) : ConnectionSecurity
    {
        public Encrypted() : this(true) { }
    }
    
    /// <summary>
    /// DANGEROUS: Unencrypted connection.
    /// </summary>
    public sealed record UnencryptedDangerous(string Justification) : ConnectionSecurity;
    
    public static ConnectionSecurity Encrypted() => new Encrypted();
    
    public static ConnectionSecurity Unencrypted(string justification) =>
        new UnencryptedDangerous(justification);
}

// Usage: Secure by default
var dbConfig = DatabaseConfig.Create("localhost", "mydb");

// Unencrypted requires justification
var devDbConfig = DatabaseConfig.Create("localhost", "mydb")
    .WithSecurity(ConnectionSecurity.Unencrypted(
        "Local development only—no sensitive data"));
```

## API Keys with Secure Defaults

```csharp
/// <summary>
/// API key configuration with secure defaults.
/// </summary>
public sealed record ApiKeyConfig
{
    public ApiKeyPermissions Permissions { get; private init; } =
        ApiKeyPermissions.ReadOnly();
    public Option<TimeSpan> ExpiresIn { get; private init; } =
        Option<TimeSpan>.FromValue(TimeSpan.FromDays(30));  // Expires by default
    public RateLimitPolicy RateLimit { get; private init; } =
        RateLimitPolicy.Conservative();
    
    private ApiKeyConfig() { }
    
    public static ApiKeyConfig Create() => new();
    
    public ApiKeyConfig WithPermissions(ApiKeyPermissions permissions) =>
        this with { Permissions = permissions };
    
    public ApiKeyConfig WithExpiration(TimeSpan expiresIn)
    {
        if (expiresIn > TimeSpan.FromDays(365))
            throw new ArgumentException(
                "API keys should not be valid for more than 1 year");
        
        return this with { ExpiresIn = Option<TimeSpan>.FromValue(expiresIn) };
    }
    
    /// <summary>
    /// DANGEROUS: Create non-expiring API key.
    /// Requires explicit justification.
    /// </summary>
    public ApiKeyConfig WithNoExpirationDangerous(string justification)
    {
        if (string.IsNullOrWhiteSpace(justification))
            throw new ArgumentException("Must justify non-expiring key");
        
        return this with { ExpiresIn = Option<TimeSpan>.None };
    }
}

/// <summary>
/// API key permissions—read-only by default.
/// </summary>
public abstract record ApiKeyPermissions
{
    private ApiKeyPermissions() { }
    
    public sealed record ReadOnly : ApiKeyPermissions
    {
        public static ReadOnly Instance { get; } = new();
    }
    
    public sealed record ReadWrite(IReadOnlySet<string> AllowedEndpoints)
        : ApiKeyPermissions;
    
    public sealed record AdminAccess(string ApprovedBy, DateTime ApprovedAt)
        : ApiKeyPermissions;
    
    public static ApiKeyPermissions ReadOnly() => ReadOnly.Instance;
    
    public static ApiKeyPermissions ReadWrite(params string[] endpoints) =>
        new ReadWrite(endpoints.ToHashSet());
    
    public static ApiKeyPermissions Admin(string approvedBy) =>
        new AdminAccess(approvedBy, DateTime.UtcNow);
}

/// <summary>
/// Rate limiting—conservative by default.
/// </summary>
public sealed record RateLimitPolicy(int RequestsPerMinute)
{
    public RateLimitPolicy() : this(60) { }
    
    public static RateLimitPolicy Conservative() => new(60);
    
    public static RateLimitPolicy Moderate() => new(300);
    
    public static RateLimitPolicy Generous() => new(1000);
}

// Usage: Secure by default—read-only, expires in 30 days
var apiKey = ApiKeyConfig.Create();

// Write access requires explicit endpoints
var writeKey = ApiKeyConfig.Create()
    .WithPermissions(ApiKeyPermissions.ReadWrite("/api/orders", "/api/products"));

// Non-expiring key requires justification
var serviceKey = ApiKeyConfig.Create()
    .WithNoExpirationDangerous("Service-to-service authentication");
```

## Password Policy with Secure Defaults

```csharp
/// <summary>
/// Password policy with secure defaults.
/// </summary>
public sealed record PasswordPolicy
{
    public int MinLength { get; private init; } = 12;
    public bool RequireUppercase { get; private init; } = true;
    public bool RequireLowercase { get; private init; } = true;
    public bool RequireDigit { get; private init; } = true;
    public bool RequireSpecialChar { get; private init; } = true;
    public int MaxPasswordAge { get; private init; } = 90; // days
    public int PasswordHistorySize { get; private init; } = 5;
    
    private PasswordPolicy() { }
    
    /// <summary>
    /// Secure default policy—meets NIST guidelines.
    /// </summary>
    public static PasswordPolicy Secure() => new();
    
    /// <summary>
    /// Relax specific requirements—still enforces others.
    /// </summary>
    public PasswordPolicy WithMinLength(int minLength)
    {
        if (minLength < 8)
            throw new ArgumentException(
                "Minimum length must be at least 8 characters");
        
        return this with { MinLength = minLength };
    }
    
    /// <summary>
    /// DANGEROUS: Allow weak passwords.
    /// Requires explicit justification.
    /// </summary>
    public static PasswordPolicy WeakDangerous(string justification)
    {
        if (string.IsNullOrWhiteSpace(justification))
            throw new ArgumentException("Must justify weak password policy");
        
        return new PasswordPolicy
        {
            MinLength = 6,
            RequireUppercase = false,
            RequireLowercase = true,
            RequireDigit = false,
            RequireSpecialChar = false
        };
    }
}

// Usage: Secure by default
var policy = PasswordPolicy.Secure();

// Customize while maintaining security
var customPolicy = PasswordPolicy.Secure()
    .WithMinLength(16);

// Weak policy requires justification
var devPolicy = PasswordPolicy.WeakDangerous(
    "Development environment—not accessible to real users");
```

## CORS Policy with Secure Defaults

```csharp
/// <summary>
/// CORS policy with secure defaults (deny all).
/// </summary>
public sealed record CorsPolicy
{
    public AllowedOrigins Origins { get; private init; } =
        AllowedOrigins.None();
    public IReadOnlySet<string> AllowedMethods { get; private init; } =
        new HashSet<string> { "GET" };
    public bool AllowCredentials { get; private init; } = false;
    public Option<TimeSpan> MaxAge { get; private init; } =
        Option<TimeSpan>.FromValue(TimeSpan.FromMinutes(5));
    
    private CorsPolicy() { }
    
    public static CorsPolicy Create() => new();
    
    public CorsPolicy WithOrigins(AllowedOrigins origins) =>
        this with { Origins = origins };
    
    public CorsPolicy WithMethods(params string[] methods) =>
        this with { AllowedMethods = methods.ToHashSet() };
}

/// <summary>
/// Allowed origins—none by default.
/// </summary>
public abstract record AllowedOrigins
{
    private AllowedOrigins() { }
    
    public sealed record None : AllowedOrigins
    {
        public static None Instance { get; } = new();
    }
    
    public sealed record Specific(IReadOnlySet<Uri> Origins) : AllowedOrigins;
    
    /// <summary>
    /// DANGEROUS: Allow all origins.
    /// </summary>
    public sealed record AllOriginsDangerous(string Justification) : AllowedOrigins;
    
    public static AllowedOrigins None() => None.Instance;
    
    public static AllowedOrigins Specific(params string[] origins) =>
        new Specific(origins.Select(o => new Uri(o)).ToHashSet());
    
    public static AllowedOrigins All(string justification) =>
        new AllOriginsDangerous(justification);
}

// Usage: Deny by default
var corsPolicy = CorsPolicy.Create();

// Allow specific origins
var allowedPolicy = CorsPolicy.Create()
    .WithOrigins(AllowedOrigins.Specific(
        "https://example.com",
        "https://app.example.com"))
    .WithMethods("GET", "POST");

// Allow all requires justification
var openPolicy = CorsPolicy.Create()
    .WithOrigins(AllowedOrigins.All("Public API—no sensitive data"));
```

## Key Principles

### 1. Secure by Default

```csharp
// ✅ Security features enabled by default
public static HttpClientConfig Secure() => new()
{
    ValidateCertificates = true,
    AllowRedirects = false,
    Timeout = TimeSpan.FromSeconds(30)
};
```

### 2. Explicit Insecurity

```csharp
// Make insecure choices obvious with:
// - Scary names: "Dangerous", "Insecure", "Unsafe"
// - Required justification parameter
// - Compiler warnings or attributes
public static CertificateValidation AcceptAllCertificatesDangerous(
    string justification);
```

### 3. Fail-Safe Defaults

```csharp
// If something goes wrong, fail securely
public sealed record SessionConfig
{
    // Timeout by default—never infinite
    public TimeSpan Timeout { get; init; } = TimeSpan.FromMinutes(15);
    
    // Secure cookies by default
    public bool SecureCookies { get; init; } = true;
    public bool HttpOnly { get; init; } = true;
}
```

### 4. Principle of Least Privilege

```csharp
// Minimal permissions by default
public static ApiKeyPermissions ReadOnly() => new ReadOnly();

// More permissions require explicit request
public static ApiKeyPermissions ReadWrite(params string[] endpoints);
```

## Why It's a Problem

1. **Insecure defaults**: Easy to accidentally use dangerous settings
2. **Opt-in security**: Developers forget to enable protections
3. **Copy-paste vulnerability**: Insecure examples get copied
4. **False sense of security**: Assuming defaults are safe
5. **Hidden dangers**: No indication that default is insecure

## Symptoms

- Security vulnerabilities from default configurations
- Comments like "// TODO: enable in production"
- Boolean flags for security features (should be secure always)
- Optional security parameters (should be required)
- Configuration errors in production

## Benefits

- **Secure by default**: Cannot accidentally be insecure
- **Explicit insecurity**: Dangerous choices are obvious
- **Pit of success**: Easy path is the secure path
- **Code review**: Insecure choices visible in code
- **Fail-safe**: Defaults protect even when misconfigured

## Trade-offs

- **Initial friction**: Secure defaults may be more restrictive
- **Development overhead**: May need different configs for dev/prod
- **Learning curve**: Developers must understand security implications
- **Verbosity**: More code to create insecure configurations

## See Also

- [Capability Security](./capability-security.md) — authorization by default
- [Principle of Least Privilege](./principle-of-least-privilege.md) — minimal permissions
- [Secret Types](./secret-types.md) — secure secret handling by default
- [Authentication Context](./authentication-context.md) — authenticated by default
