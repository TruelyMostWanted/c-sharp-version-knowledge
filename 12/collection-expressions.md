---
id: cs12-collection-expressions
csharp: 12
tfm: net8.0
status: stable
analyzer: IDE0300-IDE0305, IDE0028
tags: [collection-expressions, spread, array, list, span]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/collection-expressions
related: ../15/collection-expression-arguments.md
modernizes: new T[] { ... }, new List<T> { ... }, Array.Empty<T>(), ToArray/ToList after concatenation
---
# Collection expressions (C# 12)

> **Abstract.** `[e1, e2, ..other]` creates a collection of whatever type the context expects: arrays, `List<T>`,
> `Span<T>`, `ReadOnlySpan<T>`, `ImmutableArray<T>`, interfaces such as `IEnumerable<T>`, and any type with a
> collection builder. The spread element `..` inlines another collection. One uniform syntax replaces half a dozen
> initialization idioms, and the compiler picks the most efficient construction for the target.

| | |
|---|---|
| Introduced | C# 12 · .NET 8 (`net8.0`) |
| Status | Stable |
| Kind | Syntax + target-typed conversion |
| Runtime support needed | No (some optimizations use .NET 8 APIs such as `CollectionsMarshal.SetCount`) |
| Audience | Everyday code |
| Modernize category | AUTO (IDE0300–IDE0305), except where an interface target's concrete type matters |

## 1. Motivation

Every collection type had its own initialization syntax:

```csharp
int[] a = new int[] { 1, 2, 3 };
int[] b = new[] { 1, 2, 3 };
List<int> c = new List<int> { 1, 2, 3 };
Span<int> d = stackalloc int[] { 1, 2, 3 };
ImmutableArray<int> e = ImmutableArray.Create(1, 2, 3);
int[] empty = Array.Empty<int>();
int[] both = first.Concat(second).ToArray();
```

## 2. The feature explained

### 2.1 Syntax

```csharp
[]                       // empty
[1, 2, 3]                // elements
[.. first, .. second]    // spread
[0, .. middle, 99]       // mixed
```

### 2.2 Semantics and rules

