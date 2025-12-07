# CORS Configuration (Type-Safe Origin Validation)

> Using string-based CORS policies—use typed origin validation to prevent unauthorized cross-origin access at compile time.

## Problem

Cross-Origin Resource Sharing (CORS) policies are configured using strings, making them prone to errors. Wildcard origins (`*`) are used carelessly, origin validation is inconsistent, and there's no compile-time guarantee that CORS policies are correct or secure.

## Example

### ❌ Before

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddCors(options =>
    {
        options.AddPolicy("AllowAll", builder =>
        {
            builder.AllowAnyOrigin()  // 💥 Dangerous! Allows any site
                   .AllowAnyMethod()
                   .AllowAnyHeader();
        });
        
        // String-based origins—typos not caught
        options.AddPolicy("Production", builder =>
        {
            builder.WithOrigins("https://example.com", "https://www.exmaple.com")  // Typo!
                   .AllowAnyMethod()
                   .AllowAnyHeader();
        });
    });
}

// Different controllers might use different policies (inconsistent)
[EnableCors("AllowAll")]
public class PublicApiController : ControllerBase
{
    // Too permissive!
}
```

**Problems:**
- String-based origins prone to typos
- Wildcard origins (`*`) used carelessly
- No compile-time validation
- Inconsistent policies across controllers
- No type safety around origin validation

### ✅ After

```csharp
/// <summary>
/// Validated origin for CORS configuration.
/// Cannot be constructed from invalid origins.
/// </summary>
public readonly record struct Origin
{
    public Uri Value { get; }
    
    private Origin(Uri value) => Value = value;
    
    public static Result<Origin, string> Create(string origin)
    {
        if (string.IsNullOrWhiteSpace(origin))
            return Result<Origin, string>.Failure("Origin cannot be empty");
        
        if (!Uri.TryCreate(origin, UriKind.Absolute, out var uri))
            return Result<Origin, string>.Failure($"Invalid origin format: {origin}");
        
        if (uri.Scheme != "https" && uri.Scheme != "http")
            return Result<Origin, string>.Failure(
                $"Origin must use http or https scheme: {origin}");
        
        // Normalize: remove trailing slash, force lowercase
        var normalized = $"{uri.Scheme}://{uri.Host.ToLowerInvariant()}";
        if (!uri.IsDefaultPort)
            normalized += $":{uri.Port}";
        
        return Result<Origin, string>.Success(
            new Origin(new Uri(normalized)));
    }
    
    public static Origin Https(string host, int? port = null)
    {
        var uri = port.HasValue
            ? $"https://{host}:{port}"
            : $"https://{host}";
        
        var result = Create(uri);
        if (result.IsFailure)
            throw new ArgumentException($"Invalid host: {host}");
        
        return result.Value;
    }
    
    public static Origin Localhost(int port) =>
        Https("localhost", port);
    
    public bool Matches(string requestOrigin)
    {
        return Create(requestOrigin)
            .Match(
                onSuccess: origin => Uri.Compare(
                    origin.Value,
                    Value,
                    UriComponents.AbsoluteUri,
                    UriFormat.SafeUnescaped,
                    StringComparison.OrdinalIgnoreCase) == 0,
                onFailure: _ => false);
    }
    
    public override string ToString() => Value.ToString();
}

/// <summary>
/// CORS policy configuration with type-safe origins.
/// </summary>
public sealed record CorsPolicy
{
    public required string PolicyName { get; init; }
    public required OriginPolicy Origins { get; init; }
    public required IReadOnlySet<string> AllowedMethods { get; init; }
    public required IReadOnlySet<string> AllowedHeaders { get; init; }
    public required IReadOnlySet<string> ExposedHeaders { get; init; }
    public bool AllowCredentials { get; init; }
    public TimeSpan? MaxAge { get; init; }
    
    public void Apply(CorsPolicyBuilder builder)
    {
        Origins.Apply(builder);
        
        if (AllowedMethods.Count > 0)
            builder.WithMethods(AllowedMethods.ToArray());
        else
            builder.AllowAnyMethod();
        
        if (AllowedHeaders.Count > 0)
            builder.WithHeaders(AllowedHeaders.ToArray());
        else
            builder.AllowAnyHeader();
        
        if (ExposedHeaders.Count > 0)
            builder.WithExposedHeaders(ExposedHeaders.ToArray());
        
        if (AllowCredentials)
            builder.AllowCredentials();
        
        if (MaxAge.HasValue)
            builder.SetPreflightMaxAge(MaxAge.Value);
    }
    
