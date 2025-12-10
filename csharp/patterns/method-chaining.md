# Method Chaining (Fluent Interfaces)

> Design APIs where methods return `this` or the next builder step—enable readable, composable operation sequences.

## Problem

APIs that require intermediate variables for each operation are verbose and interrupt the flow of thought. Fluent interfaces express intent more clearly by chaining related operations together.

## Example

### ❌ Before

```csharp
public class EmailMessage
{
    public string To { get; set; }
    public string Subject { get; set; }
    public string Body { get; set; }
    public List<string> Attachments { get; } = new();
}

public class EmailService
{
    public void SendWelcomeEmail(User user)
    {
        var email = new EmailMessage();
        email.To = user.Email.Value;
        email.Subject = "Welcome!";
        email.Body = GenerateWelcomeBody(user);
        email.Attachments.Add("terms.pdf");
        email.Attachments.Add("guide.pdf");
        
        Send(email);
    }
}
```

### ✅ After

```csharp
public sealed class EmailMessage
{
    private readonly string to;
    private readonly string subject;
    private readonly string body;
    private readonly IReadOnlyList<string> attachments;

    private EmailMessage(string to, string subject, string body, IReadOnlyList<string> attachments)
    {
        this.to = to;
        this.subject = subject;
        this.body = body;
        this.attachments = attachments;
    }

    public static IToStep CreateNew() => new EmailBuilder();

    public interface IToStep
    {
        ISubjectStep To(EmailAddress address);
    }

    public interface ISubjectStep
    {
        IBodyStep WithSubject(string subject);
    }

    public interface IBodyStep
    {
        IBuildStep WithBody(string body);
    }

    public interface IBuildStep
    {
        IBuildStep Attach(string filename);
        EmailMessage Build();
    }

    private sealed class EmailBuilder : IToStep, ISubjectStep, IBodyStep, IBuildStep
    {
        private string? to;
        private string? subject;
        private string? body;
        private readonly List<string> attachments = new();

        public ISubjectStep To(EmailAddress address)
        {
            to = address.Value;
            return this;
        }

        public IBodyStep WithSubject(string subject)
        {
            this.subject = subject;
            return this;
        }

        public IBuildStep WithBody(string body)
        {
            this.body = body;
            return this;
        }

        public IBuildStep Attach(string filename)
        {
            attachments.Add(filename);
            return this;
        }

        public EmailMessage Build()
        {
            if (to is null) throw new InvalidOperationException("To address is required");
            if (subject is null) throw new InvalidOperationException("Subject is required");
            if (body is null) throw new InvalidOperationException("Body is required");

            return new EmailMessage(to, subject, body, attachments);
        }
    }
}

public class EmailService
{
    public void SendWelcomeEmail(User user)
    {
        var email = EmailMessage.CreateNew()
            .To(user.Email)
            .WithSubject("Welcome!")
            .WithBody(GenerateWelcomeBody(user))
            .Attach("terms.pdf")
            .Attach("guide.pdf")
            .Build();
        
        Send(email);
    }
}
```

## Why It's Better

1. **Readable**: Chains read like natural language.

2. **Type-safe**: Compiler enforces the sequence of operations.

3. **Discoverable**: IDE autocomplete guides you through the API.

4. **Concise**: Fewer intermediate variables and ceremony.

5. **Immutable**: Each step can return a new instance rather than mutating.

## Types of Fluent Interfaces

### 1. Simple Chaining (Return This)

The simplest form—return `this` after each operation:

```csharp
public sealed class QueryBuilder
{
    private readonly List<string> conditions = new();
    private string? orderBy;

    public QueryBuilder Where(string condition)
    {
        conditions.Add(condition);
        return this;
    }

    public QueryBuilder OrderBy(string column)
    {
        orderBy = column;
        return this;
    }

    public string Build()
    {
        var sql = "SELECT * FROM Users";
        if (conditions.Any())
            sql += " WHERE " + string.Join(" AND ", conditions);
        if (orderBy != null)
            sql += $" ORDER BY {orderBy}";
        return sql;
    }
}

// Usage
var query = new QueryBuilder()
    .Where("Age > 18")
    .Where("Active = true")
    .OrderBy("Name")
    .Build();
```

### 2. Immutable Chaining (Return New Instance)

Each method returns a new instance—safer for concurrent use:

