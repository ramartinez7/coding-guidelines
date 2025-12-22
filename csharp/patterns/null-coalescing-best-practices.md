# Null Coalescing Best Practices

> Use null-coalescing operators, null-conditional operators, and pattern matching for clean, safe null handling.

## Problem

Verbose null checking creates code clutter, while missed null checks cause NullReferenceException.

## Example

### ❌ Before

```csharp
public string GetCustomerName(Customer customer)
{
    if (customer == null)
    {
        return "Unknown";
    }

    if (customer.Name == null)
    {
        return "Unknown";
    }

    return customer.Name;
}

public int GetOrderCount(Customer customer)
{
    if (customer == null)
    {
        return 0;
    }

    if (customer.Orders == null)
    {
        return 0;
    }

    return customer.Orders.Count;
}
```

### ✅ After

```csharp
public string GetCustomerName(Customer? customer)
{
    return customer?.Name ?? "Unknown";
}

public int GetOrderCount(Customer? customer)
{
    return customer?.Orders?.Count ?? 0;
}
```

## Best Practices

### 1. Use Null-Coalescing Operator (??)

```csharp
// ❌ Verbose
string name;
if (input == null)
{
    name = "Default";
}
else
{
    name = input;
}

// ✅ Concise
string name = input ?? "Default";
```

### 2. Use Null-Conditional Operator (?.)

```csharp
// ❌ Manual null checking
int? length = null;
if (text != null)
{
    length = text.Length;
}

// ✅ Null-conditional
int? length = text?.Length;

// ❌ Nested null checks
string? city = null;
if (customer != null)
{
    if (customer.Address != null)
    {
        city = customer.Address.City;
    }
}

// ✅ Chain null-conditional
string? city = customer?.Address?.City;
```

### 3. Use Null-Coalescing Assignment (??=)

```csharp
// ❌ Check and assign
if (cache == null)
{
    cache = new Dictionary<string, object>();
}

// ✅ Null-coalescing assignment
cache ??= new Dictionary<string, object>();

// ❌ Lazy initialization
private List<Item>? items;

public List<Item> Items
{
    get
    {
        if (items == null)
        {
            items = new List<Item>();
        }
        return items;
    }
}

// ✅ Lazy with ??=
private List<Item>? items;

public List<Item> Items => items ??= new List<Item>();
```

### 4. Combine ?? and ?. for Safe Access

```csharp
// ❌ Verbose safe navigation
decimal total = 0;
if (order != null && order.Items != null)
{
    total = order.Items.Sum(i => i.Price);
}

// ✅ Concise safe navigation
decimal total = order?.Items?.Sum(i => i.Price) ?? 0;
```

### 5. Use Pattern Matching for Null Checks

```csharp
// ❌ Traditional null check
if (order != null)
{
    ProcessOrder(order);
}

// ✅ Pattern matching (C# 9+)
if (order is not null)
{
    ProcessOrder(order);
}

// ✅ Or with pattern
if (order is Order o)
{
    ProcessOrder(o);
}
```

### 6. Use Nullable Reference Types

```csharp
#nullable enable

// ❌ Unclear if null is expected
public string GetName(Customer customer)
{
    return customer.Name;  // Could be null!
}

// ✅ Explicit nullability in signature
public string GetName(Customer customer)
{
    return customer.Name ?? "Unknown";  // Compiler warns if Name is nullable
}

public string? TryGetName(Customer? customer)
{
    return customer?.Name;  // Explicit that result can be null
}
```

### 7. Use NotNullWhen for Validation

```csharp
// ✅ Annotate validation methods
public bool TryGetCustomer(
    string id,
    [NotNullWhen(true)] out Customer? customer)
{
    customer = _repository.Find(id);
    return customer != null;
}

// Usage: compiler knows customer is not null here
if (TryGetCustomer(id, out var customer))
{
    ProcessCustomer(customer);  // No null warning
}
```

### 8. Use ArgumentNullException.ThrowIfNull

```csharp
// ❌ Manual null argument check
public void ProcessOrder(Order order)
{
    if (order == null)
    {
        throw new ArgumentNullException(nameof(order));
    }
    // Process
}

// ✅ Built-in helper (C# 11+)
public void ProcessOrder(Order order)
{
    ArgumentNullException.ThrowIfNull(order);
    // Process
}
```

### 9. Use Null-Forgiving Operator Sparingly

```csharp
// ❌ Overusing null-forgiving operator
public void Process(string? input)
{
    var length = input!.Length;  // Suppresses warning, could crash!
}

// ✅ Only when you're certain it's not null
public void Process(Order order)
{
    Debug.Assert(order.Customer != null);
    var email = order.Customer!.Email;  // We know it's not null
}

// ✅ Better: validate properly
public void Process(string? input)
{
    if (string.IsNullOrEmpty(input))
    {
        return;
    }

    var length = input.Length;  // Compiler knows it's not null
}
```

### 10. Use Switch Expression with Null

```csharp
// ❌ If-else chain
string GetStatus(Order? order)
{
    if (order == null)
    {
        return "No order";
    }
    if (order.Status == OrderStatus.Pending)
    {
        return "Pending";
    }
    if (order.Status == OrderStatus.Completed)
    {
        return "Completed";
    }
    return "Unknown";
}

// ✅ Switch expression with null pattern
string GetStatus(Order? order) => order switch
{
    null => "No order",
    { Status: OrderStatus.Pending } => "Pending",
    { Status: OrderStatus.Completed } => "Completed",
    _ => "Unknown"
};
```

### 11. Use FirstOrDefault with Null-Coalescing

```csharp
// ❌ Multiple null checks
var customer = customers.FirstOrDefault(c => c.Id == id);
if (customer == null)
{
    customer = new Customer();
}

// ✅ Null-coalescing with default
var customer = customers.FirstOrDefault(c => c.Id == id) 
    ?? new Customer();

// ✅ Or provide default directly (C# 9+)
var customer = customers.FirstOrDefault(c => c.Id == id, new Customer());
```

### 12. Use Required Properties (C# 11+)

```csharp
// ❌ Constructor with null checks
public class Customer
{
    public string Name { get; set; } = null!;
    public string Email { get; set; } = null!;

    public Customer(string name, string email)
    {
        ArgumentNullException.ThrowIfNull(name);
        ArgumentNullException.ThrowIfNull(email);

        Name = name;
        Email = email;
    }
}

// ✅ Required properties
public class Customer
{
    public required string Name { get; init; }
    public required string Email { get; init; }
}

// Compiler error if not initialized:
var customer = new Customer();  // Error!

// Correct usage:
var customer = new Customer
{
    Name = "John",
    Email = "john@example.com"
};
```

## Symptoms

- NullReferenceException in production
- Verbose null-checking code
- Difficulty knowing when null is valid
- Inconsistent null handling

## Benefits

- **Fewer NullReferenceExceptions** with safe navigation
- **Cleaner code** with concise null operators
- **Compile-time safety** with nullable reference types
- **Better intent** with explicit nullability
- **Less boilerplate** with modern C# features

## See Also

- [Nullable Reference Types](./nullable-reference-types.md) — Compile-time null safety
- [Nullability vs. Optionality](./nullability-optionality.md) — Option types
- [Option Monad](./option-monad.md) — Explicit optionality
- [Honest Functions](./honest-functions.md) — Explicit return types
