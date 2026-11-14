---
id: cs14-partial-events-and-constructors
csharp: 14
tfm: net10.0
status: stable
tags: [partial, constructor, event, source-generator]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/partial-member
related: ../13/partial-properties-and-indexers.md
---
# Partial events and partial constructors (C# 14)

> **Abstract.** Instance constructors and events can be declared `partial`: one **defining** declaration and one
> **implementing** declaration. Together with partial methods (C# 3/9) and partial properties/indexers (C# 13), nearly
> every member kind can now be split between user code and a **source generator**, in either direction.

| | |
|---|---|
| Introduced | C# 14 · .NET 10 (`net10.0`) |
| Status | Stable |
| Kind | Syntax |
| Runtime support needed | No |
| Audience | Source-generator authors and their users |
| Modernize category | n/a |

## 1. Motivation

Generators that weave behaviour into constructors (dependency injection, interop marshalling, logging setup) or events
(weak events, thread-safe invocation lists, signal bridges) could not let users **declare** the member and supply the body,
or vice versa. Users had to call generated `Initialize()` methods manually or forgo `event` syntax.

## 2. The feature explained

### 2.1 Syntax

```csharp
partial class Widget
{
    public partial Widget(string name);              // defining constructor
    public partial event EventHandler? Changed;      // defining event (field-like syntax)
}

partial class Widget
{
    public partial Widget(string name) { ... }       // implementing constructor
    public partial event EventHandler? Changed        // implementing event: add and remove required
    {
        add { ... }
        remove { ... }
    }
}
```

### 2.2 Semantics and rules

1. Exactly one defining and one implementing declaration per member.
2. Constructors: only the **implementing** declaration may have a constructor initializer (`: this(...)` / `: base(...)`).
3. Only one partial type declaration may use primary-constructor syntax.
4. Events: the defining declaration uses field-like syntax; the implementing declaration **must** have `add` and `remove` accessors (it is not field-like).
5. Signatures and modifiers must match; attributes are merged.
6. Static constructors cannot be partial.

## 3. Code examples

### 3.1 Minimal example

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

### 3.2 Extended examples

Weak event generated from a user declaration:

```csharp
// user code
public partial class Inventory
{
    [WeakEvent]
    public partial event EventHandler<ItemEventArgs>? ItemAdded;
}

// generated
partial class Inventory
{
    private readonly WeakEventSource<ItemEventArgs> _itemAdded = new();
    public partial event EventHandler<ItemEventArgs>? ItemAdded
    {
        add    => _itemAdded.Subscribe(value);
        remove => _itemAdded.Unsubscribe(value);
    }
}
```

Generator-declared constructor implemented by the user:

```csharp
// generated: the generator knows the required dependencies
partial class EnemySpawner
{
    public partial EnemySpawner(IRandom rng, SpawnTable table);
}

// user code
partial class EnemySpawner
{
    public partial EnemySpawner(IRandom rng, SpawnTable table) : base()
    {
        _rng = rng;
        _table = table;
    }
    private readonly IRandom _rng;
    private readonly SpawnTable _table;
}
```

## 4. Before this feature

```csharp
public partial class Widget
{
    public Widget(string name) { Name = name; InitializeGenerated(); }   // manual hook call
    partial void InitializeGenerated();
}
```

## 5. Under the hood

Both declarations are merged into one ordinary constructor or event in metadata; partiality leaves no trace in IL.

## 6. Performance and allocation

None.

## 7. Pitfalls

- An implementing event must not be field-like; forgetting accessors is an error.
- Placing the constructor initializer on the defining declaration is an error.

## 8. Concerns and limitations

- Readability: the logic of a constructor or event may live in a generated file. Name generated files clearly (`*.g.cs`).
- Godot: Godot's `[Signal]` delegates already generate events; partial events are for your own generators.

## 9. When to use: clean-code guidance

### 9.1 Use when
- Building or using source generators that supply constructor bodies or event accessors.

### 9.2 Avoid when
- Hand-written code with no generator.

### 9.3 Good versus bad

```csharp
// Good: generator implements thread-safe accessors for a declared event
[ThreadSafeEvent] public partial event Action? Tick;

// Bad: splitting a constructor across two hand-written files for no reason
```

## 10. Related features

- [../13/partial-properties-and-indexers.md](../13/partial-properties-and-indexers.md)
- [../12/primary-constructors.md](../12/primary-constructors.md)

## 11. References

- [Partial members](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/partial-member)
- [Proposal: Partial events and constructors](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-14.0/partial-events-and-constructors)