```csharp
public sealed record HttpRequest
{
    private readonly string url;
    private readonly IReadOnlyDictionary<string, string> headers;
    private readonly string? body;

    private HttpRequest(
        string url,
        IReadOnlyDictionary<string, string> headers,
        string? body)
    {
        this.url = url;
        this.headers = headers;
        this.body = body;
    }

    public static HttpRequest To(string url)
    {
        return new HttpRequest(url, new Dictionary<string, string>(), null);
    }

    public HttpRequest WithHeader(string name, string value)
    {
        var newHeaders = new Dictionary<string, string>(headers)
        {
            [name] = value
        };
        return new HttpRequest(url, newHeaders, body);
    }

    public HttpRequest WithBody(string body)
    {
        return new HttpRequest(url, headers, body);
    }
}

// Usage (each step creates a new immutable instance)
var request = HttpRequest.To("https://api.example.com/users")
    .WithHeader("Authorization", "Bearer token")
    .WithHeader("Content-Type", "application/json")
    .WithBody("{\"name\":\"Alice\"}");
```

### 3. Type-Safe Builder (Change Return Type)

Each step returns a different interface—enforces correct order at compile time:

```csharp
public sealed class Order
{
    private Order() { }

    public static ICustomerStep Create() => new OrderBuilder();

    public interface ICustomerStep
    {
        IItemsStep ForCustomer(CustomerId customerId);
    }

    public interface IItemsStep
    {
        IItemsStep AddItem(ProductId productId, Quantity quantity);
        IBuildStep Done();
    }

    public interface IBuildStep
    {
        IBuildStep WithShipping(Address address);
        Order Build();
    }

    private sealed class OrderBuilder : ICustomerStep, IItemsStep, IBuildStep
    {
        private CustomerId? customerId;
        private readonly List<OrderItem> items = new();
        private Address? shippingAddress;

        public IItemsStep ForCustomer(CustomerId customerId)
        {
            this.customerId = customerId;
            return this;
        }

        public IItemsStep AddItem(ProductId productId, Quantity quantity)
        {
            items.Add(new OrderItem(productId, quantity));
            return this;
        }

        public IBuildStep Done()
        {
            if (items.Count == 0)
                throw new InvalidOperationException("Order must have at least one item");
            return this;
        }

        public IBuildStep WithShipping(Address address)
        {
            shippingAddress = address;
            return this;
        }

        public Order Build()
        {
            return new Order
            {
                CustomerId = customerId!,
                Items = items,
                ShippingAddress = shippingAddress
            };
        }
    }
}

// Usage - compiler enforces order
var order = Order.Create()
    .ForCustomer(customerId)      // Must be first
    .AddItem(productId, quantity) // Must have customer
    .Done()                       // Must have items
    .WithShipping(address)        // Optional
    .Build();
```

## Design Guidelines

### 1. Start Simple

Begin with basic chaining before adding complexity:

```csharp
// Start here
public class Builder
{
    public Builder SetName(string name) { this.name = name; return this; }
    public Builder SetAge(int age) { this.age = age; return this; }
}

// Add type safety only if order matters
```

### 2. Make Terminal Operations Clear

Methods that end the chain should not return `this`:

```csharp
public class QueryBuilder
{
    public QueryBuilder Where(string condition) => this;  // Continues chain
    
    public string Build() => sql;          // Ends chain
    public int Execute() => db.Execute();  // Ends chain
}
```

### 3. Consider Immutability

Decide early whether to mutate or return new instances:

```csharp
// Mutable (fine for local builders)
public QueryBuilder Where(string condition)
{
    this.conditions.Add(condition);
    return this;
}

// Immutable (better for shared/reused builders)
public QueryBuilder Where(string condition)
{
    return new QueryBuilder(conditions.Append(condition));
}
```

### 4. Name Methods Clearly

Use verbs that indicate what the method does:

```csharp
// ✅ Good
builder.AddItem(item)
builder.WithTimeout(timeout)
builder.Retry(3)

// ❌ Unclear
builder.Item(item)
builder.Timeout(timeout)
builder.Three()
```

## When to Use

Use fluent interfaces when:

- Configuring complex objects with many options
- Building queries or expressions
- Defining sequences of operations
- The API is used frequently and readability matters
- Operations naturally flow in a sequence

## When Not to Use

Avoid fluent interfaces when:

- The operation is simple (one or two parameters)
- Order doesn't matter
- Methods have side effects that aren't obvious
- The API is rarely used
- It makes error handling complicated

```csharp
// ❌ Overuse - this is too simple for fluency
user.SetName("Alice").SetAge(30);

// ✅ Better - direct assignment
user.Name = "Alice";
user.Age = 30;

// ❌ Overuse - hides important error handling
await httpClient.Get(url).ThrowIfError().ReadJson<User>();

// ✅ Better - explicit error handling
var response = await httpClient.Get(url);
if (!response.IsSuccess)
    throw new HttpException(response);
var user = await response.ReadJson<User>();
```

## See Also

- [Type-Safe Builder](./type-safe-builder.md) — enforcing build sequences with types
- [Enforcing Call Order](./enforcing-call-order.md) — preventing invalid method sequences
- [Value Semantics](./value-semantics.md) — immutable chaining with records
- [Smart Constructors](./smart-constructors.md) — validation during construction
