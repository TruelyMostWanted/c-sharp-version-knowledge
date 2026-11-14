---
id: cs13-partial-properties-and-indexers
csharp: 13
tfm: net9.0
status: stable
tags: [partial, property, indexer, source-generator]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/partial-member
related: ../14/partial-events-and-constructors.md
---
# Partial properties and indexers (C# 13)

> **Abstract.** Properties and indexers can be `partial`, following the model of partial methods: one **declaring**
> declaration (no bodies) and one **implementing** declaration (with bodies), with matching signatures. The main
> use: a developer declares a property and a **source generator** implements it (change notification, validation,
> registry lookups, regex), or the reverse.

| | |
|---|---|
| Introduced | C# 13 · .NET 9 (`net9.0`) |
| Status | Stable |
| Kind | Syntax |
| Runtime support needed | No |
| Audience | Source-generator authors and their users |
| Modernize category | n/a |

## 1. Motivation

Generators such as the MVVM Toolkit had to work from **fields** marked with attributes (`[ObservableProperty] private string _name;`)
and generate the property, because properties could not be split across files. Users then wrote code against a
property that did not exist in their source, and attributes on the property had to be forwarded awkwardly.
`[GeneratedRegex]` had the same problem and could only target partial **methods**.

## 2. The feature explained

### 2.1 Syntax

```csharp
public partial class C
{
    public partial string Name { get; set; }        // declaring: accessors without bodies
}

public partial class C
{
    public partial string Name { get => ...; set => ...; }   // implementing
}
```

### 2.2 Semantics and rules

1. Exactly one declaring and one implementing declaration.
2. A declaration whose accessors all lack bodies is the **declaring** one; it looks like an auto-property but is not.
3. Signatures must match: type, name, accessors (`get`, `set`, `init`), modifiers, accessibility; nullable annotations and parameter names (for indexers) should match (warnings otherwise).
4. The implementing declaration **cannot** be an auto-property in C# 13; in C# 14 it may use the `field` keyword.
5. Attributes from both declarations are merged.
6. Partial indexers follow the same rules (`public partial int this[int i] { get; }`).

## 3. Code examples

### 3.1 Minimal example

```csharp
public partial class C
{
    public partial string Name { get; set; }        // declaring (no body)
}

public partial class C
{
    private string _name = "";
    public partial string Name                      // implementing
    {
        get => _name;
        set => _name = value;
    }
}
```

### 3.2 Extended examples

User side with an MVVM-style generator:

```csharp
public partial class PlayerViewModel : ObservableObject
{
    [ObservableProperty]
    public partial string Name { get; set; }

    [ObservableProperty]
    public partial int Level { get; set; }
}
```

Generated side (sketch):

```csharp
partial class PlayerViewModel
{
    private string _name = "";
    public partial string Name
    {
        get => _name;
        set { if (SetProperty(ref _name, value)) OnNameChanged(value); }
    }
    partial void OnNameChanged(string value);
}
```

`[GeneratedRegex]` on a property (.NET 9):

```csharp
public static partial class Patterns
{
    [GeneratedRegex(@"^[a-z_][a-z0-9_]*$")]
    public static partial Regex Identifier { get; }
}
```

Partial indexer:

```csharp
public partial class StatTable
{
    public partial int this[StatId id] { get; set; }
}
```

## 4. Before this feature

```csharp
public partial class PlayerViewModel : ObservableObject
{
    [ObservableProperty]
    private string _name = "";   // field; property "Name" is generated and invisible in source
}
```

## 5. Under the hood

The compiler merges both declarations into **one** ordinary property in metadata, with the implementing accessors
and the union of attributes. There is no trace of partiality in IL.

## 6. Performance and allocation

None.

## 7. Pitfalls

- Forgetting the implementing part gives an error (unlike partial void methods, partial properties always need an implementation).
- A declaring declaration looks like an auto-property; readers may not notice the `partial` keyword.
- Signature mismatches in nullability produce warnings that are easy to ignore; keep them identical.

## 8. Concerns and limitations

- Requires a C# 13 compiler for both the user code and the generator output.
- Godot: Godot's source generators work on `[Export]` properties; partial properties are not required, but can be combined with your own generators.

## 9. When to use: clean-code guidance

### 9.1 Use when
- A generator implements properties or indexers declared by users.

### 9.2 Avoid when
- Hand-written code split across files without a generator: put the property in one place.

### 9.3 Good versus bad

```csharp
// Good: generator-backed
[ObservableProperty] public partial string Title { get; set; }

// Bad: manual split with no reason
partial class A { public partial int X { get; set; } }
partial class A { public partial int X { get => _x; set => _x = value; } private int _x; }
```

## 10. Related features

- [../14/partial-events-and-constructors.md](../14/partial-events-and-constructors.md)
- [../14/field-keyword.md](../14/field-keyword.md)
- [../11/file-local-types.md](../11/file-local-types.md)

## 11. References

- [Partial members](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/partial-member)
- [Proposal: Partial properties](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-13.0/partial-properties)