    public static CorsPolicy DenyAll => new()
    {
        PolicyName = "DenyAll",
        Origins = OriginPolicy.None(),
        AllowedMethods = new HashSet<string>(),
        AllowedHeaders = new HashSet<string>(),
        ExposedHeaders = new HashSet<string>(),
        AllowCredentials = false
    };
    
    public static CorsPolicy AllowLocalhost(int port) => new()
    {
        PolicyName = "Localhost",
        Origins = OriginPolicy.Specific(Origin.Localhost(port)),
        AllowedMethods = new HashSet<string> { "GET", "POST", "PUT", "DELETE", "PATCH" },
        AllowedHeaders = new HashSet<string> { "Content-Type", "Authorization" },
        ExposedHeaders = new HashSet<string>(),
        AllowCredentials = true,
        MaxAge = TimeSpan.FromMinutes(10)
    };
}

/// <summary>
/// Origin validation policy (none, specific, pattern, any).
/// </summary>
public abstract record OriginPolicy
{
    private OriginPolicy() { }
    
    public sealed record None : OriginPolicy
    {
        public override void Apply(CorsPolicyBuilder builder)
        {
            // Don't call WithOrigins—effectively denies all
        }
    }
    
    public sealed record Specific(IReadOnlySet<Origin> AllowedOrigins) : OriginPolicy
    {
        public override void Apply(CorsPolicyBuilder builder)
        {
            builder.WithOrigins(AllowedOrigins.Select(o => o.ToString()).ToArray());
        }
    }
    
    public sealed record Pattern(Func<Origin, bool> Predicate) : OriginPolicy
    {
        public override void Apply(CorsPolicyBuilder builder)
        {
            builder.SetIsOriginAllowed(origin =>
                Origin.Create(origin)
                    .Match(
                        onSuccess: o => Predicate(o),
                        onFailure: _ => false));
        }
    }
    
    public sealed record Any : OriginPolicy
    {
        public override void Apply(CorsPolicyBuilder builder)
        {
            builder.AllowAnyOrigin();
        }
    }
    
    public abstract void Apply(CorsPolicyBuilder builder);
    
    public static OriginPolicy None() => new None();
    
    public static OriginPolicy Specific(params Origin[] origins) =>
        new Specific(origins.ToHashSet());
    
    public static OriginPolicy Pattern(Func<Origin, bool> predicate) =>
        new Pattern(predicate);
    
    public static OriginPolicy Any() => new Any();
}

/// <summary>
/// CORS policy registry with typed policies.
/// </summary>
public sealed class CorsConfiguration
{
    private readonly Dictionary<string, CorsPolicy> _policies = new();
    
    public CorsConfiguration AddPolicy(CorsPolicy policy)
    {
        _policies[policy.PolicyName] = policy;
        return this;
    }
    
    public void ConfigureCors(CorsOptions options)
    {
        foreach (var (name, policy) in _policies)
        {
            options.AddPolicy(name, builder => policy.Apply(builder));
        }
    }
    
    public static CorsConfiguration Default => new CorsConfiguration()
        .AddPolicy(CorsPolicy.DenyAll);
}

// Register in Startup
public void ConfigureServices(IServiceCollection services)
{
    var corsConfig = new CorsConfiguration()
        .AddPolicy(new CorsPolicy
        {
            PolicyName = "ProductionApi",
            Origins = OriginPolicy.Specific(
                Origin.Https("example.com"),
                Origin.Https("www.example.com")),
            AllowedMethods = new HashSet<string> { "GET", "POST", "PUT", "DELETE" },
            AllowedHeaders = new HashSet<string> { "Content-Type", "Authorization" },
            ExposedHeaders = new HashSet<string> { "X-Total-Count" },
            AllowCredentials = true,
            MaxAge = TimeSpan.FromHours(1)
        })
        .AddPolicy(new CorsPolicy
        {
            PolicyName = "PublicApi",
            Origins = OriginPolicy.Any(),
            AllowedMethods = new HashSet<string> { "GET" },
            AllowedHeaders = new HashSet<string> { "Content-Type" },
            ExposedHeaders = new HashSet<string>(),
            AllowCredentials = false
        });
    
    services.AddCors(corsConfig.ConfigureCors);
}
```

## Pattern-Based Origin Validation

```csharp
/// <summary>
/// Origin patterns for flexible validation.
/// </summary>
public static class OriginPatterns
{
    /// <summary>
    /// Allows origins from specific domain and all subdomains.
    /// </summary>
    public static Func<Origin, bool> SubdomainsOf(string domain)
    {
        return origin =>
        {
            var host = origin.Value.Host.ToLowerInvariant();
            return host == domain || host.EndsWith($".{domain}");
        };
    }
    
