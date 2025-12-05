# Temporal Safety (The Timezone Trap)

> Using `DateTime` with its hidden `Kind` property—use `DateTimeOffset` or dedicated temporal types to make time zone handling explicit.

## Problem

In C#, `DateTime` is a struct that carries a hidden "Kind" (`Utc`, `Local`, or `Unspecified`). This is dynamic typing hidden inside a struct. Arithmetic operations between different Kinds often succeed silently but produce incorrect results.

## The Hidden Danger

```csharp
DateTime utcTime = DateTime.UtcNow;           // Kind = Utc
DateTime localTime = DateTime.Now;             // Kind = Local
DateTime parsedTime = DateTime.Parse("2024-03-15 10:00:00");  // Kind = Unspecified

// What timezone is this? Nobody knows.
TimeSpan difference = localTime - parsedTime;  // Silently wrong if timezones differ

// This "works" but is probably a bug
if (utcTime > parsedTime)  // Comparing Utc to Unspecified—meaningless
{
    // ...
}
```

## Example

### ❌ Before

```csharp
public class EventService
{
    public void ScheduleEvent(string title, DateTime startTime)
    {
        // What timezone is startTime in? We have no idea.
        var evt = new Event
        {
            Title = title,
            StartTime = startTime,  // Stored as-is—what does it mean?
            CreatedAt = DateTime.Now  // Local time on this server
        };
        
        _repository.Save(evt);
    }

    public bool IsEventStartingSoon(Event evt)
    {
        // Comparing potentially different Kinds
        return evt.StartTime < DateTime.Now.AddMinutes(30);  // Bug if evt.StartTime is UTC
    }

    public TimeSpan TimeUntilEvent(Event evt)
    {
        // Subtracting different Kinds—silently wrong
        return evt.StartTime - DateTime.UtcNow;
    }
}

// Parse from user input—what timezone?
var userInput = "2024-12-25 09:00:00";
var eventTime = DateTime.Parse(userInput);  // Kind = Unspecified 🤷
```

### ✅ After

```csharp
// Option 1: Use DateTimeOffset everywhere
public class EventService
{
    public void ScheduleEvent(string title, DateTimeOffset startTime)
    {
        // DateTimeOffset carries the offset—no ambiguity
        var evt = new Event
        {
            Title = title,
            StartTime = startTime,
            CreatedAt = DateTimeOffset.UtcNow
        };
        
        _repository.Save(evt);
    }

    public bool IsEventStartingSoon(Event evt)
    {
        // Both are DateTimeOffset—comparison is correct
        return evt.StartTime < DateTimeOffset.UtcNow.AddMinutes(30);
    }

    public TimeSpan TimeUntilEvent(Event evt)
    {
        // Arithmetic is correct because offsets are considered
        return evt.StartTime - DateTimeOffset.UtcNow;
    }
}

// Option 2: Dedicated temporal types for different concepts
public readonly record struct UtcDateTime
{
    public DateTime Value { get; }
    
    private UtcDateTime(DateTime value) => Value = value;
    
    public static UtcDateTime Now => new(DateTime.UtcNow);
    
    public static UtcDateTime FromUtc(DateTime dt)
    {
        if (dt.Kind != DateTimeKind.Utc)
            throw new ArgumentException("DateTime must be UTC", nameof(dt));
        return new UtcDateTime(dt);
    }
    
    public static UtcDateTime FromOffset(DateTimeOffset dto) 
        => new(dto.UtcDateTime);
    
    public DateTimeOffset ToOffset() => new(Value, TimeSpan.Zero);
    
    public static TimeSpan operator -(UtcDateTime a, UtcDateTime b) 
        => a.Value - b.Value;
    
    public static bool operator <(UtcDateTime a, UtcDateTime b) 
        => a.Value < b.Value;
    
    public static bool operator >(UtcDateTime a, UtcDateTime b) 
        => a.Value > b.Value;
}

public readonly record struct LocalDateTime
{
    public DateTime Value { get; }
    public TimeZoneInfo TimeZone { get; }
    
    private LocalDateTime(DateTime value, TimeZoneInfo tz)
    {
        Value = value;
        TimeZone = tz;
    }
    
    public static LocalDateTime Now(TimeZoneInfo tz) 
        => new(TimeZoneInfo.ConvertTimeFromUtc(DateTime.UtcNow, tz), tz);
    
    public UtcDateTime ToUtc() 
        => UtcDateTime.FromUtc(TimeZoneInfo.ConvertTimeToUtc(Value, TimeZone));
}

// Usage: types prevent mixing
public class EventService
{
    public void ScheduleEvent(string title, UtcDateTime startTime)
    {
        // Can only pass UtcDateTime—no ambiguity
        var evt = new Event
        {
            Title = title,
            StartTimeUtc = startTime,
            CreatedAtUtc = UtcDateTime.Now
        };
        
        _repository.Save(evt);
    }

    public bool IsEventStartingSoon(Event evt)
    {
        // Both are UtcDateTime—can't accidentally mix
        return evt.StartTimeUtc < UtcDateTime.Now + TimeSpan.FromMinutes(30);
    }
}
```

