---
id: cs14-field-keyword
csharp: 14
tfm: net10.0
status: stable (preview in C# 13)
tags: [field, property, backing-field, auto-property]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-14
---
# `field` keyword (C# 14)

Inside a property accessor, `field` refers to the compiler-synthesized backing field.
Validation, normalization, lazy init or change notification no longer need a manual backing field.

## Syntax
```csharp
public string Message
{
    get;
    set => field = value ?? throw new ArgumentNullException(nameof(value));
}

public SqlConnection Connection => field ??= new SqlConnection("connection-string");   // lazy

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

## Rules
- You may give a body to one or both accessors.
- If the type already has a member named `field`, use `@field` or `this.field`, or rename it. This can be a breaking change when upgrading.

## Example (Godot)
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
Note: `[Export]` setters also run when the scene loads; guard against side effects before `_Ready` if needed.
