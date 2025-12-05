# Type-Safe Step Builder

> Builder patterns that allow calling `.Build()` before all required fields are set—use interface segregation to enforce the sequence at compile time.

## Problem

The standard Builder pattern often allows developers to call `.Build()` prematurely, resulting in runtime exceptions. The type system should guide users through required steps and only expose `.Build()` when all prerequisites are met.

## Example

### ❌ Before

```csharp
public class HttpRequestBuilder
{
    private string? _url;
    private string? _method;
    private Dictionary<string, string> _headers = new();
    
    public HttpRequestBuilder WithUrl(string url) { _url = url; return this; }
    public HttpRequestBuilder WithMethod(string method) { _method = method; return this; }
    public HttpRequestBuilder WithHeader(string key, string value) 
    { 
        _headers[key] = value; 
        return this; 
    }
    
    public HttpRequest Build()
    {
        if (_url == null) throw new InvalidOperationException("URL is required");
        if (_method == null) throw new InvalidOperationException("Method is required");
        return new HttpRequest(_url, _method, _headers);
    }
}

// Caller forgets the Method
var request = new HttpRequestBuilder()
    .WithUrl("http://api.com")
    .Build(); // 💥 Runtime explosion
```

### ✅ After

```csharp
// Step interfaces guide the user through required configuration
public interface IHttpRequestBuilderNeedsUrl
{
    IHttpRequestBuilderNeedsMethod WithUrl(string url);
}

public interface IHttpRequestBuilderNeedsMethod
{
    IHttpRequestBuilderReady WithMethod(string method);
    IHttpRequestBuilderReady Get();    // Convenience: implies GET
    IHttpRequestBuilderReady Post();   // Convenience: implies POST
}

public interface IHttpRequestBuilderReady
{
    IHttpRequestBuilderReady WithHeader(string key, string value);
    IHttpRequestBuilderReady WithTimeout(TimeSpan timeout);
    HttpRequest Build();
}

// Single class implements all interfaces
public class HttpRequestBuilder : 
    IHttpRequestBuilderNeedsUrl, 
    IHttpRequestBuilderNeedsMethod, 
    IHttpRequestBuilderReady
{
    private string _url = null!;
    private string _method = null!;
    private readonly Dictionary<string, string> _headers = new();
    private TimeSpan _timeout = TimeSpan.FromSeconds(30);

    private HttpRequestBuilder() { }

    public static IHttpRequestBuilderNeedsUrl Create() => new HttpRequestBuilder();

    // Step 1: URL (required)
    public IHttpRequestBuilderNeedsMethod WithUrl(string url)
    {
        _url = url;
        return this;
    }

    // Step 2: Method (required)
    public IHttpRequestBuilderReady WithMethod(string method)
    {
        _method = method;
        return this;
    }

    public IHttpRequestBuilderReady Get() => WithMethod("GET");
    public IHttpRequestBuilderReady Post() => WithMethod("POST");

    // Step 3+: Optional configuration
    public IHttpRequestBuilderReady WithHeader(string key, string value)
    {
        _headers[key] = value;
        return this;
    }

    public IHttpRequestBuilderReady WithTimeout(TimeSpan timeout)
    {
        _timeout = timeout;
        return this;
    }

    // Only available after all required steps
    public HttpRequest Build() => new(_url, _method, _headers, _timeout);
}

// Usage: compiler guides you through the steps
var request = HttpRequestBuilder
    .Create()                              // Returns IHttpRequestBuilderNeedsUrl
    .WithUrl("http://api.com")             // Returns IHttpRequestBuilderNeedsMethod
    .Get()                                 // Returns IHttpRequestBuilderReady
    .WithHeader("Authorization", token)    // Still IHttpRequestBuilderReady
    .Build();                              // Now available!

// This won't compile—no .Build() on IHttpRequestBuilderNeedsMethod
var invalid = HttpRequestBuilder
    .Create()
    .WithUrl("http://api.com")
    .Build();  // ❌ Compile error: IHttpRequestBuilderNeedsMethod has no Build()
```

## Why It's a Problem

- **Temporal coupling**: `WithUrl` must happen before `Build`, but nothing enforces it
- **Deferred validation**: You don't find out you missed a step until runtime
- **Poor discoverability**: IDE shows all methods at every step, even invalid ones
- **Silent misconfiguration**: Easy to forget required steps in complex builders

## Symptoms

- Runtime exceptions like `"X is required"` in `Build()` methods
- Builders with many nullable fields that get validated at the end
- Documentation or comments explaining which methods must be called first
- Unit tests that specifically check for "forgot to call X" scenarios
- Defensive null checks scattered throughout the built object

## When to Use This Pattern

**Good fit:**
- Complex objects with multiple required fields
- Configuration that must follow a specific sequence
- Public APIs where misconfiguration is common
- Builders with many optional settings after required ones

**Might be overkill:**
- Simple objects with 1-2 required fields (use a constructor)
- Internal code with well-understood conventions
- Objects where all fields are optional

## Variations

### Branching Steps

Sometimes the next step depends on a choice:

```csharp
public interface IConnectionBuilderNeedsAuth
{
    IConnectionBuilderReady Anonymous();
    IConnectionBuilderNeedsPassword WithUsername(string username);
}

public interface IConnectionBuilderNeedsPassword
{
    IConnectionBuilderReady WithPassword(string password);
}

// Usage
Connection.Create()
    .WithHost("db.example.com")
    .WithUsername("admin")      // Now you MUST provide password
    .WithPassword("secret")
    .Build();

Connection.Create()
    .WithHost("db.example.com")
    .Anonymous()                // No password needed
    .Build();
```

### Generic Step Builder

For reusable patterns:

```csharp
public interface IBuilder<T>
{
    T Build();
}

public interface IRequires<TNext>
{
    TNext With(/* ... */);
}
```

## Benefits

- **Compile-time safety**: Missing steps are caught by the compiler
- **IDE guidance**: Autocomplete only shows valid next steps
- **Self-documenting**: The interface chain reveals the required sequence
- **No runtime validation**: `Build()` doesn't need null checks for required fields
- **Impossible to misuse**: The type system prevents invalid configurations

## Trade-offs

- **More interfaces**: Each step needs its own interface
- **Complexity**: Simple builders don't need this machinery
- **Refactoring cost**: Adding a new required step changes the interface chain

## See Also

- [Enforcing Call Order](./enforcing-call-order.md) — same principle applied to workflows
- [Static Factory Methods](./static-factory-methods.md) — control object creation
- [Ghost States](./ghost-states.md) — avoid partially-configured objects