    /// <summary>
    /// Allows origins with specific port range on localhost.
    /// </summary>
    public static Func<Origin, bool> LocalhostPortRange(int minPort, int maxPort)
    {
        return origin =>
        {
            if (origin.Value.Host != "localhost")
                return false;
            
            var port = origin.Value.Port;
            return port >= minPort && port <= maxPort;
        };
    }
    
    /// <summary>
    /// Allows origins from preview/staging environments.
    /// </summary>
    public static Func<Origin, bool> PreviewEnvironments(string baseDomain)
    {
        return origin =>
        {
            var host = origin.Value.Host.ToLowerInvariant();
            // Matches: pr-123.example.com, staging-feature-x.example.com
            return Regex.IsMatch(host, $@"^(pr-\d+|staging-[\w-]+)\.{Regex.Escape(baseDomain)}$");
        };
    }
}

// Usage
var corsPolicy = new CorsPolicy
{
    PolicyName = "Development",
    Origins = OriginPolicy.Pattern(
        OriginPatterns.LocalhostPortRange(3000, 3999)),
    AllowedMethods = new HashSet<string> { "GET", "POST", "PUT", "DELETE" },
    AllowedHeaders = new HashSet<string> { "Content-Type", "Authorization" },
    ExposedHeaders = new HashSet<string>(),
    AllowCredentials = true
};

var stagingPolicy = new CorsPolicy
{
    PolicyName = "Staging",
    Origins = OriginPolicy.Pattern(
        OriginPatterns.SubdomainsOf("example.com")),
    AllowedMethods = new HashSet<string> { "GET", "POST" },
    AllowedHeaders = new HashSet<string> { "Content-Type" },
    ExposedHeaders = new HashSet<string>(),
    AllowCredentials = false
};
```

## Environment-Specific CORS

```csharp
/// <summary>
/// CORS configuration that adapts to environment.
/// </summary>
public sealed class EnvironmentAwareCorsConfiguration
{
    private readonly IWebHostEnvironment _environment;
    
    public EnvironmentAwareCorsConfiguration(IWebHostEnvironment environment)
    {
        _environment = environment;
    }
    
    public CorsConfiguration GetConfiguration()
    {
        return _environment.EnvironmentName switch
        {
            "Development" => DevelopmentCors(),
            "Staging" => StagingCors(),
            "Production" => ProductionCors(),
            _ => CorsConfiguration.Default
        };
    }
    
    private CorsConfiguration DevelopmentCors() => new CorsConfiguration()
        .AddPolicy(new CorsPolicy
        {
            PolicyName = "Default",
            Origins = OriginPolicy.Pattern(
                origin => origin.Value.Host == "localhost"),
            AllowedMethods = new HashSet<string> { "GET", "POST", "PUT", "DELETE", "PATCH" },
            AllowedHeaders = new HashSet<string> { "Content-Type", "Authorization" },
            ExposedHeaders = new HashSet<string>(),
            AllowCredentials = true
        });
    
    private CorsConfiguration StagingCors() => new CorsConfiguration()
        .AddPolicy(new CorsPolicy
        {
            PolicyName = "Default",
            Origins = OriginPolicy.Specific(
                Origin.Https("staging.example.com")),
            AllowedMethods = new HashSet<string> { "GET", "POST", "PUT", "DELETE" },
            AllowedHeaders = new HashSet<string> { "Content-Type", "Authorization" },
            ExposedHeaders = new HashSet<string>(),
            AllowCredentials = true
        });
    
    private CorsConfiguration ProductionCors() => new CorsConfiguration()
        .AddPolicy(new CorsPolicy
        {
            PolicyName = "Default",
            Origins = OriginPolicy.Specific(
                Origin.Https("example.com"),
                Origin.Https("www.example.com")),
            AllowedMethods = new HashSet<string> { "GET", "POST" },
            AllowedHeaders = new HashSet<string> { "Content-Type", "Authorization" },
            ExposedHeaders = new HashSet<string> { "X-Total-Count", "X-Page-Number" },
            AllowCredentials = true,
            MaxAge = TimeSpan.FromHours(24)
        });
}

// Register
public void ConfigureServices(IServiceCollection services)
{
    var corsConfig = new EnvironmentAwareCorsConfiguration(Environment)
        .GetConfiguration();
    
    services.AddCors(corsConfig.ConfigureCors);
}
```

## Controller-Level CORS

```csharp
/// <summary>
/// Attribute for controller-level CORS policies.
/// </summary>
[AttributeUsage(AttributeTargets.Class | AttributeTargets.Method)]
public class TypedCorsAttribute : Attribute
{
    public string PolicyName { get; }
    
