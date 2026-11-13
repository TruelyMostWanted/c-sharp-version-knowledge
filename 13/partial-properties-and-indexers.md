---
id: cs13-partial-properties-and-indexers
csharp: 13
tfm: net9.0
status: stable
tags: [partial, property, indexer, source-generator]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-13
related: ../14/partial-events-and-constructors.md
---
# Partial properties and indexers (C# 13)

Same model as partial methods: one **declaring** and one **implementing** declaration with matching signatures.
Typical use: a source generator implements a property the user declares (e.g. change notification).

```csharp
public partial class C
{
    public partial string Name { get; set; }        // declaring (no body)
}

public partial class C
{
    private string _name;
    public partial string Name                      // implementing
    {
        get => _name;
        set => _name = value;
    }
}
```

## Rules
- A property without a body is the declaring declaration.
- An auto-property cannot be the implementing declaration (with C# 14 you can use `field` in the implementation).
