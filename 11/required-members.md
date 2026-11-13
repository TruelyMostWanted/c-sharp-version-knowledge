---
id: cs11-required-members
csharp: 11
tfm: net7.0
status: stable
tags: [required, init, object-initializer, dto, config]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/required
modernizes: large constructors only to enforce mandatory values; runtime null checks on DTOs
---
# Required members (C# 11)

`required` on a property or field forces callers to set it in an object initializer.

```csharp
public sealed class UnitConfig
{
    public required string Id { get; init; }
    public required int MaxHealth { get; init; }
    public float Speed { get; init; } = 3f;
}

var archer = new UnitConfig { Id = "archer", MaxHealth = 40 };   // missing Id or MaxHealth -> compile error
```

## Rules
- A constructor marked `[SetsRequiredMembers]` tells the compiler that it sets all required members.
- Required members must be at least as visible as the containing type.
- System.Text.Json (.NET 7+) treats `required` properties as required during deserialization.

## Pitfalls
- Godot: `GodotObject`-derived types (`Resource`, `Node`) are created by the engine without an initializer;
  do not use `required` on them. Use it on plain C# config and DTO types.
