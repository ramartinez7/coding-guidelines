# Type-Safe Query Building (LINQ Safety)

> Building LINQ queries with string column names risks runtime errors—use expression trees and typed query builders to make invalid queries uncompilable.

## Problem

Dynamic LINQ queries using string-based column names, reflection, or `dynamic` types fail at runtime when properties are renamed, removed, or mistyped. The compiler can't verify that query expressions reference valid properties or use correct types.

## Example

### ❌ Before

```csharp
public class UserRepository
{
    private readonly DbContext _context;
    
    public IQueryable<User> FilterBy(string propertyName, object value)
    {
        // String-based property access—no compile-time safety
        return _context.Users.Where($"{propertyName} == @0", value);
    }
    
    public IQueryable<User> OrderBy(string columnName, bool ascending)
    {
        // String-based ordering—property rename breaks this
        if (ascending)
            return _context.Users.OrderBy(columnName);
        else
            return _context.Users.OrderByDescending(columnName);
    }
    
    public IQueryable<User> Search(string searchTerm, string[] columns)
    {
        // Runtime column validation
        var query = _context.Users.AsQueryable();
        foreach (var column in columns)
        {
            query = query.Where($"{column}.Contains(@0)", searchTerm);
        }
        return query;
    }
}

// Problems: All failures are runtime errors
var users = repo.FilterBy("Emai", "john@example.com");  // Typo! Runtime error
var ordered = repo.OrderBy("CreatedDate", true);  // Property renamed? Runtime error
```

### ✅ After

```csharp
/// <summary>
/// Type-safe query builder using expression trees.
/// </summary>
public sealed class TypedQueryBuilder<T> where T : class
{
    private readonly IQueryable<T> _query;
    private readonly List<Expression<Func<T, bool>>> _filters = new();
    
    private TypedQueryBuilder(IQueryable<T> query)
    {
        _query = query;
    }
    
    public static TypedQueryBuilder<T> From(IQueryable<T> source)
        => new(source);
    
    public TypedQueryBuilder<T> Where(Expression<Func<T, bool>> predicate)
    {
        _filters.Add(predicate);
        return this;
    }
    
    public TypedQueryBuilder<T> WhereEquals<TProperty>(
        Expression<Func<T, TProperty>> selector,
        TProperty value)
    {
        var parameter = Expression.Parameter(typeof(T), "x");
        var property = Expression.Invoke(selector, parameter);
        var constant = Expression.Constant(value, typeof(TProperty));
        var equals = Expression.Equal(property, constant);
        var lambda = Expression.Lambda<Func<T, bool>>(equals, parameter);
        
        _filters.Add(lambda);
        return this;
    }
    
    public TypedQueryBuilder<T> WhereContains(
        Expression<Func<T, string>> selector,
        string value)
    {
        var parameter = Expression.Parameter(typeof(T), "x");
        var property = Expression.Invoke(selector, parameter);
        var constant = Expression.Constant(value);
        var method = typeof(string).GetMethod(nameof(string.Contains), new[] { typeof(string) })!;
        var contains = Expression.Call(property, method, constant);
        var lambda = Expression.Lambda<Func<T, bool>>(contains, parameter);
        
        _filters.Add(lambda);
        return this;
    }
    
    public IOrderedQueryBuilder<T, TKey> OrderBy<TKey>(
        Expression<Func<T, TKey>> keySelector)
    {
        return new OrderedQueryBuilder<T, TKey>(this, keySelector, ascending: true);
    }
    
    public IOrderedQueryBuilder<T, TKey> OrderByDescending<TKey>(
        Expression<Func<T, TKey>> keySelector)
    {
        return new OrderedQueryBuilder<T, TKey>(this, keySelector, ascending: false);
    }
    
    public IQueryable<T> Build()
    {
        var query = _query;
        foreach (var filter in _filters)
        {
            query = query.Where(filter);
        }
        return query;
    }
}

public interface IOrderedQueryBuilder<T, TKey> where T : class
{
    IOrderedQueryBuilder<T, TKey2> ThenBy<TKey2>(Expression<Func<T, TKey2>> keySelector);
    IOrderedQueryBuilder<T, TKey2> ThenByDescending<TKey2>(Expression<Func<T, TKey2>> keySelector);
    IQueryable<T> Build();
}

public sealed class OrderedQueryBuilder<T, TKey> : IOrderedQueryBuilder<T, TKey> where T : class
{
    private readonly TypedQueryBuilder<T> _builder;
    private readonly List<(LambdaExpression selector, bool ascending)> _orderings = new();
    
    internal OrderedQueryBuilder(
        TypedQueryBuilder<T> builder,
        Expression<Func<T, TKey>> keySelector,
        bool ascending)
    {
        _builder = builder;
        _orderings.Add((keySelector, ascending));
    }
    
    public IOrderedQueryBuilder<T, TKey2> ThenBy<TKey2>(
        Expression<Func<T, TKey2>> keySelector)
    {
        _orderings.Add((keySelector, ascending: true));
        return new OrderedQueryBuilder<T, TKey2>(_builder, keySelector, true);
    }
    
    public IOrderedQueryBuilder<T, TKey2> ThenByDescending<TKey2>(
        Expression<Func<T, TKey2>> keySelector)
    {
        _orderings.Add((keySelector, ascending: false));
        return new OrderedQueryBuilder<T, TKey2>(_builder, keySelector, false);
    }
    
    public IQueryable<T> Build()
    {
        var query = _builder.Build();
        
        if (_orderings.Count == 0)
            return query;
        
        var first = _orderings[0];
        IOrderedQueryable<T> ordered = first.ascending
            ? query.OrderBy((Expression<Func<T, object>>)first.selector)
            : query.OrderByDescending((Expression<Func<T, object>>)first.selector);
        
        foreach (var ordering in _orderings.Skip(1))
        {
            ordered = ordering.ascending
                ? ordered.ThenBy((Expression<Func<T, object>>)ordering.selector)
                : ordered.ThenByDescending((Expression<Func<T, object>>)ordering.selector);
        }
        
        return ordered;
    }
}

// Usage: Compile-time safety for queries
public class UserRepository
{
    private readonly DbContext _context;
    
    public IQueryable<User> SearchUsers(string email, string searchTerm)
    {
        return TypedQueryBuilder<User>
            .From(_context.Users)
            .WhereEquals(u => u.Email, email)  // Compile-time property check
            .WhereContains(u => u.Name, searchTerm)  // Type-safe string search
            .OrderBy(u => u.CreatedAt)  // Property must exist
            .ThenByDescending(u => u.Id)  // Chained ordering
            .Build();
    }
}

// Won't compile:
// .WhereEquals(u => u.Emai, "test")  // Error: No property 'Emai'
// .OrderBy(u => u.CreatedDate)  // Error: Property doesn't exist
// .WhereEquals(u => u.Email, 123)  // Error: Type mismatch (int vs string)
```

