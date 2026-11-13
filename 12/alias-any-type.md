---
id: cs12-alias-any-type
csharp: 12
tfm: net8.0
status: stable
tags: [using, alias, tuple, array]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-12
---
# Alias any type (C# 12)

`using` alias directives can name any type: tuples, arrays, pointers and other unsafe types, not only named types.

```csharp
using GridPos = (int X, int Y);
using HeightField = float[];
global using EntityId = int;     // global alias: whole project

GridPos p = (3, 4);
```

## Pitfalls
- An alias is not a new type: `EntityId` and `int` are interchangeable, so it gives readability, not type safety.
  For type safety use a `readonly record struct EntityId(int Value)`.
- Pointer aliases need `using unsafe`.