    public TypedCorsAttribute(string policyName)
    {
        PolicyName = policyName;
    }
}

// Usage
[TypedCors("PublicApi")]
public class PublicApiController : ControllerBase
{
    [HttpGet("data")]
    public IActionResult GetData()
    {
        // Uses PublicApi CORS policy
        return Ok(data);
    }
}

[TypedCors("ProductionApi")]
public class SecureApiController : ControllerBase
{
    [HttpPost("sensitive")]
    public IActionResult PostSensitive([FromBody] SensitiveData data)
    {
        // Uses stricter ProductionApi CORS policy
        return Ok();
    }
}
```

## CORS Preflight Optimization

```csharp
/// <summary>
/// Optimized CORS policy with aggressive preflight caching.
/// </summary>
public static class CorsOptimizations
{
    public static CorsPolicy WithLongPreflight(this CorsPolicy policy) =>
        policy with { MaxAge = TimeSpan.FromDays(7) };
    
    public static CorsPolicy WithShortPreflight(this CorsPolicy policy) =>
        policy with { MaxAge = TimeSpan.FromMinutes(5) };
    
    public static CorsPolicy WithMinimalHeaders(this CorsPolicy policy) =>
        policy with
        {
            AllowedHeaders = new HashSet<string> { "Content-Type" },
            ExposedHeaders = new HashSet<string>()
        };
}
```

## Testing

```csharp
public class CorsConfigurationTests
{
    [Fact]
    public void Origin_Create_WithValidHttpsUrl_Succeeds()
    {
        var result = Origin.Create("https://example.com");
        
        Assert.True(result.IsSuccess);
        Assert.Equal("https://example.com", result.Value.ToString());
    }
    
    [Fact]
    public void Origin_Create_WithInvalidUrl_ReturnsFailure()
    {
        var result = Origin.Create("not-a-url");
        
        Assert.True(result.IsFailure);
    }
    
    [Fact]
    public void Origin_Https_CreatesValidOrigin()
    {
        var origin = Origin.Https("example.com");
        
        Assert.Equal("https://example.com", origin.ToString());
    }
    
    [Fact]
    public void Origin_Localhost_WithPort_CreatesValidOrigin()
    {
        var origin = Origin.Localhost(3000);
        
        Assert.Equal("https://localhost:3000", origin.ToString());
    }
    
    [Fact]
    public void OriginPolicy_Specific_AllowsOnlyListedOrigins()
    {
        var policy = OriginPolicy.Specific(
            Origin.Https("example.com"),
            Origin.Https("test.com"));
        
        var builder = new CorsPolicyBuilder();
        policy.Apply(builder);
        
        var corsPolicy = builder.Build();
        Assert.NotNull(corsPolicy);
    }
    
    [Fact]
    public void OriginPattern_SubdomainsOf_MatchesSubdomains()
    {
        var pattern = OriginPatterns.SubdomainsOf("example.com");
        
        Assert.True(pattern(Origin.Https("example.com")));
        Assert.True(pattern(Origin.Https("api.example.com")));
        Assert.True(pattern(Origin.Https("www.example.com")));
        Assert.False(pattern(Origin.Https("other.com")));
    }
    
    [Fact]
    public void OriginPattern_LocalhostPortRange_MatchesPortsInRange()
    {
        var pattern = OriginPatterns.LocalhostPortRange(3000, 3999);
        
        Assert.True(pattern(Origin.Localhost(3000)));
        Assert.True(pattern(Origin.Localhost(3500)));
        Assert.True(pattern(Origin.Localhost(3999)));
        Assert.False(pattern(Origin.Localhost(4000)));
    }
}
```

## Why It's a Problem

1. **String-based origins**: Typos not caught until runtime
2. **Wildcard abuse**: `AllowAnyOrigin()` used carelessly
3. **No validation**: Invalid origin formats accepted
4. **Inconsistent policies**: Different security levels across endpoints
5. **No type safety**: Cannot leverage compiler for validation

## Symptoms

- CORS errors in browser console
- Wildcard (`*`) origins in production
- String-based origin validation
- Inconsistent CORS policies across controllers
- Typos in origin URLs

## Benefits

- **Type safety**: Compile-time validation of origins
- **Validated origins**: Cannot create invalid `Origin` values
- **Pattern matching**: Flexible origin validation with predicates
- **Environment-aware**: Different policies for dev/staging/prod
- **Testable**: Easy to unit test origin validation logic

## See Also

- [Security Headers](./security-headers.md) — type-safe HTTP security
- [Authentication Context](./authentication-context.md) — user identity
- [Input Sanitization](./input-sanitization.md) — trusted types
- [Capability Security](./capability-security.md) — authorization tokens
