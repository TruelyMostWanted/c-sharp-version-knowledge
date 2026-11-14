---
id: cs11-list-patterns
csharp: 11
tfm: net7.0
status: stable
tags: [pattern-matching, list-pattern, slice, arrays, spans]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/patterns#list-patterns
modernizes: manual Length checks plus index access
---
# List patterns (C# 11)

> **Abstract.** A list pattern `[p1, p2, ..]` matches a sequence by **shape**: element count and per-element
> patterns. The slice pattern `..` matches zero or more elements and can capture them (`.. var rest`).
> It replaces chains of `Length` checks and indexing with one declarative expression, and works on arrays, lists,
> spans and any countable, indexable type.

| | |
|---|---|
| Introduced | C# 11 · .NET 7 (`net7.0`) |
| Status | Stable |
| Kind | Syntax (pattern matching) |
| Runtime support needed | No (uses `Index` / `Range`, available since .NET Core 3.0) |
| Audience | Everyday code |
| Modernize category | CASE |

## 1. Motivation

Parsing command lines, tokens or small fixed-format records needed imperative code:

```csharp
if (args.Length == 3 && args[0] == "spawn" && int.TryParse(args[2], out var n)) { ... }
```

C# already had property, type and relational patterns, but nothing for sequences.

## 2. The feature explained

### 2.1 Syntax

```csharp
[]                        // empty
[var x]                   // exactly one element
[1, _, 3]                 // three elements, middle any
[> 0, ..]                 // at least one element, first positive
[.., var last]            // at least one, capture last
[var first, .. var mid, var last]  // at least two, capture slice
```

### 2.2 Semantics and rules

1. **Countable**: the type has an accessible `Length` or `Count` property of type `int`.
2. **Indexable**: an indexer taking `Index` or `int` (the compiler can use `^n` via `Count - n`).
3. **Slice capture** (`.. var rest`) also needs a `Range` indexer or a `Slice(int, int)` method. A bare `..` without capture needs nothing extra.
4. At most **one** slice pattern per list pattern, at any position.
5. Element patterns can be any pattern, including nested list patterns and property patterns.
6. Works on `string` (it is countable and indexable): `s is ['#', ..]`.
7. Exhaustiveness analysis understands lengths (`[]`, `[_]`, `[_, _, ..]` covers all lengths).
8. `IEnumerable<T>` alone is **not** supported (not indexable).

## 3. Code examples

### 3.1 Minimal example

```csharp
int[] numbers = { 1, 2, 3 };
bool a = numbers is [1, 2, 3];          // true
bool b = numbers is [_, > 1, ..];       // true
if (numbers is [var first, .. var rest]) { /* first = 1, rest = {2, 3} */ }
```

### 3.2 Extended examples

Command parsing (debug console):

```csharp
string Execute(string[] args) => args switch
{
    []                                              => "no command",
    ["help"]                                        => Help(),
    ["spawn", var name]                             => Spawn(name, 1),
    ["spawn", var name, var n] when int.TryParse(n, out var count) => Spawn(name, count),
    ["tp", var x, var y]                            => Teleport(x, y),
    [var cmd, ..]                                   => $"unknown: {cmd}",
};
```

CSV-like records:

```csharp
Unit? ParseUnit(string line) => line.Split(',') switch
{
    [var id, var hp, var speed]          => new Unit(id, int.Parse(hp), float.Parse(speed)),
    [var id, var hp]                     => new Unit(id, int.Parse(hp), 1f),
    _                                    => null,
};
```

Binary headers with spans:

```csharp
ReadOnlySpan<byte> data = File.ReadAllBytes(path);
string kind = data switch
{
    [0x89, (byte)'P', (byte)'N', (byte)'G', ..] => "png",
    [0xFF, 0xD8, 0xFF, ..]                      => "jpeg",
    [(byte)'R', (byte)'I', (byte)'F', (byte)'F', ..] => "riff",
    _                                           => "unknown",
};
```

Recursion over a span (functional style):

```csharp
static int Sum(ReadOnlySpan<int> xs) => xs switch
{
    []                => 0,
    [var head, .. var tail] => head + Sum(tail),   // span slice: no allocation
};
```

Nested and property patterns inside elements:

```csharp
if (path is [{ IsWalkable: true }, .., { IsGoal: true }]) Follow(path);
```

## 4. Before this feature

```csharp
if (args.Length == 2 && args[0] == "spawn")
    return Spawn(args[1], 1);
if (args.Length == 3 && args[0] == "spawn" && int.TryParse(args[2], out var n))
    return Spawn(args[1], n);
if (args.Length > 0)
    return "unknown: " + args[0];
return "no command";
```

## 5. Under the hood

`args is ["spawn", var name]` is lowered to length checks and indexing:

```csharp
string name;
bool result = false;
if (args != null && args.Length == 2 && args[0] == "spawn")
{
    name = args[1];
    result = true;
}
```

`[.., var last]` reads `args[args.Length - 1]` (or the `Index` indexer with `^1`).
`.. var rest` calls the range indexer (`args[1..]`) or `Slice(1, length - 1)`.
In `switch` expressions the length test is shared across arms via the decision DAG.

## 6. Performance and allocation

- Matching itself does not allocate.
- **Slice captures allocate for arrays and strings**: `arr[1..]` on an array creates a new array (`RuntimeHelpers.GetSubArray`); on `string` a new string.
  On `Span<T>` / `ReadOnlySpan<T>` slicing is free. In hot code, match on spans (`arr.AsSpan()`).
- `List<T>` has no range indexer; slice capture on `List<T>` is not available (bare `..` works).

## 7. Pitfalls

- Capturing a slice of an array inside a loop silently allocates on every iteration.
- `IEnumerable<T>` / LINQ results do not support list patterns; materialize or use spans.
- `when` clauses with `TryParse` produce out variables only in that arm; keep arms readable.

## 8. Concerns and limitations

- Readability is excellent for small, fixed shapes; long patterns with many nested elements become cryptic.
- Godot: Godot collections (`Godot.Collections.Array`) are countable and indexable, but each element access crosses into the engine and returns `Variant`; convert to typed C# arrays or spans first.

## 9. When to use: clean-code guidance

### 9.1 Use when
- Matching small sequences by shape: commands, tokens, headers, tuples encoded as arrays.
- Recursive algorithms over spans.

### 9.2 Avoid when
- Large inputs that need loops anyway.
- Hot paths with slice captures on arrays or strings.

### 9.3 Good versus bad

```csharp
// Good
if (tokens is ["move", var dir]) Move(dir);

// Bad: slice capture of an array in a per-frame loop allocates every frame
while (queue is [var next, .. var remaining]) { Process(next); queue = remaining; }
```

## 10. Related features

- [span-pattern-constant-string.md](span-pattern-constant-string.md): `"text"` patterns on spans.
- [../10/extended-property-patterns.md](../10/extended-property-patterns.md)
- [../12/collection-expressions.md](../12/collection-expressions.md): the construction counterpart of list patterns.

## 11. References

- [List patterns](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/patterns#list-patterns)
- [Proposal: List patterns](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-11.0/list-patterns)
