# Units of Measure (Dimensional Analysis in Types)

> Mixing incompatible units (meters + feet, seconds + milliseconds) causes calculation errors—encode units in the type system to prevent dimensional mistakes.

## Problem

Numeric values representing physical quantities lack unit information. Adding meters to feet, passing seconds where milliseconds expected, or returning Celsius where Fahrenheit expected all compile successfully but produce wrong results.

## Example

### ❌ Before

```csharp
public class Physics
{
    public double CalculateVelocity(double distance, double time)
    {
        // Is distance in meters or feet? Is time in seconds or milliseconds?
        return distance / time;
    }
    
    public double ConvertToMeters(double value, string unit)
    {
        // Runtime string matching
        return unit switch
        {
            "feet" => value * 0.3048,
            "inches" => value * 0.0254,
            _ => value  // Assume meters?
        };
    }
}

// Problems: Can mix units accidentally
var distance = 100;  // meters? feet?
var time = 5;  // seconds? milliseconds?
var velocity = CalculateVelocity(distance, time);  // What units is this?
```

### ✅ After

```csharp
/// <summary>
/// Marker interface for all units of measure.
/// </summary>
public interface IUnit { }

// Length units
public interface ILength : IUnit { }
public interface IMeters : ILength { }
public interface IFeet : ILength { }
public interface IInches : ILength { }

// Time units
public interface ITime : IUnit { }
public interface ISeconds : ITime { }
public interface IMilliseconds : ITime { }

// Velocity units (derived from Length/Time)
public interface IVelocity : IUnit { }
public interface IMetersPerSecond : IVelocity { }

/// <summary>
/// Quantity with units encoded in type parameter.
/// </summary>
public readonly record struct Quantity<TUnit>(double Value) where TUnit : IUnit
{
    public static Quantity<TUnit> operator +(Quantity<TUnit> left, Quantity<TUnit> right)
        => new(left.Value + right.Value);
    
    public static Quantity<TUnit> operator -(Quantity<TUnit> left, Quantity<TUnit> right)
        => new(left.Value - right.Value);
    
    public static Quantity<TUnit> operator *(Quantity<TUnit> quantity, double scalar)
        => new(quantity.Value * scalar);
    
    public override string ToString() => $"{Value} {typeof(TUnit).Name}";
}

// Type aliases for readability
public using Distance = Quantity<IMeters>;
public using DistanceInFeet = Quantity<IFeet>;
public using Duration = Quantity<ISeconds>;
public using DurationInMs = Quantity<IMilliseconds>;
public using Velocity = Quantity<IMetersPerSecond>;

public static class DistanceExtensions
{
    public static Distance Meters(this double value) => new(value);
    public static Distance Meters(this int value) => new(value);
    
    public static DistanceInFeet Feet(this double value) => new(value);
    public static DistanceInFeet Feet(this int value) => new(value);
    
    public static Distance ToMeters(this DistanceInFeet feet)
        => new(feet.Value * 0.3048);
    
    public static DistanceInFeet ToFeet(this Distance meters)
        => new(meters.Value / 0.3048);
}

public static class TimeExtensions
{
    public static Duration Seconds(this double value) => new(value);
    public static Duration Seconds(this int value) => new(value);
    
    public static DurationInMs Milliseconds(this double value) => new(value);
    public static DurationInMs Milliseconds(this int value) => new(value);
    
    public static Duration ToSeconds(this DurationInMs ms)
        => new(ms.Value / 1000.0);
}

public static class Physics
{
    // Type signature documents units
    public static Velocity CalculateVelocity(Distance distance, Duration time)
    {
        return new Velocity(distance.Value / time.Value);
    }
}

// Usage
var distance = 100.Meters();
var distanceInFeet = 300.Feet();
var time = 5.Seconds();

// Type-safe: units documented in types
var velocity = Physics.CalculateVelocity(distance, time);
Console.WriteLine(velocity);  // "20 IMetersPerSecond"

// Convert units explicitly
var totalDistance = distance + distanceInFeet.ToMeters();

// Won't compile: type mismatch
// var bad = distance + distanceInFeet;  // Error: Cannot add IMeters + IFeet
// var bad2 = distance + time;  // Error: Cannot add ILength + ITime
```

## F#-Style Units of Measure (Approximation)

```csharp
// Using generic math for better type inference (C# 11+)
public readonly record struct Measure<TUnit, T>(T Value) 
    where TUnit : IUnit
    where T : INumber<T>
{
    public static Measure<TUnit, T> operator +(Measure<TUnit, T> a, Measure<TUnit, T> b)
        => new(a.Value + b.Value);
    
    public static Measure<TUnit, T> operator -(Measure<TUnit, T> a, Measure<TUnit, T> b)
        => new(a.Value - b.Value);
    
    public static Measure<TUnit, T> operator *(Measure<TUnit, T> a, T scalar)
        => new(a.Value * scalar);
    
    public static Measure<TUnit, T> operator /(Measure<TUnit, T> a, T scalar)
        => new(a.Value / scalar);
}

// Inferred units from division
public static Measure<IMetersPerSecond, double> operator /(
    Measure<IMeters, double> distance,
    Measure<ISeconds, double> time)
{
    return new(distance.Value / time.Value);
}
```

## Temperature (Non-Linear Conversions)

```csharp
public interface ITemperature : IUnit { }
public interface ICelsius : ITemperature { }
public interface IFahrenheit : ITemperature { }
public interface IKelvin : ITemperature { }

public using Temperature = Quantity<ICelsius>;
public using TemperatureF = Quantity<IFahrenheit>;
public using TemperatureK = Quantity<IKelvin>;

public static class TemperatureExtensions
{
    public static Temperature Celsius(this double value) => new(value);
    public static TemperatureF Fahrenheit(this double value) => new(value);
    public static TemperatureK Kelvin(this double value) => new(value);
    
    public static TemperatureF ToFahrenheit(this Temperature celsius)
        => new(celsius.Value * 9.0 / 5.0 + 32);
    
    public static Temperature ToCelsius(this TemperatureF fahrenheit)
        => new((fahrenheit.Value - 32) * 5.0 / 9.0);
    
    public static TemperatureK ToKelvin(this Temperature celsius)
        => new(celsius.Value + 273.15);
}
```

## Why It's a Problem

1. **Unit confusion**: No indication of what unit a value represents
2. **Calculation errors**: Mixing incompatible units produces wrong results
3. **API ambiguity**: Method parameters don't document expected units
4. **Conversion bugs**: Forgetting unit conversions

## Symptoms

- Variables named with unit suffixes (`distanceInMeters`, `timeInMs`)
- Comments documenting units (`// in meters`)
- Runtime string-based unit tracking
- Conversion bugs in production

## Benefits

- **Compile-time safety**: Cannot mix incompatible units
- **Self-documenting**: Type signature shows units
- **Explicit conversions**: Must convert explicitly
- **Dimensional analysis**: Type system enforces physics

## See Also

- [Strongly Typed IDs](./strongly-typed-ids.md) — type safety for identifiers
- [Primitive Obsession](./primitive-obsession.md) — avoiding raw primitives
- [Branded Primitives](./branded-primitives.md) — nominal typing
