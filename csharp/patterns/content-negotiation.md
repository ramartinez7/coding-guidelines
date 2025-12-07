# Content Negotiation (Type-Driven Response Formatting)

> Checking `Accept` headers with strings and manually serializing responses—use types to represent content types and let the framework handle serialization.

## Problem

APIs that support multiple content types (JSON, XML, CSV) often check headers manually and serialize responses with string-based format specifiers. This is error-prone and provides no compile-time guarantees about format support.

## Example

### ❌ Before

```csharp
[ApiController]
[Route("api/users")]
public class UsersController : ControllerBase
{
    [HttpGet("{id}")]
    public IActionResult GetUser(int id)
    {
        var user = _repository.GetUser(id);
        var accept = Request.Headers["Accept"].ToString();
        
        // String matching at runtime
        if (accept.Contains("application/json"))
        {
            return Ok(JsonSerializer.Serialize(user));
        }
        else if (accept.Contains("application/xml"))
        {
            var xml = SerializeToXml(user);
            return Content(xml, "application/xml");
        }
        else if (accept.Contains("text/csv"))
        {
            var csv = ConvertToCsv(user);
            return Content(csv, "text/csv");
        }
        
        return BadRequest("Unsupported content type");
    }
}
```

**Problems:**
- Manual header parsing
- String-based format checking
- Duplicate serialization code
- No compile-time verification formats are supported
- Easy to forget to implement a format

### ✅ After

```csharp
/// <summary>
/// Marker interface for supported content types.
/// </summary>
public interface IContentType
{
    string MediaType { get; }
}

public sealed record JsonContentType : IContentType
{
    public static JsonContentType Instance { get; } = new();
    private JsonContentType() { }
    public string MediaType => "application/json";
}

public sealed record XmlContentType : IContentType
{
    public static XmlContentType Instance { get; } = new();
    private XmlContentType() { }
    public string MediaType => "application/xml";
}

public sealed record CsvContentType : IContentType
{
    public static CsvContentType Instance { get; } = new();
    private CsvContentType() { }
    public string MediaType => "text/csv";
}

/// <summary>
/// Formatter that can serialize a type to a specific content type.
/// </summary>
public interface IFormatter<T, TContentType> where TContentType : IContentType
{
    string Format(T value);
}

// JSON formatter
public class UserJsonFormatter : IFormatter<User, JsonContentType>
{
    public string Format(User user)
    {
        return JsonSerializer.Serialize(new
        {
            id = user.Id,
            name = user.Name,
            email = user.Email.Value
        });
    }
}

// XML formatter
public class UserXmlFormatter : IFormatter<User, XmlContentType>
{
    public string Format(User user)
    {
        var serializer = new XmlSerializer(typeof(UserDto));
        using var writer = new StringWriter();
        serializer.Serialize(writer, new UserDto
        {
            Id = user.Id,
            Name = user.Name,
            Email = user.Email.Value
        });
        return writer.ToString();
    }
}

// CSV formatter
public class UserCsvFormatter : IFormatter<User, CsvContentType>
{
    public string Format(User user)
    {
        return $"{user.Id},{user.Name},{user.Email.Value}";
    }
}

/// <summary>
/// Negotiates content type and formats response.
/// </summary>
public class ContentNegotiator
{
    public Result<FormattedResponse, string> Negotiate<T>(
        HttpRequest request,
        T data,
        IEnumerable<IContentType> supportedTypes)
    {
        var acceptHeader = request.Headers["Accept"].ToString();
        
        foreach (var contentType in supportedTypes)
        {
            if (acceptHeader.Contains(contentType.MediaType, StringComparison.OrdinalIgnoreCase))
            {
                return contentType switch
                {
                    JsonContentType => FormatAsJson(data),
                    XmlContentType => FormatAsXml(data),
                    CsvContentType => FormatAsCsv(data),
                    _ => Result<FormattedResponse, string>.Failure("Unsupported content type")
                };
            }
        }
        
        return Result<FormattedResponse, string>.Failure("No acceptable content type found");
    }
    
    private Result<FormattedResponse, string> FormatAsJson<T>(T data)
    {
        var json = JsonSerializer.Serialize(data);
        return Result<FormattedResponse, string>.Success(
            new FormattedResponse(json, JsonContentType.Instance.MediaType));
    }
    
    private Result<FormattedResponse, string> FormatAsXml<T>(T data)
    {
        // Implementation
        return Result<FormattedResponse, string>.Success(
            new FormattedResponse("<xml/>", XmlContentType.Instance.MediaType));
    }
    
    private Result<FormattedResponse, string> FormatAsCsv<T>(T data)
    {
        // Implementation
        return Result<FormattedResponse, string>.Success(
            new FormattedResponse("csv", CsvContentType.Instance.MediaType));
    }
}

public sealed record FormattedResponse(string Content, string MediaType);

[ApiController]
[Route("api/users")]
public class UsersController : ControllerBase
{
    private readonly ContentNegotiator _negotiator;
    
    [HttpGet("{id}")]
    public IActionResult GetUser(int id)
    {
        var user = _repository.GetUser(id);
        
        var supportedTypes = new IContentType[]
        {
            JsonContentType.Instance,
            XmlContentType.Instance,
            CsvContentType.Instance
        };
        
        var result = _negotiator.Negotiate(Request, user, supportedTypes);
        
        return result.Match(
            onSuccess: response => Content(response.Content, response.MediaType),
            onFailure: error => StatusCode(406, error)  // 406 Not Acceptable
        );
    }
}
```

## ASP.NET Core Integration

```csharp
// Custom output formatter
public class CsvOutputFormatter : TextOutputFormatter
{
    public CsvOutputFormatter()
    {
        SupportedMediaTypes.Add(MediaTypeHeaderValue.Parse("text/csv"));
        SupportedEncodings.Add(Encoding.UTF8);
    }
    
    public override async Task WriteResponseBodyAsync(
        OutputFormatterWriteContext context,
        Encoding selectedEncoding)
    {
        var response = context.HttpContext.Response;
        var csv = FormatAsCsv(context.Object);
        await response.WriteAsync(csv, selectedEncoding);
    }
    
    private string FormatAsCsv(object? data)
    {
        if (data is IEnumerable<User> users)
        {
            var sb = new StringBuilder();
            sb.AppendLine("Id,Name,Email");
            foreach (var user in users)
            {
                sb.AppendLine($"{user.Id},{user.Name},{user.Email.Value}");
            }
            return sb.ToString();
        }
        
        return string.Empty;
    }
}

// Register in Startup
public void ConfigureServices(IServiceCollection services)
{
    services.AddControllers(options =>
    {
        options.OutputFormatters.Add(new CsvOutputFormatter());
    });
}

// Controller just returns data—framework handles formatting
[HttpGet]
[Produces("application/json", "text/csv", "application/xml")]
public IActionResult GetUsers()
{
    var users = _repository.GetAll();
    return Ok(users);  // Framework negotiates and formats automatically
}
```

## Why It's a Problem

1. **Manual serialization**: Duplicate code for each format
2. **String-based**: No compile-time verification
3. **Forgotten formats**: Easy to not implement all supported types
4. **Testing complexity**: Must test each format separately

## Benefits

- **Framework-handled**: ASP.NET Core does negotiation
- **Type-safe**: Content types are types, not strings
- **DRY**: Single serialization implementation per format
- **Declarative**: `[Produces]` attribute declares support

## See Also

- [Versioned Endpoints](./versioned-endpoints.md) — type-safe versioning
- [DTO vs. Domain Boundary](./dto-domain-boundary.md) — response types
- [Honest Functions](./honest-functions.md) — explicit contracts
