---
id: cs13-params-collections
csharp: 13
tfm: net9.0
status: stable
tags: [params, span, allocation, performance]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/method-parameters#params-modifier
modernizes: params T[] on hot APIs
---
# `params` collections (C# 13)

> **Abstract.** The `params` modifier now works with any type that collection expressions support, not only arrays:
> `Span<T>`, `ReadOnlySpan<T>`, `List<T>`, interfaces like `IEnumerable<T>` / `IReadOnlyList<T>`, and collection-builder types.
> With `params ReadOnlySpan<T>` the compiler passes the arguments in **stack memory**, so variadic calls stop allocating.

| | |
|---|---|
| Introduced | C# 13 · .NET 9 (`net9.0`) |
| Status | Stable |
| Kind | Syntax + semantics |
| Runtime support needed | No (stack buffers use inline arrays, .NET 8+) |
| Audience | Everyday code, library authors |
| Modernize category | PREFER for new APIs, CASE for existing |

## 1. Motivation

`params T[]` allocates a new array on **every** call: `string.Join(", ", a, b, c)`, `Path.Combine(x, y, z)`, logging
helpers. In hot code this garbage adds up. APIs that only read the arguments do not need an array at all.

## 2. The feature explained

### 2.1 Syntax

```csharp
void M(params ReadOnlySpan<int> values);
void N(params IEnumerable<string> names);
void O(params List<Item> items);
```

### 2.2 Semantics and rules

1. Allowed `params` types: arrays, `Span<T>`, `ReadOnlySpan<T>`, types with `[CollectionBuilder]`, types with a collection initializer pattern (`IEnumerable` + `Add`), and the interfaces `IEnumerable<T>`, `IReadOnlyCollection<T>`, `IReadOnlyList<T>`, `ICollection<T>`, `IList<T>`.
2. The call site may pass zero or more arguments (packed into the collection) or one argument of the collection type (passed as is).
3. For span types the compiler may use a stack-allocated inline array; for interfaces it synthesizes storage like a collection expression.
4. Overload resolution prefers `params ReadOnlySpan<T>` over `params T[]` when both apply (better conversion).
5. `params` remains limited to the last parameter, one per method.

## 3. Code examples

### 3.1 Minimal example

```csharp
public static void Print<T>(params ReadOnlySpan<T> items)
{
    for (int i = 0; i < items.Length; i++)
        Console.Write($"{items[i]} ");
}

Print(1, 2, 3);   // no heap allocation for the arguments
```

### 3.2 Extended examples

Hot math helper:

```csharp
public static float Max(params ReadOnlySpan<float> values)
{
    float max = float.NegativeInfinity;
    foreach (float v in values) if (v > max) max = v;
    return max;
}

float threat = Max(distanceScore, healthScore, aggroScore);   // per frame, allocation-free
```

Accepting arrays and lists without copies:

```csharp
public void Enqueue(params IEnumerable<Command> commands)
{
    foreach (var c in commands) _queue.Enqueue(c);
}

Enqueue(cmdA, cmdB);           // packed
Enqueue(existingList);         // passed as is
```

Span overload next to an array overload (library):

```csharp
public static string JoinTags(params string[] tags) => string.Join(",", tags);          // kept for binary compat
public static string JoinTags(params ReadOnlySpan<string> tags) => string.Join(",", tags);  // .NET 9 string.Join has span overloads
```

## 4. Before this feature

```csharp
public static float Max(params float[] values) { ... }    // allocates per call

// workaround: fixed-arity overloads
public static float Max(float a, float b) => ...;
public static float Max(float a, float b, float c) => ...;
```

## 5. Under the hood

```csharp
// Print(1, 2, 3);  with params ReadOnlySpan<int>
<>y__InlineArray3<int> buffer = default;          // compiler-synthesized [InlineArray(3)] struct, on the stack
<PrivateImplementationDetails>.InlineArrayElementRef<<>y__InlineArray3<int>, int>(ref buffer, 0) = 1;
... = 2;
... = 3;
Print(<PrivateImplementationDetails>.InlineArrayAsReadOnlySpan<<>y__InlineArray3<int>, int>(in buffer, 3));
```

For constant arguments the compiler may instead point the span at static data. The parameter is marked with
`[ParamCollection]` in metadata (arrays keep `[ParamArray]`).

## 6. Performance and allocation

- `params ReadOnlySpan<T>` / `Span<T>`: no heap allocation for arguments.
- Interface `params` types allocate like collection expressions targeting those interfaces.
- `List<T>` `params` allocates the list.

## 7. Pitfalls

- Adding a `params ReadOnlySpan<T>` overload next to `params T[]` changes which overload **existing** calls bind to (usually the span one) when recompiled. Make both behave identically.
- Span parameters cannot be stored or used after `await`; copy them (`ToArray()`) if needed.
- Ambiguities can arise with other overloads taking a single collection; test call sites.

## 8. Concerns and limitations

- Binary compatibility: removing `params T[]` breaks compiled callers; keep it.
- Godot: `GD.Print(params object[])` remains array-based; wrap with your own `params ReadOnlySpan<object>` helper only if profiling shows a problem (boxing of value types still happens).

## 9. When to use: clean-code guidance

### 9.1 Use when
- New variadic APIs that only read their arguments: `params ReadOnlySpan<T>`.
- APIs that accept "any collection or a few items": `params IEnumerable<T>`.

### 9.2 Avoid when
- The method stores the arguments (you need a copy anyway) or is `async`.

### 9.3 Good versus bad

```csharp
// Good
public static int Sum(params ReadOnlySpan<int> xs) { int s = 0; foreach (var x in xs) s += x; return s; }

// Bad: span params stored beyond the call (does not compile) or copied anyway
public void Remember(params ReadOnlySpan<int> xs) => _saved = xs.ToArray();   // just take int[]
```

## 10. Related features

- [../12/collection-expressions.md](../12/collection-expressions.md)
- [../12/inline-arrays.md](../12/inline-arrays.md)
- [overload-resolution-priority.md](overload-resolution-priority.md)

## 11. References

- [params modifier](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/method-parameters#params-modifier)
- [Proposal: params collections](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-13.0/params-collections)
