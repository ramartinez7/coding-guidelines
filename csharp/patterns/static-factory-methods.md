# Static Factory Methods

> Use static methods instead of constructors to create instances with clear, descriptive names.

## Problem

Constructors have limitations: they must match the class name, can't have descriptive names, and multiple constructors with similar parameter types become ambiguous. This makes the code harder to read and maintain.

## Example

### ❌ Before

```csharp
public sealed class Temperature
{
    public double Kelvin { get; }

    public Temperature(double kelvin)
    {
        Kelvin = kelvin;
    }

    // Ambiguous: Is this Celsius, Fahrenheit, or Kelvin?
    public Temperature(double value, bool isCelsius)
    {
        Kelvin = isCelsius ? value + 273.15 : (value - 32) * 5 / 9 + 273.15;
    }
}

// Confusing at call site
var temp1 = new Temperature(100);           // What unit?
var temp2 = new Temperature(100, true);     // Have to remember what 'true' means
var temp3 = new Temperature(100, false);    // Easy to get wrong
```

### ✅ After

```csharp
public sealed class Temperature
{
    public double Kelvin { get; }

    private Temperature(double kelvin)
    {
        Kelvin = kelvin;
    }

    public static Temperature FromKelvin(double kelvin) => new(kelvin);
    public static Temperature FromCelsius(double celsius) => new(celsius + 273.15);
    public static Temperature FromFahrenheit(double fahrenheit) => new((fahrenheit - 32) * 5 / 9 + 273.15);
}

// Crystal clear at call site
var temp1 = Temperature.FromKelvin(373.15);
var temp2 = Temperature.FromCelsius(100);
var temp3 = Temperature.FromFahrenheit(212);
```

## When to Use

- **Multiple creation paths**: When an object can be created from different inputs (e.g., `FromJson`, `FromXml`, `FromFile`)
- **Descriptive naming**: When the constructor parameters don't clearly describe what's being created
- **Validation with failure**: When creation can fail and you want to return `null` or a `Result<T>` instead of throwing
- **Caching/pooling**: When you might want to return an existing instance instead of creating a new one
- **Async creation**: When initialization requires async operations (constructors can't be async)

## Symptoms That Suggest This Pattern

- Boolean or enum parameters that change how the object is constructed
- Multiple constructors with similar parameter lists
- Comments needed to explain what constructor parameters mean
- `// TODO: make this clearer` near `new` calls

## Benefits

- **Self-documenting code**: Method names describe what's being created
- **Flexibility**: Can return subtypes, cached instances, or null
- **Validation options**: Can use `TryCreate` pattern to avoid exceptions
- **Async support**: Static methods can be async; constructors cannot

## Variations

### TryCreate Pattern

```csharp
public static bool TryFromCelsius(double celsius, [NotNullWhen(true)] out Temperature? result)
{
    if (celsius < -273.15)
    {
        result = null;
        return false;
    }
    result = new Temperature(celsius + 273.15);
    return true;
}
```

### Result Pattern

```csharp
public static Result<Temperature> FromCelsius(double celsius)
{
    if (celsius < -273.15)
        return Result.Failure<Temperature>("Temperature cannot be below absolute zero");
    
    return Result.Success(new Temperature(celsius + 273.15));
}
```

## See Also

- [Primitive Obsession](./primitive-obsession.md)
- [Data Clumps](./data-clump.md)
