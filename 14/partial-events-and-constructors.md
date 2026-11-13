---
id: cs14-partial-events-and-constructors
csharp: 14
tfm: net10.0
status: stable
tags: [partial, constructor, event, source-generator]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-14
related: ../13/partial-properties-and-indexers.md
---
# Partial events and partial constructors (C# 14)

Instance constructors and events can be `partial`: one **defining** and one **implementing** declaration.
Mainly for source generators (user declares, generator implements, or the other way round).

```csharp
public partial class Widget
{
    public partial Widget(string name);              // defining
    public partial event EventHandler? Changed;      // defining: field-like event
}

public partial class Widget
{
    public partial Widget(string name) { Name = name; }        // implementing
    public partial event EventHandler? Changed                // implementing: needs add and remove
    {
        add    { _changed += value; }
        remove { _changed -= value; }
    }
    private EventHandler? _changed;
    public string Name { get; }
}
```

## Rules
- Exactly one defining and one implementing declaration.
- Only the implementing constructor may have an initializer (`: this(...)` / `: base(...)`).
- Only one partial type declaration may use primary constructor syntax.
- The implementing event declaration must have `add` and `remove` accessors.
