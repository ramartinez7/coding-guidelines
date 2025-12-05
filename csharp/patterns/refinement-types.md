# Structural Constraints (Refinement Types)

> Using standard collection types when the structure matters—wrap collections to enforce constraints like "non-empty" at compile time.

## Problem

Sometimes the type is correct (`List<T>`), but the *structure* is dangerous (it's empty). Operations like `.First()`, `.Average()`, or `list[0]` fail on empty collections. Refinement types constrain not just *what* something is, but *how* it's shaped.

## Example

### ❌ Before

```csharp
public class ReportGenerator
{
    public decimal CalculateAverage(List<decimal> values)
    {
        // What if values is empty? Runtime exception!
        return values.Average();
    }

    public Report Generate(List<DataPoint> dataPoints)
    {
        if (!dataPoints.Any())
            throw new ArgumentException("Need at least one data point");

        var first = dataPoints.First();  // Safe only because we checked above
        var last = dataPoints.Last();
        var average = CalculateAverage(dataPoints.Select(d => d.Value).ToList());
        
        // ... but what if someone refactors and removes the check?
        return new Report(first, last, average);
    }
}

// Caller has no idea this will blow up
var report = generator.Generate(new List<DataPoint>());  // 💥
```

### ✅ After

```csharp
/// <summary>
/// A list guaranteed to have at least one element.
/// </summary>
public sealed class NonEmptyList<T> : IReadOnlyList<T>
{
    private readonly IReadOnlyList<T> _items;

    private NonEmptyList(IReadOnlyList<T> items) => _items = items;

    // Construction: only way in is through validation
    public static NonEmptyList<T> FromList(IReadOnlyList<T> items)
    {
        if (items.Count == 0)
            throw new ArgumentException("List cannot be empty", nameof(items));
        return new NonEmptyList<T>(items);
    }

    public static Option<NonEmptyList<T>> TryFromList(IReadOnlyList<T> items)
        => items.Count > 0 
            ? Option<NonEmptyList<T>>.Some(new NonEmptyList<T>(items)) 
            : Option<NonEmptyList<T>>.None;

    public static NonEmptyList<T> Of(T first, params T[] rest)
        => new(new[] { first }.Concat(rest).ToList());

    // Safe operations: guaranteed to succeed
    public T First => _items[0];
    public T Last => _items[^1];
    public int Count => _items.Count;

    // Indexer: still requires bounds checking (only index 0 is guaranteed)
    public T this[int index] => _items[index];
    
    public Option<T> TryGet(int index)
        => index >= 0 && index < Count 
            ? Option<T>.Some(_items[index]) 
            : Option<T>.None;

    // Preserve the constraint through transformations
    // Select is 1:1 mapping, so Count is preserved: if Count >= 1 before, Count >= 1 after
    public NonEmptyList<TResult> Select<TResult>(Func<T, TResult> selector)
        => new(this.Select(selector).ToList());
    
    // Where does NOT preserve the constraint (could filter to zero elements)
    // so it must return a regular list or Option<NonEmptyList<T>>
    public IReadOnlyList<T> Where(Func<T, bool> predicate)
        => _items.Where(predicate).ToList();

    public Option<NonEmptyList<T>> TryWhere(Func<T, bool> predicate)
        => TryFromList(_items.Where(predicate).ToList());

    public IEnumerator<T> GetEnumerator() => _items.GetEnumerator();
    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}

public class ReportGenerator
{
    // Signature is honest: requires non-empty input
    public decimal CalculateAverage(NonEmptyList<decimal> values)
    {
        // No check needed—type guarantees at least one element
        return values.Average();
    }

    public Report Generate(NonEmptyList<DataPoint> dataPoints)
    {
        // All these are safe—guaranteed by the type
        var first = dataPoints.First;
        var last = dataPoints.Last;
        var average = CalculateAverage(dataPoints.Select(d => d.Value));
        
        return new Report(first, last, average);
    }
}

// Caller must prove the list is non-empty at the boundary
var dataPoints = LoadDataPoints();

NonEmptyList<DataPoint>.TryFromList(dataPoints).Match(
    onSome: nonEmpty => generator.Generate(nonEmpty),
    onNone: () => ShowError("No data points found")
);

// Or use the guaranteed constructor
var report = generator.Generate(NonEmptyList<DataPoint>.Of(firstPoint, morePoints));
```

## Common Refinement Types

### NonEmptyList<T>

Guarantees at least one element:

```csharp
public T First => _items[0];           // Always safe
public T Single => _items.Single();    // Safe if Count == 1
public decimal Average() => ...;       // Always safe
```

### NonEmptyString

Guarantees non-null and non-whitespace:

```csharp
public readonly record struct NonEmptyString
{
    public string Value { get; }

    private NonEmptyString(string value) => Value = value;

    public static Option<NonEmptyString> Create(string? input)
        => !string.IsNullOrWhiteSpace(input)
            ? Option<NonEmptyString>.Some(new NonEmptyString(input.Trim()))
            : Option<NonEmptyString>.None;

    public static implicit operator string(NonEmptyString s) => s.Value;
}

// Usage
void Greet(NonEmptyString name)
{
    Console.WriteLine($"Hello, {name}!");  // name is guaranteed non-empty
}
```

### PositiveInt / NaturalNumber

Guarantees positive values:

```csharp
public readonly record struct PositiveInt
{
    public int Value { get; }

    private PositiveInt(int value) => Value = value;

    public static Option<PositiveInt> Create(int value)
        => value > 0 
            ? Option<PositiveInt>.Some(new PositiveInt(value)) 
            : Option<PositiveInt>.None;

    // Safe operations
    public static PositiveInt operator +(PositiveInt a, PositiveInt b)
        => new(a.Value + b.Value);  // Sum of positives is positive
}

// Usage: array index that's guaranteed valid
void GetItem<T>(T[] array, PositiveInt index) where index.Value <= array.Length
```

### BoundedList<T>

Guarantees size constraints:

```csharp
public sealed class BoundedList<T>
{
    private readonly List<T> _items;
    private readonly int _maxSize;

    public bool CanAdd => _items.Count < _maxSize;
    public bool IsFull => _items.Count >= _maxSize;

    public Option<BoundedList<T>> TryAdd(T item)
    {
        if (IsFull) return Option<BoundedList<T>>.None;
        return Option<BoundedList<T>>.Some(new BoundedList<T>([.._items, item], _maxSize));
    }
}

// A hand in a card game can't exceed 5 cards
BoundedList<Card> hand = BoundedList<Card>.Empty(maxSize: 5);
```

## Why It's a Problem

- **Deferred failures**: Empty collection errors surface deep in call stacks
- **Defensive programming**: Repeating `.Any()` checks everywhere
- **Lost knowledge**: After checking `if (list.Any())`, the type is still `List<T>`
- **Refactoring hazards**: Removing a check breaks distant code silently
- **Unclear contracts**: Does this method handle empty input or not?

## Symptoms

- `InvalidOperationException: Sequence contains no elements`
- `.FirstOrDefault()` followed by null checks
- Guard clauses like `if (!list.Any()) throw ...` at method start
- Comments like `// Assumes non-empty list`
- Unit tests specifically for empty collection edge cases

## Refinement at System Boundaries

Parse into refined types at the edge, use guaranteed types inside:

```csharp
public class DataImporter
{
    // Boundary: raw input, returns refined type
    public Result<NonEmptyList<Record>, ImportError> Import(Stream stream)
    {
        var records = ParseCsv(stream);
        
        return NonEmptyList<Record>.TryFromList(records)
            .Match(
                onSome: list => Result<NonEmptyList<Record>, ImportError>.Success(list),
                onNone: () => Result<NonEmptyList<Record>, ImportError>.Failure(
                    new ImportError("File contains no records"))
            );
    }
}

public class ReportService
{
    // Internal: works only with guaranteed non-empty data
    public Report BuildReport(NonEmptyList<Record> records)
    {
        // No defensive checks needed anywhere in here
        var summary = Summarize(records);
        var chart = BuildChart(records);
        return new Report(summary, chart);
    }
}
```

## Benefits

- **Compile-time guarantees**: Invalid structures can't reach unsafe operations
- **Self-documenting APIs**: `NonEmptyList<T>` parameter tells you what's required
- **No defensive checks**: Internal code trusts the refined type
- **Preserved through transformations**: `Select` on `NonEmptyList` returns `NonEmptyList`
- **Single validation point**: Check once at the boundary, trust everywhere else

## See Also

- [Primitive Obsession](./primitive-obsession.md) — refinement is a special case
- [Honest Functions](./honest-functions.md) — refined types make signatures honest
- [Nullability vs. Optionality](./nullability-optionality.md) — `Option` for values that might not exist
- [Static Factory Methods](./static-factory-methods.md) — controlled construction enforces constraints