## Specification Pattern with Type Safety

```csharp
/// <summary>
/// Type-safe specification for composable query logic.
/// </summary>
public interface ISpecification<T>
{
    Expression<Func<T, bool>> ToExpression();
}

public sealed class EmailSpecification : ISpecification<User>
{
    private readonly string _email;
    
    public EmailSpecification(string email)
    {
        _email = email;
    }
    
    public Expression<Func<User, bool>> ToExpression()
    {
        return user => user.Email == _email;
    }
}

public sealed class ActiveUserSpecification : ISpecification<User>
{
    public Expression<Func<User, bool>> ToExpression()
    {
        return user => user.IsActive;
    }
}

public static class SpecificationExtensions
{
    public static ISpecification<T> And<T>(
        this ISpecification<T> left,
        ISpecification<T> right)
    {
        return new AndSpecification<T>(left, right);
    }
    
    public static ISpecification<T> Or<T>(
        this ISpecification<T> left,
        ISpecification<T> right)
    {
        return new OrSpecification<T>(left, right);
    }
    
    public static ISpecification<T> Not<T>(this ISpecification<T> spec)
    {
        return new NotSpecification<T>(spec);
    }
}

internal sealed class AndSpecification<T> : ISpecification<T>
{
    private readonly ISpecification<T> _left;
    private readonly ISpecification<T> _right;
    
    public AndSpecification(ISpecification<T> left, ISpecification<T> right)
    {
        _left = left;
        _right = right;
    }
    
    public Expression<Func<T, bool>> ToExpression()
    {
        var leftExpr = _left.ToExpression();
        var rightExpr = _right.ToExpression();
        
        var parameter = Expression.Parameter(typeof(T));
        var body = Expression.AndAlso(
            Expression.Invoke(leftExpr, parameter),
            Expression.Invoke(rightExpr, parameter));
        
        return Expression.Lambda<Func<T, bool>>(body, parameter);
    }
}

// Usage: Composable, type-safe specifications
var spec = new EmailSpecification("john@example.com")
    .And(new ActiveUserSpecification());

var users = _context.Users.Where(spec.ToExpression());
```

## Why It's a Problem

1. **Runtime failures**: String-based queries fail when properties are renamed or removed
2. **Type mismatches**: No compile-time verification of property types
3. **Refactoring hazards**: Automated refactoring doesn't update string literals
4. **No IntelliSense**: IDE can't suggest valid property names

## Symptoms

- String-based column names in queries
- Dynamic LINQ with magic strings
- Runtime `PropertyNotFoundException`
- Comments listing valid column names
- Tests that verify query string syntax

## Benefits

- **Compile-time safety**: Invalid queries don't compile
- **Refactoring-safe**: Renaming properties updates all query expressions
- **IntelliSense support**: IDE suggests valid properties
- **Type checking**: Compiler verifies property types match values
- **Expression composition**: Combine query logic with type safety

## See Also

- [Type-Safe String Interpolation](./type-safe-string-interpolation.md) — preventing SQL injection
- [Specification Pattern](./specification-pattern.md) — encapsulating query logic
- [Strongly Typed IDs](./strongly-typed-ids.md) — type-safe identifiers
- [Compile-Time Validation](./compile-time-validation.md) — catching errors early
