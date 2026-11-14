---
id: cs12-inline-arrays
csharp: 12
tfm: net8.0
status: stable
needs-runtime: true (InlineArrayAttribute, .NET 8)
tags: [inline-array, fixed-buffer, struct, performance]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/struct#inline-arrays
modernizes: unsafe fixed-size buffers in structs
---
# Inline arrays (C# 12)

> **Abstract.** A struct marked `[InlineArray(N)]` with exactly one instance field is laid out by the runtime as
> **N consecutive copies** of that field. The language lets you index it, iterate it and convert it to
> `Span<T>` / `ReadOnlySpan<T>`. Result: a fixed-size buffer inside a struct, without a heap array and without
> `unsafe`, for any element type (including references).

| | |
|---|---|
| Introduced | C# 12 · .NET 8 (`net8.0`) |
| Status | Stable |
| Kind | Runtime layout + language support |
| Runtime support needed | Yes: .NET 8 (`System.Runtime.CompilerServices.InlineArrayAttribute`) |
| Audience | Low-level and performance code; runtime and library authors |
| Modernize category | CASE |

## 1. Motivation

A struct that needs "8 neighbours" or "4 wheel contacts" had three options:

- a reference to a heap array (`int[]`): extra allocation, indirection, nullable;
- `unsafe fixed int buf[8];`: only primitive element types, needs `unsafe`, no bounds checks;
- eight separate fields plus a `switch` indexer: verbose.

## 2. The feature explained

### 2.1 Syntax

```csharp
[InlineArray(8)]
public struct Buffer8<T>
{
    private T _element0;
}
```

### 2.2 Semantics and rules

1. The struct must have exactly **one** instance field; its type is the element type. Length N > 0 is a constant.
2. The struct cannot have an explicit layout and should not declare other instance state.
3. Language support: element access `buffer[i]` (bounds-checked), `buffer[^1]`, ranges `buffer[1..3]` (as span),
   `foreach`, and implicit conversion to `Span<T>` (for variables) and `ReadOnlySpan<T>`.
4. Element type can be anything, including reference types and generic `T`.
5. Inline arrays are value types: assigning copies all N elements.
6. Indexing out of range throws `IndexOutOfRangeException`, as with arrays.

## 3. Code examples

### 3.1 Minimal example

```csharp
[System.Runtime.CompilerServices.InlineArray(10)]
public struct Buffer
{
    private int _element0;
}

var buffer = new Buffer();
for (int i = 0; i < 10; i++) buffer[i] = i;
foreach (var i in buffer) Console.WriteLine(i);
```

### 3.2 Extended examples

Grid neighbours without allocation:

```csharp
[InlineArray(8)]
public struct Neighbours
{
    private Vector2I _e0;
}

public static int GetNeighbours(Vector2I c, ref Neighbours result)
{
    int n = 0;
    for (int dy = -1; dy <= 1; dy++)
    for (int dx = -1; dx <= 1; dx++)
        if (dx != 0 || dy != 0) result[n++] = new Vector2I(c.X + dx, c.Y + dy);
    return n;
}
```

Fixed slots inside a data-oriented entity record:

```csharp
[InlineArray(4)]
public struct AbilitySlots { private int _id0; }

public struct CombatantData
{
    public int Hp;
    public AbilitySlots Abilities;   // stored inline, no extra allocation per entity
}

var entities = new CombatantData[10_000];
entities[42].Abilities[0] = FireballId;
Span<int> slots = entities[42].Abilities;   // span over the inline storage
```

Using span APIs:

```csharp
var buf = new Buffer();
Span<int> span = buf;
span.Fill(-1);
int index = span.IndexOf(5);
```

## 4. Before this feature

```csharp
public unsafe struct Buffer
{
    public fixed int Data[10];             // primitives only, unsafe, no bounds checks
}

public struct Neighbours
{
    public Vector2I N0, N1, N2, N3, N4, N5, N6, N7;
    public Vector2I this[int i] => i switch { 0 => N0, 1 => N1, /* ... */ _ => throw new IndexOutOfRangeException() };
}
```

## 5. Under the hood

The runtime reads `[InlineArray(N)]` and sizes the struct as `N * sizeof(element)` (with alignment).
The compiler lowers element access to helper methods on `<PrivateImplementationDetails>` such as
`InlineArrayElementRef<TBuffer, TElement>(ref buffer, index)` built on `Unsafe.As` / `Unsafe.Add`, plus
`MemoryMarshal.CreateSpan` for span conversions, after a bounds check against N.

The compiler itself uses synthesized inline array types (`<>y__InlineArray3<T>`) for `params ReadOnlySpan<T>` arguments
and collection expressions targeting spans.

## 6. Performance and allocation

- No heap allocation; elements live inside the struct (on the stack, inside an array element, or inside a class).
- Access cost is similar to an array element access with bounds check; the JIT can remove checks in simple loops.
- Large inline arrays make the struct large: copying is expensive. Pass by `ref` / `in`.

## 7. Pitfalls

- Value semantics: `var copy = entity.Abilities;` copies, modifications do not affect the original.
- Converting a **readonly** or temporary inline array to `Span<T>` is not allowed (only `ReadOnlySpan<T>`).
- Returning a span over a local inline array escapes the stack frame (compiler error, by design).
- Do not declare extra fields; the attribute requires exactly one.

## 8. Concerns and limitations

- Requires .NET 8 runtime; not available on older frameworks or Godot builds targeting older runtimes.
- Microsoft positions it mainly for runtime and library authors; most code meets it indirectly through spans.
- Godot: inline arrays are not Variant-compatible; keep them in pure C# data structures (ECS-style component arrays).

## 9. When to use: clean-code guidance

### 9.1 Use when
- Small, fixed-size, frequently accessed data inside structs (neighbours, slots, small vectors).
- Replacing `unsafe fixed` buffers.

### 9.2 Avoid when
- Variable sizes, large N, or when a plain array allocated once is simpler.

### 9.3 Good versus bad

```csharp
// Good: compact, safe, no allocation per entity
[InlineArray(4)] public struct WheelContacts { private ContactPoint _c0; }

// Bad: 4 KB struct copied on every assignment
[InlineArray(1024)] public struct Huge { private float _f0; }
```

## 10. Related features

- [collection-expressions.md](collection-expressions.md)
- [../13/params-collections.md](../13/params-collections.md)
- [../11/ref-fields-scoped.md](../11/ref-fields-scoped.md)

## 11. References

- [Inline arrays](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/struct#inline-arrays)
- [Proposal: Inline arrays](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-12.0/inline-arrays)
