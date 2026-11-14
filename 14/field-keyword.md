---
id: cs14-field-keyword
csharp: 14
tfm: net10.0
status: stable (preview in C# 13)
tags: [field, property, backing-field, auto-property]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/field
modernizes: private backing field used only by one property with logic
---
# `field` keyword (C# 14)

> **Abstract.** Inside a property accessor, the contextual keyword `field` refers to the property's
> **compiler-synthesized backing field**. Properties with validation, normalization, lazy initialization or change
> notification no longer need a manually declared backing field; one accessor can stay `get;` or `set;` while the other
> has a body.

| | |
|---|---|
| Introduced | C# 14 · .NET 10 (`net10.0`); preview in C# 13 |
| Status | Stable |
| Kind | Syntax (contextual keyword) |
| Runtime support needed | No |
| Audience | Everyday code |
| Modernize category | AUTO (check that no member named `field` exists) |

## 1. Motivation

Auto-properties are concise until you need one line of logic; then you must declare a private field, name it, and
write both accessors. The field is visible to the whole class, so other members can bypass the property's validation.

## 2. The feature explained

### 2.1 Syntax

```csharp
public string Name
{
    get;
    set => field = value?.Trim() ?? throw new ArgumentNullException(nameof(value));
}
```

### 2.2 Semantics and rules

1. `field` is valid only inside property accessors (and `init` accessors); it names the synthesized backing field.
2. Either or both accessors may have bodies; an accessor without a body (`get;`, `set;`) is auto-implemented over the same field.
3. A property initializer (`= value;`) initializes the backing field directly (it does **not** call the setter).
4. The backing field's type is the property type; attributes can target it with `[field: ...]`.
5. If the type has a member named `field`, the keyword wins inside accessors; use `this.field` or `@field` to refer to the member (breaking change on upgrade, warning in C# 13+).
6. Works in classes, structs (struct must still satisfy definite assignment; auto-default applies), and static properties.
7. Nullable analysis: for a non-nullable reference property with a lazy getter (`field ??= ...`), the compiler infers the field's nullability from usage.

## 3. Code examples

### 3.1 Minimal example

```csharp
public string Message
{
    get;
    set => field = value ?? throw new ArgumentNullException(nameof(value));
}
```

### 3.2 Extended examples

Lazy initialization:

```csharp
public SqlConnection Connection => field ??= new SqlConnection(_connectionString);
```

Change notification (MVVM):

```csharp
public string DisplayName
{
    get;
    set
    {
        if (field == value) return;
        field = value;
        OnPropertyChanged();
    }
}
```

Clamping:

```csharp
public float Volume
{
    get;
    set => field = Math.Clamp(value, 0f, 1f);
} = 0.8f;   // initializer writes the field directly
```

Godot exported property with a signal:

```csharp
[Signal] public delegate void HealthChangedEventHandler(int value);

[Export]
public int Health
{
    get;
    set
    {
        value = Math.Clamp(value, 0, MaxHealth);
        if (field == value) return;
        field = value;
        EmitSignal(SignalName.HealthChanged, value);
    }
}
```

Attribute on the backing field:

```csharp
[field: NonSerialized]
public Cache? Cache { get => field; set => field = value; }
```

## 4. Before this feature

```csharp
private string _message = "";

public string Message
{
    get => _message;
    set => _message = value ?? throw new ArgumentNullException(nameof(value));
}
```

## 5. Under the hood

The property gets an ordinary private backing field, named like an auto-property's (`<Message>k__BackingField`),
and `field` in the accessors compiles to loads and stores of that field:

```csharp
[CompilerGenerated] private string <Message>k__BackingField;

public string Message
{
    get => <Message>k__BackingField;
    set => <Message>k__BackingField = value ?? throw new ArgumentNullException(nameof(value));
}
```

## 6. Performance and allocation

Identical to the hand-written version.

## 7. Pitfalls

- **Name collision:** existing members called `field` change meaning inside accessors when upgrading to C# 14. Rename them first (see [../11/warning-wave-7.md](../11/warning-wave-7.md)).
- The initializer bypasses the setter: validation and notifications do not run for the initial value.
- Godot: `[Export]` setters also run when the scene loads (before `_Ready`); guard against side effects such as signals to nodes that are not ready (`if (!IsNodeReady()) { field = value; return; }`).
- Serializers that look for private fields by name will see `<Name>k__BackingField`.

## 8. Concerns and limitations

- Other members of the class cannot access the backing field (by design). If they must, keep an explicit field.
- `field` in a property with only a getter body and no setter makes the field effectively write-once via initializer or getter logic.

## 9. When to use: clean-code guidance

### 9.1 Use when
- A property needs light logic on get or set and nothing else uses the backing field.

### 9.2 Avoid when
- Several members need direct access to the stored value, or the logic is complex enough to deserve its own method.

### 9.3 Good versus bad

```csharp
// Good
public int Level { get; set => field = Math.Max(1, value); } = 1;

// Bad: business logic and I/O inside a setter
public string SavePath { get; set { field = value; File.WriteAllText(value, Serialize()); } }
```

## 10. Related features

- [../13/field-keyword-preview.md](../13/field-keyword-preview.md)
- [../13/partial-properties-and-indexers.md](../13/partial-properties-and-indexers.md): `field` in implementing declarations.
- [../11/warning-wave-7.md](../11/warning-wave-7.md)

## 11. References

- [field keyword](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/field)
- [What's new in C# 14](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-14)
- [Proposal: field keyword](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-14.0/field-keyword)
