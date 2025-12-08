# Meaningful Names

> Names should reveal intent—no comments needed to explain what a variable, method, or class does.

## Problem

Generic names like `data`, `temp`, `x`, or abbreviated names like `usr` force readers to hold context in their heads. Misleading names (e.g., `GetUser` that modifies state) create cognitive dissonance and bugs.

## Example

### ❌ Before

```csharp
public class Manager
{
    public void Process(List<int> d)
    {
        var temp = new List<int>();
        foreach (var x in d)
        {
            if (x > 0)
            {
                temp.Add(x * 2);
            }
        }
        this.d = temp;
    }

    private List<int> d;
}
```

### ✅ After

```csharp
public class OrderProcessor
{
    public void DoublePositiveOrderQuantities(List<OrderQuantity> quantities)
    {
        var doubledQuantities = new List<OrderQuantity>();

        foreach (var quantity in quantities)
        {
            if (quantity.IsPositive)
            {
                doubledQuantities.Add(quantity.Double());
            }
        }

        this.processedQuantities = doubledQuantities;
    }

    private List<OrderQuantity> processedQuantities;
}
```

## Naming Guidelines

### Classes and Types

**Use nouns or noun phrases** that describe what the type represents:

```csharp
// ❌ Avoid generic suffixes
public class DataManager { }
public class InfoHandler { }
public class BaseProcessor { }

// ✅ Specific, intention-revealing names
public class OrderRepository { }
public class PaymentGateway { }
public class ShippingCalculator { }
```

### Methods and Functions

**Use verbs or verb phrases** that describe what the method does:

```csharp
// ❌ Ambiguous or misleading
public User GetUser(UserId id) { /* modifies cache */ }
public void Handle(Order order) { }
public bool Check(Order order) { }

// ✅ Clear intent
public Result<User, UserError> FindUserAndUpdateCache(UserId id) { }
public void ProcessPaymentForOrder(Order order) { }
public bool IsOrderEligibleForDiscount(Order order) { }
```

### Variables

**Use descriptive names** that reveal the variable's purpose:

```csharp
// ❌ Short, cryptic names
var d = DateTime.Now;
var u = GetUser();
var temp = orders.Where(o => o.Total > 100).ToList();

// ✅ Intention-revealing names
var currentDate = DateTime.Now;
var authenticatedUser = GetUser();
var highValueOrders = orders.Where(o => o.Total > 100).ToList();
```

### Booleans

**Use affirmative names** that read naturally in conditionals:

```csharp
// ❌ Negative or unclear
bool notReady;
bool flag;
bool status;

if (!notReady)  // Double negative!
{
    Process();
}

// ✅ Affirmative and clear
bool isReady;
bool hasPermission;
bool canProcess;

if (isReady)
{
    Process();
}
```

### Constants and Configuration

**Use descriptive names** that explain the constant's meaning:

```csharp
// ❌ Magic numbers and unclear names
const int X = 86400;
const int MAX = 100;

// ✅ Self-documenting names
const int SecondsPerDay = 86400;
const int MaxRetryAttempts = 100;
const decimal DefaultTaxRate = 0.08m;
```

## Avoid Encoding Types in Names

Modern IDEs show type information—don't duplicate it in the name:

```csharp
// ❌ Hungarian notation (outdated)
string strName;
int iCount;
List<Order> orderList;
IUserRepository userRepositoryInterface;

// ✅ Clean, type-system does the work
string name;
int count;
List<Order> orders;
IUserRepository userRepository;
```

## Use Domain Language

Names should reflect the **ubiquitous language** of the domain:

```csharp
// ❌ Technical/generic names
public class DataContainer
{
    public void ProcessItems() { }
    public bool ValidateFields() { }
}

// ✅ Domain-specific names
public class ShoppingCart
{
    public void AddProduct(Product product) { }
    public bool HasSufficientInventory() { }
}
```

## Searchable Names

Avoid single-letter names except for short-scope loop counters:

```csharp
// ❌ Not searchable
for (int i = 0; i < customers.Count; i++)
{
    var c = customers[i];
    ProcessCustomer(c);
}

// ✅ Searchable and clear (for larger scopes)
foreach (var customer in customers)
{
    ProcessCustomer(customer);
}

// ✅ Single-letter OK for tiny scopes
var sum = numbers.Select(n => n * 2).Sum();
```

## Consistent Naming Across the Codebase

Use consistent vocabulary for the same concepts:

```csharp
// ❌ Inconsistent synonyms
void FetchUser();
void RetrieveOrder();
void GetProduct();
void LoadInvoice();

// ✅ Consistent pattern
void GetUser();
void GetOrder();
void GetProduct();
void GetInvoice();
```

## Symptoms

- Team members asking "what does this variable do?"
- Comments explaining variable names
- Using debugger because variable names don't reveal their purpose
- Confusion about similar-sounding methods that do different things
- Names that lie about what they do

## Benefits

- **Self-documenting code** reduces need for comments
- **Faster comprehension** for new team members
- **Easier debugging** when variable names reveal their purpose
- **Better IDE support** with searchable, descriptive names
- **Reduced cognitive load** when reading code

## See Also

- [Ubiquitous Language](./ubiquitous-language.md) — Using domain terminology in code
- [Boolean Blindness](./boolean-blindness.md) — Creating meaningful types instead of booleans
- [Primitive Obsession](./primitive-obsession.md) — Using descriptive types instead of primitives
