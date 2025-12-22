# Equality and Comparison Best Practices

> Implement equality and comparison correctly using records, IEquatable, and consistent GetHashCode.

## Problem

Incorrect equality implementation leads to bugs with dictionaries, hash sets, and comparison operations.

## Example

### ❌ Before

```csharp
public class Money
{
    public decimal Amount { get; set; }
    public string Currency { get; set; }

    // Default reference equality - two Money objects with same values are not equal!
}

var m1 = new Money { Amount = 10, Currency = "USD" };
var m2 = new Money { Amount = 10, Currency = "USD" };
Console.WriteLine(m1 == m2);  // False!
```

### ✅ After

```csharp
public sealed record Money(decimal Amount, string Currency);

var m1 = new Money(10, "USD");
var m2 = new Money(10, "USD");
Console.WriteLine(m1 == m2);  // True!
```

## Best Practices

### 1. Use Records for Value Equality

```csharp
// ✅ Record provides value equality automatically
public sealed record EmailAddress(string Value);

public sealed record Money(decimal Amount, string Currency);

public sealed record Address(
    string Street,
    string City,
    string State,
    string ZipCode);
```

### 2. Implement IEquatable<T> for Performance

```csharp
// ✅ Avoid boxing with IEquatable<T>
public sealed class UserId : IEquatable<UserId>
{
    public Guid Value { get; }

    public UserId(Guid value)
    {
        this.Value = value;
    }

    public bool Equals(UserId? other)
    {
        return other != null && Value.Equals(other.Value);
    }

    public override bool Equals(object? obj)
    {
        return Equals(obj as UserId);
    }

    public override int GetHashCode()
    {
        return Value.GetHashCode();
    }

    public static bool operator ==(UserId? left, UserId? right)
    {
        return Equals(left, right);
    }

    public static bool operator !=(UserId? left, UserId? right)
    {
        return !Equals(left, right);
    }
}
```

### 3. Always Override GetHashCode with Equals

```csharp
// ❌ Overrides Equals but not GetHashCode
public class Product
{
    public string Sku { get; set; }

    public override bool Equals(object? obj)
    {
        return obj is Product p && Sku == p.Sku;
    }
    // Missing GetHashCode override!
}

// ✅ Both Equals and GetHashCode
public class Product
{
    public string Sku { get; set; }

    public override bool Equals(object? obj)
    {
        return obj is Product p && Sku == p.Sku;
    }

    public override int GetHashCode()
    {
        return Sku.GetHashCode();
    }
}
```

### 4. Use HashCode.Combine for Multiple Fields

```csharp
// ❌ Manual hash code calculation (error-prone)
public override int GetHashCode()
{
    return 17 * 23 + Amount.GetHashCode() * 23 + Currency.GetHashCode();
}

// ✅ Use HashCode.Combine
public override int GetHashCode()
{
    return HashCode.Combine(Amount, Currency);
}
```

### 5. Implement IComparable for Ordering

```csharp
// ✅ Implement IComparable<T>
public sealed class Priority : IComparable<Priority>
{
    public int Value { get; }

    public Priority(int value)
    {
        if (value < 1 || value > 5)
        {
            throw new ArgumentException("Priority must be 1-5");
        }

        this.Value = value;
    }

    public int CompareTo(Priority? other)
    {
        if (other == null) return 1;
        return Value.CompareTo(other.Value);
    }

    public static bool operator <(Priority left, Priority right)
    {
        return left.CompareTo(right) < 0;
    }

    public static bool operator >(Priority left, Priority right)
    {
        return left.CompareTo(right) > 0;
    }

    public static bool operator <=(Priority left, Priority right)
    {
        return left.CompareTo(right) <= 0;
    }

    public static bool operator >=(Priority left, Priority right)
    {
        return left.CompareTo(right) >= 0;
    }
}
```

### 6. Use StringComparison for String Equality

```csharp
// ❌ Culture-sensitive comparison
if (status == "active")  // May fail for "Active"
{
}

// ✅ Explicit comparison
if (status.Equals("active", StringComparison.OrdinalIgnoreCase))
{
}
```

### 7. Be Consistent with Null Handling

```csharp
// ✅ Consistent null handling
public override bool Equals(object? obj)
{
    if (obj == null) return false;
    if (ReferenceEquals(this, obj)) return true;
    if (obj.GetType() != GetType()) return false;

    var other = (OrderId)obj;
    return Value.Equals(other.Value);
}
```

### 8. Use Records with Custom Equality

```csharp
// ✅ Custom equality in record
public sealed record CaseInsensitiveString(string Value)
{
    public virtual bool Equals(CaseInsensitiveString? other)
    {
        return other != null &&
            string.Equals(Value, other.Value, StringComparison.OrdinalIgnoreCase);
    }

    public override int GetHashCode()
    {
        return StringComparer.OrdinalIgnoreCase.GetHashCode(Value);
    }
}
```

### 9. Use EqualityComparer<T> for Custom Comparisons

```csharp
// ✅ Custom comparer for collections
public class ProductSkuComparer : IEqualityComparer<Product>
{
    public bool Equals(Product? x, Product? y)
    {
        if (x == null || y == null) return false;
        return x.Sku.Equals(y.Sku, StringComparison.OrdinalIgnoreCase);
    }

    public int GetHashCode(Product obj)
    {
        return StringComparer.OrdinalIgnoreCase.GetHashCode(obj.Sku);
    }
}

// Usage
var products = new HashSet<Product>(new ProductSkuComparer());
```

### 10. Test Equality Implementation

```csharp
[Test]
public void Money_Equality_Works()
{
    var m1 = new Money(10, "USD");
    var m2 = new Money(10, "USD");
    var m3 = new Money(20, "USD");

    // Reflexive
    Assert.That(m1.Equals(m1), Is.True);

    // Symmetric
    Assert.That(m1.Equals(m2), Is.True);
    Assert.That(m2.Equals(m1), Is.True);

    // Transitive (if m1 == m2 and m2 == m3, then m1 == m3)
    // (tested with more examples)

    // Null handling
    Assert.That(m1.Equals(null), Is.False);

    // Inequality
    Assert.That(m1.Equals(m3), Is.False);

    // HashCode consistency
    Assert.That(m1.GetHashCode(), Is.EqualTo(m2.GetHashCode()));
}
```

## Symptoms

- Objects not found in HashSet or Dictionary
- Sorting producing unexpected results
- Comparison operators inconsistent
- Performance issues from boxing
- InvalidOperationException in hash-based collections

## Benefits

- **Correct behavior** in collections
- **Better performance** with IEquatable<T>
- **Predictable comparisons** with explicit operators
- **Type safety** with strong equality
- **Less boilerplate** with records

## See Also

- [Value Semantics](./value-semantics.md) — Value-based equality
- [Record Equality](./record-equality.md) — Compiler-generated equality
- [Value Objects](./entities-vs-value-objects.md) — Identity vs equality
