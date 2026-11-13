---
id: cs10-struct-improvements
csharp: 10
tfm: net6.0
status: stable
tags: [struct, constructor, initializer, with]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/struct
---
# Improvements of structure types (C# 10)

- Explicit **parameterless constructors** in structs.
- **Field and property initializers** in structs.
- `with` expressions work on any struct (and anonymous types), not only records.

```csharp
public struct Spawner
{
    public float Interval { get; set; } = 1.5f;
    public int Max = 10;
    public Spawner() { }   // needed so the initializers run for `new Spawner()`
}

var fast = new Spawner() with { Interval = 0.5f };
```

## Pitfalls
- `default(Spawner)` and array elements (`new Spawner[100]`) do **not** run constructors or initializers: all fields are zero.
  Do not rely on initializers for structs stored in arrays (SoA/AoS buffers).