## DateTimeOffset vs DateTime

| Aspect | `DateTime` | `DateTimeOffset` |
|--------|-----------|------------------|
| Stores offset? | No (just Kind enum) | Yes (actual offset) |
| Comparison | Ignores Kind—often wrong | Uses offset—always correct |
| Serialization | Loses Kind in JSON/DB | Preserves offset |
| Default | Unspecified (dangerous) | Local offset (safer) |

**Rule**: Use `DateTimeOffset` for points in time. Use `DateTime` only for calendar concepts without timezone (birthdays, holidays).

## Common Traps

### Trap 1: Database Round-Trip

```csharp
// Storing DateTime
entity.CreatedAt = DateTime.UtcNow;  // Kind = Utc
await _db.SaveChangesAsync();

// Loading—Kind is lost!
var loaded = await _db.FindAsync(id);
Console.WriteLine(loaded.CreatedAt.Kind);  // Unspecified! Not Utc!

// Now this comparison is wrong
if (loaded.CreatedAt < DateTime.UtcNow) { }  // Comparing Unspecified to Utc
```

**Fix**: Use `DateTimeOffset` in your entity, or always re-specify Kind after loading.

### Trap 2: JSON Serialization

```csharp
var dto = new EventDto { StartTime = DateTime.UtcNow };
var json = JsonSerializer.Serialize(dto);
// "2024-12-05T10:30:00Z" — note the Z

var parsed = JsonSerializer.Deserialize<EventDto>(json);
Console.WriteLine(parsed.StartTime.Kind);  // Could be Local or Utc depending on settings!
```

**Fix**: Use `DateTimeOffset` or configure serializer explicitly.

### Trap 3: Daylight Saving Time

```csharp
// "Spring forward" day—2:30 AM doesn't exist!
var localTime = new DateTime(2024, 3, 10, 2, 30, 0, DateTimeKind.Local);

// "Fall back" day—1:30 AM exists twice!
var ambiguousTime = new DateTime(2024, 11, 3, 1, 30, 0, DateTimeKind.Local);

// DateTimeOffset handles this correctly
var offset = new DateTimeOffset(2024, 11, 3, 1, 30, 0, TimeSpan.FromHours(-4));  // EDT
var offset2 = new DateTimeOffset(2024, 11, 3, 1, 30, 0, TimeSpan.FromHours(-5)); // EST
// These are different instants!
```

### Trap 4: Arithmetic Across Kinds

```csharp
DateTime utc = DateTime.UtcNow;
DateTime local = DateTime.Now;

// This "works" but is wrong—should throw or require conversion
TimeSpan diff = local - utc;  // Ignores that they're different timezones!
```

## Why It's a Problem

- **Silent bugs**: Mixing `DateTime` Kinds compiles and runs, but produces wrong results
- **Lost information**: `Kind` is lost during serialization and database round-trips
- **Implicit conversions**: `DateTime` silently converts between Kinds in many operations
- **DST ambiguity**: Local times can be invalid or ambiguous during DST transitions
- **Server location dependency**: `DateTime.Now` depends on server timezone

## Symptoms

- Off-by-N-hours bugs that only appear in production (different server timezone)
- Scheduled events firing at wrong times
- Date filters including or excluding wrong records
- "Works on my machine" issues with time-based logic
- Logs showing times that don't match user expectations

## Guidelines

1. **Store UTC**: All persisted times should be UTC (or `DateTimeOffset`)
2. **Convert at boundaries**: Convert to local time only for display
3. **Use `DateTimeOffset`**: Prefer over `DateTime` for points in time
4. **Explicit timezone**: Always specify timezone when converting
5. **Test across timezones**: Run tests in different timezone configurations

```csharp
// Good pattern: accept offset, store UTC, display local
public class EventController
{
    public IActionResult Create(CreateEventRequest request)
    {
        // Input: DateTimeOffset from client (carries their timezone)
        DateTimeOffset clientTime = request.StartTime;
        
        // Store: Convert to UTC for persistence
        var entity = new Event { StartTimeUtc = clientTime.UtcDateTime };
        
        // Display: Convert to user's timezone
        var userTz = GetUserTimeZone();
        var displayTime = TimeZoneInfo.ConvertTime(clientTime, userTz);
        
        return Ok(new { LocalTime = displayTime });
    }
}
```

## Benefits

- **Explicit offsets**: `DateTimeOffset` makes timezone information visible
- **Correct arithmetic**: Operations account for timezone differences
- **Serialization-safe**: Offset survives JSON/database round-trips
- **DST-aware**: Handles daylight saving transitions correctly
- **No hidden state**: Types like `UtcDateTime` make the semantics obvious

## See Also

- [Primitive Obsession](./primitive-obsession.md) — `DateTime` is a primitive hiding complexity
- [Boolean Blindness](./boolean-blindness.md) — `DateTimeKind` is boolean-blindness with 3 values
- [Ghost States](./ghost-states.md) — `Unspecified` is a ghost state
