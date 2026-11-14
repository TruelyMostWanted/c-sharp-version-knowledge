---
id: cs12-alias-any-type
csharp: 12
tfm: net8.0
status: stable
tags: [using, alias, tuple, array]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/using-directive#using-alias
modernizes: long tuple or array types repeated in signatures
---
# Alias any type (C# 12)

> **Abstract.** A `using` alias can now name **any** type: tuples, arrays, pointers, nullable value types,
> generic instantiations with tuples. `using GridPos = (int X, int Y);` gives a long type a short name.
> An alias is only a name: it adds readability, **not** type safety.

| | |
|---|---|
| Introduced | C# 12 · .NET 8 (`net8.0`) |
| Status | Stable |
| Kind | Syntax (name binding) |
| Runtime support needed | No |
| Audience | Everyday code |
| Modernize category | CASE |

## 1. Motivation

Before C# 12, `using X = ...;` accepted only named types (`using Map = System.Collections.Generic.Dictionary<string, int>;`).
Tuple types such as `(int X, int Y)` or arrays such as `float[]` could not be aliased, so long tuple signatures were repeated everywhere.

## 2. The feature explained

### 2.1 Syntax

```csharp
using GridPos = (int X, int Y);
using HeightField = float[];
using Matrix = double[,];
using MaybeInt = int?;
using Edge = (int From, int To, float Cost);
using unsafe BytePtr = byte*;            // pointer aliases need `unsafe`
global using EntityId = int;             // global alias
```

### 2.2 Semantics and rules

1. The right side may be any type, including tuple, array, pointer, function pointer and nullable value types.
2. **Nullable reference types** cannot be aliased (`using S = string?;` is an error); nullable value types can.
3. Pointer and function-pointer aliases require `using unsafe` and `<AllowUnsafeBlocks>`.
4. An alias is a synonym: `GridPos` and `(int X, int Y)` are the same type; tuple element names come with the alias.
5. File-local unless declared `global using`.
6. Aliases cannot be generic (`using Pair<T> = (T, T);` is not allowed).

## 3. Code examples

### 3.1 Minimal example

```csharp
using GridPos = (int X, int Y);

GridPos p = (3, 4);
Console.WriteLine(p.X);
```

### 3.2 Extended examples

Graph code with edge tuples:

```csharp
using Edge = (int From, int To, float Cost);
using AdjacencyList = System.Collections.Generic.Dictionary<int, System.Collections.Generic.List<(int To, float Cost)>>;

static AdjacencyList Build(IEnumerable<Edge> edges)
{
    var graph = new AdjacencyList();
    foreach (var (from, to, cost) in edges)
    {
        if (!graph.TryGetValue(from, out var list)) graph[from] = list = [];
        list.Add((to, cost));
    }
    return graph;
}
```

Arrays with meaning:

```csharp
using HeightField = float[];
using Palette = Godot.Color[];

HeightField Generate(int size) => new float[size * size];
```

Project-wide aliases (one place):

```csharp
// GlobalUsings.cs
global using EntityId = int;
global using Vector2 = Godot.Vector2;
```

## 4. Before this feature

```csharp
public static Dictionary<int, List<(int To, float Cost)>> Build(IEnumerable<(int From, int To, float Cost)> edges) { ... }
```

## 5. Under the hood

Aliases are erased at compile time. Tuple element names are stored with `[TupleElementNames]` on signatures,
exactly as if the tuple type had been written out.

## 6. Performance and allocation

None.

## 7. Pitfalls

- **No type safety.** `global using EntityId = int;` lets `EntityId` and any `int` mix freely; `Give(itemId, entityId)` still compiles if both are `int`.
  For real distinction use `readonly record struct EntityId(int Value)`.
- Aliases are not visible to other projects; public APIs show the underlying type.

## 8. Concerns and limitations

- Readability can suffer if readers must look up aliases. Keep aliases near their use (file-local) or in `GlobalUsings.cs`.
- IDE tooltips show the underlying type in some contexts.

## 9. When to use: clean-code guidance

### 9.1 Use when
- A tuple or array type is repeated in a file and has a clear meaning.

### 9.2 Avoid when
- You need type safety (use a record struct).
- Public API surface: callers do not see the alias.

### 9.3 Good versus bad

```csharp
// Good: local readability
using Bounds = (Godot.Vector2 Min, Godot.Vector2 Max);

// Bad: pretending to be a distinct type
global using PlayerId = int;
global using ItemId = int;      // PlayerId and ItemId are interchangeable
```

## 10. Related features

- [../10/global-using.md](../10/global-using.md)
- [../10/record-structs.md](../10/record-structs.md): for type-safe wrappers.

## 11. References

- [using alias](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/using-directive#using-alias)
- [Proposal: Alias any type](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-12.0/using-alias-types)
