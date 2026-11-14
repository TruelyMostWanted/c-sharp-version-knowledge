---
id: cs13-field-keyword-preview
csharp: 13
tfm: net9.0
status: preview (stable in C# 14)
tags: [field, property, preview]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-13
---
# `field` keyword (preview in C# 13, pointer chapter)

> **Abstract.** In C# 13 (Visual Studio 17.12+, .NET 9 SDK) the contextual keyword `field` inside property accessors
> was a **preview** feature that required `<LangVersion>preview</LangVersion>`. It became stable in C# 14 (.NET 10).
> On a `net9.0` project without preview, write a manual backing field.

| | |
|---|---|
| Introduced | C# 13 (preview) · stable in C# 14 · .NET 10 |
| Status | Preview in C# 13 |
| Kind | Pointer chapter |
| Runtime support needed | No |
| Audience | Everyone on `net9.0` |
| Modernize category | NEVER-AUTO on C# 13 |

## 1. What this means for a C# 13 project

`field` is not available unless the project already opts into preview. The full chapter is
[../14/field-keyword.md](../14/field-keyword.md).

## 2. Opting in (and why you usually should not)

```xml
<LangVersion>preview</LangVersion>
```

- Preview features may change; code is not guaranteed to compile unchanged with the C# 14 compiler.
- Preview applies to the **whole** project and unlocks all other preview features too.

## 3. Code without the feature (what to write instead)

```csharp
private int _health;

public int Health
{
    get => _health;
    set => _health = Math.Clamp(value, 0, MaxHealth);
}

private string? _name;
public string Name => _name ??= LoadName();   // lazy initialization
```

## 4. Migration once the project reaches C# 14

```csharp
public int Health
{
    get;
    set => field = Math.Clamp(value, 0, MaxHealth);
}

public string Name => field ??= LoadName();
```

Before migrating, check for members named `field` (they change meaning inside accessors), see
[../11/warning-wave-7.md](../11/warning-wave-7.md).

## 5. References

- [What's new in C# 13](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-13)
- [field keyword (C# 14)](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/field)
