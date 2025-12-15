# Type-Safe Localization (i18n Keys)

> String-based localization keys fail silently when keys are misspelled or missing—use typed resource keys to catch localization errors at compile time.

## Problem

Localization systems that use string keys for translations fail at runtime when keys are mistyped, missing, or removed. The compiler can't verify that all required translations exist or that the correct parameters are provided.

## Example

### ❌ Before

```csharp
public class UserController
{
    private readonly IStringLocalizer _localizer;
    
    public IActionResult Welcome(string name)
    {
        // String keys—no compile-time safety
        var message = _localizer["WelcomeMessge", name];  // Typo!
        return Ok(message);
    }
    
    public IActionResult Error()
    {
        var message = _localizer["ErrorMessage"];
        // Missing parameter—runtime error
        return BadRequest(message);
    }
}
```

### ✅ After

```csharp
public interface ILocalizationKey<T>
{
    string Key { get; }
    T Format(params object[] args);
}

public sealed record WelcomeMessage(string UserName) : ILocalizationKey<string>
{
    public string Key => "Welcome.Message";
    
    public string Format(params object[] args)
    {
        return string.Format("Welcome, {0}!", UserName);
    }
}

public sealed record ErrorMessage(string ErrorCode) : ILocalizationKey<string>
{
    public string Key => "Error.Generic";
    
    public string Format(params object[] args)
    {
        return string.Format("Error: {0}", ErrorCode);
    }
}

public interface ITypedLocalizer
{
    string GetString<T>(T key) where T : ILocalizationKey<string>;
}

public sealed class TypedLocalizer : ITypedLocalizer
{
    private readonly IStringLocalizer _localizer;
    
    public TypedLocalizer(IStringLocalizer localizer)
    {
        _localizer = localizer;
    }
    
    public string GetString<T>(T key) where T : ILocalizationKey<string>
    {
        var template = _localizer[key.Key];
        return key.Format(template);
    }
}

// Usage: Compile-time verified keys
public class UserController
{
    private readonly ITypedLocalizer _localizer;
    
    public IActionResult Welcome(string name)
    {
        var message = _localizer.GetString(new WelcomeMessage(name));
        return Ok(message);
    }
}
```

## See Also

- [Strongly Typed IDs](./strongly-typed-ids.md)
- [Type-Safe Configuration](./type-safe-configuration.md)
