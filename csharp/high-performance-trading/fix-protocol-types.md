# FIX Protocol Types

> Model FIX protocol messages with strongly-typed fields—prevent field tag errors and ensure protocol compliance at compile time.

## Problem

FIX (Financial Information eXchange) protocol uses numeric tags to identify fields, making it error-prone with stringly-typed APIs. Wrong tag numbers, incorrect data types, and missing required fields cause runtime failures. Strongly-typed FIX message models catch errors at compile time and provide IntelliSense.

## Example

### ❌ Before (String-Based Tags)

```csharp
public class FixMessage
{
    private Dictionary<int, string> fields = new();

    public void SetField(int tag, string value)
    {
        fields[tag] = value;
    }

    public string GetField(int tag)
    {
        return fields[tag];  // Runtime error if missing!
    }
}

// Usage: error-prone
var message = new FixMessage();
message.SetField(55, "MSFT");  // What is tag 55?
message.SetField(44, "150.50");  // Is this price or quantity?
var symbol = message.GetField(56);  // Wrong tag! Runtime error
```

### ✅ After (Strongly-Typed Fields)

```csharp
public static class FixTag
{
    public const int Symbol = 55;
    public const int Price = 44;
    public const int OrderQty = 38;
    public const int Side = 54;
    public const int OrderType = 40;
    public const int ClOrdID = 11;
}

public readonly struct Symbol
{
    public string Value { get; }
    public Symbol(string value) => Value = value;
    public override string ToString() => Value;
}

public readonly struct Price
{
    public decimal Value { get; }
    public Price(decimal value) => Value = value;
    public override string ToString() => Value.ToString("F2");
}

public class NewOrderSingle
{
    public Symbol Symbol { get; set; }
    public Price Price { get; set; }
    public long OrderQty { get; set; }
    public Side Side { get; set; }
    public OrderType OrderType { get; set; }
    public string ClOrdID { get; set; } = "";

    public static NewOrderSingle Parse(ReadOnlySpan<byte> data)
    {
        var order = new NewOrderSingle();

        foreach (var field in ParseFields(data))
        {
            switch (field.Tag)
            {
                case FixTag.Symbol:
                    order.Symbol = new Symbol(field.GetString());
                    break;
                case FixTag.Price:
                    order.Price = new Price(field.GetDecimal());
                    break;
                case FixTag.OrderQty:
                    order.OrderQty = field.GetLong();
                    break;
                // ... other fields
            }
        }

        return order;
    }
}

public enum Side : byte
{
    Buy = (byte)'1',
    Sell = (byte)'2'
}

public enum OrderType : byte
{
    Market = (byte)'1',
    Limit = (byte)'2',
    Stop = (byte)'3'
}
```

## Zero-Copy FIX Parser

```csharp
public ref struct FixField
{
    public int Tag { get; }
    public ReadOnlySpan<byte> Value { get; }

    public FixField(int tag, ReadOnlySpan<byte> value)
    {
        Tag = tag;
        Value = value;
    }

    public string GetString()
    {
        return Encoding.ASCII.GetString(Value);
    }

    public decimal GetDecimal()
    {
        Span<char> chars = stackalloc char[Value.Length];
        Encoding.ASCII.GetChars(Value, chars);
        return decimal.Parse(chars);
    }

    public long GetLong()
    {
        long result = 0;
        foreach (var b in Value)
        {
            result = result * 10 + (b - '0');
        }
        return result;
    }
}

public static IEnumerable<FixField> ParseFields(ReadOnlySpan<byte> data)
{
    const byte SOH = 0x01;  // FIX field delimiter

    int start = 0;
    for (int i = 0; i < data.Length; i++)
    {
        if (data[i] == SOH)
        {
            var field = data[start..i];
            var equals = field.IndexOf((byte)'=');

            if (equals > 0)
            {
                var tagSpan = field[..equals];
                var valueSpan = field[(equals + 1)..];

                int tag = ParseTag(tagSpan);
                yield return new FixField(tag, valueSpan);
            }

            start = i + 1;
        }
    }
}

private static int ParseTag(ReadOnlySpan<byte> tagBytes)
{
    int tag = 0;
    foreach (var b in tagBytes)
    {
        tag = tag * 10 + (b - '0');
    }
    return tag;
}
```

## Type-Safe Message Builder

```csharp
public class FixMessageBuilder
{
    private readonly List<(int Tag, string Value)> fields = new();

    public FixMessageBuilder WithSymbol(Symbol symbol)
    {
        fields.Add((FixTag.Symbol, symbol.Value));
        return this;
    }

    public FixMessageBuilder WithPrice(Price price)
    {
        fields.Add((FixTag.Price, price.ToString()));
        return this;
    }

    public FixMessageBuilder WithOrderQty(long qty)
    {
        fields.Add((FixTag.OrderQty, qty.ToString()));
        return this;
    }

    public FixMessageBuilder WithSide(Side side)
    {
        fields.Add((FixTag.Side, ((byte)side).ToString()));
        return this;
    }

    public byte[] Build()
    {
        using var ms = new MemoryStream();
        foreach (var (tag, value) in fields)
        {
            ms.Write(Encoding.ASCII.GetBytes($"{tag}={value}\x01"));
        }
        return ms.ToArray();
    }
}

// Usage
var message = new FixMessageBuilder()
    .WithSymbol(new Symbol("MSFT"))
    .WithPrice(new Price(150.50m))
    .WithOrderQty(1000)
    .WithSide(Side.Buy)
    .Build();
```

## Related Patterns

- [Zero-Copy Techniques](./zero-copy-techniques.md) — Efficient parsing
- [Strongly Typed IDs](../patterns/strongly-typed-ids.md) — Type-safe identifiers
- [Smart Constructors](../patterns/smart-constructors.md) — Validated construction
- [Type-Safe Builder](../patterns/type-safe-builder.md) — Fluent APIs
