# Flag Arguments

> Boolean or enum parameters that control which code path a method takes.

## Problem

Flag arguments hide multiple behaviors behind a single method, violating the single responsibility principle. They lead to combinatorial explosion in testing, make call sites unclear, and prevent extension without modification.

## Example

### ❌ Before

```csharp
public class MessageBody
{
    public string Body { get; private set; } = "";

    public void AddLine(string line, bool isHtml)
    {
        if (isHtml)
        {
            Body += $"{line}</br>";
        }
        else
        {
            Body += line + "\n";
        }
    }

    public void AddEmphaticLine(string line, bool isHtml)
    {
        if (isHtml)
        {
            Body += $"<b>{line}</b></br>";
        }
        else
        {
            Body += $"{line.ToUpper()}\n";
        }
    }
}

// Unclear at call site - what does 'true' mean?
message.AddLine("Hello", true);
message.AddEmphaticLine("Warning!", false);
```

### ✅ After

```csharp
public interface IMessageBody
{
    string Body { get; }
    void AddLine(string line);
    void AddEmphaticLine(string line);
}

public sealed class HtmlMessageBody : IMessageBody
{
    public string Body { get; private set; } = "";

    public void AddLine(string line) => Body += $"{line}</br>";
    public void AddEmphaticLine(string line) => Body += $"<b>{line}</b></br>";
}

public sealed class PlainTextMessageBody : IMessageBody
{
    public string Body { get; private set; } = "";

    public void AddLine(string line) => Body += line + "\n";
    public void AddEmphaticLine(string line) => Body += $"{line.ToUpper()}\n";
}

// Self-documenting at call site
var html = new HtmlMessageBody();
html.AddLine("Hello");

var plain = new PlainTextMessageBody();
plain.AddEmphaticLine("Warning!");
```

## Why It's a Problem

1. **Open-closed violation**: Adding a new format requires modifying all existing methods.
    ```csharp
    // Want Markdown support? Update every method signature and add more if-branches
    void AddLine(string line, bool isHtml, bool isMarkdown)
    ```

2. **Combinatorial explosion**: Every flag doubles the test cases required.
    ```csharp
    // What should this do?
    message.AddLine("Hi", isHtml: true, isMarkdown: true);
    ```

3. **Unclear call sites**: Readers must check the implementation to understand what flags mean.

## Symptoms

- Boolean or enum parameters controlling method behavior
- Method calls where you need to look at the implementation to understand what flags do
- Multiple boolean flags that could conflict or create invalid combinations
- If-else chains based on boolean parameters

## Benefits

- **Self-documenting call sites**: `new HtmlMessageBody()` is clearer than `isHtml: true`
- **Invalid combinations prevented**: Can't accidentally mix incompatible options
- **Open for extension**: Add `MarkdownMessageBody` without touching existing code
- **Additive testing**: Test each implementation independently, not every combination

## See Also

- [Primitive Obsession](./primitive-obsession.md)
- [Static Factory Methods](./static-factory-methods.md)