1. A collection expression has **no natural type**; it needs a target type (`var x = [1, 2];` is an error).
2. Supported targets:
   - arrays (`T[]`), `Span<T>`, `ReadOnlySpan<T>`;
   - types with a `[CollectionBuilder]` attribute (`ImmutableArray<T>`, `ImmutableList<T>`, ...);
   - types supporting collection initializers (`IEnumerable` + `Add`), e.g. `List<T>`, `HashSet<T>`, `Dictionary` (C# 12: no `k: v` syntax);
   - interfaces `IEnumerable<T>`, `IReadOnlyCollection<T>`, `IReadOnlyList<T>` (compiler picks a read-only type),
     `ICollection<T>`, `IList<T>` (compiler uses `List<T>`).
3. `..e` spreads all elements of `e`; `e` must be enumerable. Spreads and elements are evaluated left to right.
4. Empty `[]` for arrays and read-only interfaces may use a cached empty instance (`Array.Empty<T>()`).
5. `ReadOnlySpan<T>` targets with constant elements may point to static data (no allocation); `Span<T>` targets may use stack memory (inline arrays).
6. Overload resolution prefers span-based overloads (`ReadOnlySpan<T>` over `T[]`, etc.) when ambiguous.
7. Constructor arguments (capacity, comparer) are not expressible in C# 12; C# 15 adds `with(...)`.

## 3. Code examples

### 3.1 Minimal example

```csharp
int[] a = [1, 2, 3];
List<string> b = ["one", "two", "three"];
Span<char> c = ['a', 'b', 'c'];
int[][] jagged = [[1, 2, 3], [4, 5, 6]];

int[] row0 = [1, 2, 3];
int[] row1 = [4, 5, 6];
int[] single = [.. row0, .. row1];      // spread
List<int> empty = [];
```

### 3.2 Extended examples

Fields and properties:

```csharp
public sealed class Inventory
{
    private readonly List<Item> _items = [];
    public IReadOnlyList<string> DefaultTags { get; } = ["common", "tradable"];
}
```

Combining sources without LINQ:

```csharp
Unit[] everyone = [.. allies, player, .. mercenaries];
```

Method arguments and returns:

```csharp
DrawPolygon([new(0, 0), new(10, 0), new(5, 8)]);

IEnumerable<string> GetLayers() => ["ground", "objects", "ui"];
```

Immutable collections via collection builders:

```csharp
ImmutableArray<int> primes = [2, 3, 5, 7, 11];
```

Allocation-free lookup tables:

```csharp
static ReadOnlySpan<int> XpPerLevel => [0, 100, 250, 500, 1000, 2000];   // static data, no allocation
int needed = XpPerLevel[level];
```

Stack buffers:

```csharp
Span<Vector2I> neighbours = [new(1, 0), new(-1, 0), new(0, 1), new(0, -1)];   // stack, no heap
```

Your own type with a collection builder:

```csharp
[CollectionBuilder(typeof(TagSet), nameof(TagSet.Create))]
public sealed class TagSet : IEnumerable<string>
{
    private readonly HashSet<string> _tags;
    private TagSet(HashSet<string> tags) => _tags = tags;
    public static TagSet Create(ReadOnlySpan<string> items) => new(new HashSet<string>(items.ToArray(), StringComparer.OrdinalIgnoreCase));
    public IEnumerator<string> GetEnumerator() => _tags.GetEnumerator();
    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}

TagSet tags = ["Fire", "fire", "Ice"];   // 2 tags
```

## 4. Before this feature

```csharp
int[] a = new[] { 1, 2, 3 };
var list = new List<string> { "one", "two" };
int[] none = Array.Empty<int>();
Unit[] everyone = allies.Append(player).Concat(mercenaries).ToArray();
ImmutableArray<int> primes = ImmutableArray.Create(2, 3, 5, 7, 11);
```

## 5. Under the hood

The lowering depends on the target type (simplified):

```csharp
// int[] a = [1, 2, 3];
int[] a = new int[] { 1, 2, 3 };              // RuntimeHelpers.InitializeArray from static data

// List<string> b = ["one", "two", "three"];   (.NET 8+)
var b = new List<string>(3);                   // exact capacity
CollectionsMarshal.SetCount(b, 3);
Span<string> s = CollectionsMarshal.AsSpan(b);
s[0] = "one"; s[1] = "two"; s[2] = "three";

// ReadOnlySpan<int> r = [1, 2, 3];   (constants)
ReadOnlySpan<int> r = <static data blob>;      // no allocation

// int[] single = [.. row0, .. row1];
int[] single = new int[row0.Length + row1.Length];
row0.AsSpan().CopyTo(single);
row1.AsSpan().CopyTo(single.AsSpan(row0.Length));

// IEnumerable<int> e = [1, 2, 3];
IEnumerable<int> e = new <>z__ReadOnlyArray<int>(new[] { 1, 2, 3 });   // compiler-synthesized read-only wrapper

// int[] none = [];
int[] none = Array.Empty<int>();
```

If a spread source's count is unknown (`IEnumerable<T>`), the compiler falls back to a growing list and copies.

## 6. Performance and allocation

- Never worse than the hand-written equivalents; often better (exact capacity, `CopyTo` for spreads, static data for constant spans, cached empties).
- `[]` into a new array or `List<T>` still allocates when non-empty. In hot paths prefer `ReadOnlySpan<T>` targets for constant data.
- Spreading an `IEnumerable<T>` of unknown length allocates intermediate buffers.

## 7. Pitfalls

- `IEnumerable<T> x = [...]` lets the compiler choose the concrete type; do not cast it to `List<T>` or mutate it.
- `var x = [1, 2];` does not compile; give a target type.
- Changing `new List<T>()` to `[]` for a field typed as an interface (`IList<T>`) keeps a `List<T>`, but for `IReadOnlyList<T>` the result is read-only. Check consumers that cast.
- Dictionaries: C# 12 supports `Dictionary<K,V> d = [];` (empty) but not key/value elements; use initializers for content.

## 8. Concerns and limitations

- Constructor arguments (capacity, comparer) need C# 15 `with(...)` or the old syntax.
- Godot: `Godot.Collections.Array` and `Dictionary` support collection initializers, so `[]` works, but every element crosses into `Variant`. Prefer C# collections in logic code.

## 9. When to use: clean-code guidance

### 9.1 Use when
- All new code that creates arrays, lists, spans or immutable collections.
- Concatenating known collections (`[.. a, .. b]` instead of `Concat().ToArray()`).

### 9.2 Avoid when
- You need a specific concrete type behind an interface variable.
- You need capacity or a comparer and are below C# 15.

### 9.3 Good versus bad

```csharp
// Good
private readonly List<Enemy> _enemies = [];
static ReadOnlySpan<byte> Magic => [0x47, 0x44];

// Bad: relies on the hidden concrete type
IEnumerable<int> ids = [1, 2, 3];
((List<int>)ids).Add(4);   // InvalidCastException
```

## 10. Related features

- [../15/collection-expression-arguments.md](../15/collection-expression-arguments.md): `with(capacity: n)`.
- [../11/list-patterns.md](../11/list-patterns.md): matching by shape.
- [inline-arrays.md](inline-arrays.md): stack storage behind `Span<T>` targets.
- [../13/params-collections.md](../13/params-collections.md)

## 11. References

- [Collection expressions](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/collection-expressions)
- [Proposal: Collection expressions](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-12.0/collection-expressions)
- [IDE0300–IDE0305](https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/style-rules/ide0300)
