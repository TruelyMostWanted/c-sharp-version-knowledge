---
id: cs14-implicit-span-conversions
csharp: 14
tfm: net10.0
status: stable
tags: [span, readonlyspan, array, conversion, performance]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-14.0/first-class-span-types
---
# First-class span types and implicit span conversions (C# 14)

> **Abstract.** C# 14 makes `Span<T>` and `ReadOnlySpan<T>` first-class in the language: there are built-in implicit
> conversions `T[]` → `Span<T>` → `ReadOnlySpan<T>` (and `T[]` → `ReadOnlySpan<T>`, `string` → `ReadOnlySpan<char>`),
> they participate in **extension method receivers** and in **generic type inference**. One API on
> `ReadOnlySpan<T>` now serves arrays, spans and collection expressions without overloads.

| | |
|---|---|
| Introduced | C# 14 · .NET 10 (`net10.0`) |
| Status | Stable |
| Kind | Semantics (conversions, overload resolution) |
| Runtime support needed | No |
| Audience | Everyday code, library authors |
| Modernize category | n/a (automatic; check behaviour changes) |

## 1. Motivation

Conversions from arrays to spans were **user-defined** operators on `Span<T>`. User-defined conversions are not
considered for extension method receivers or type inference, so:

```csharp
int[] arr = ...;
arr.Sum();                     // could not bind to an extension on ReadOnlySpan<int>
M(arr);                        // void M<T>(ReadOnlySpan<T> s): T could not be inferred
```

Libraries had to duplicate APIs for `T[]`, `Span<T>` and `ReadOnlySpan<T>`.

## 2. The feature explained

### 2.1 Syntax

No new syntax.

### 2.2 Semantics and rules

1. New implicit span conversions (language-defined): `T[]` → `Span<T>`, `T[]` → `ReadOnlySpan<T>`, `Span<T>` → `ReadOnlySpan<T>`,
   `ReadOnlySpan<U>` → `ReadOnlySpan<T>` when `U` is a reference type convertible to `T` (covariance), `string` → `ReadOnlySpan<char>`.
2. Extension methods whose receiver is `Span<T>` / `ReadOnlySpan<T>` apply to arrays and spans directly.
3. Generic type inference sees through these conversions.
4. Overload resolution: span conversions are "better" in more situations; `ReadOnlySpan<T>` overloads may be chosen over `IEnumerable<T>` overloads.

## 3. Code examples

### 3.1 Minimal example

```csharp
public static class SpanMath
{
    public static float Sum(this ReadOnlySpan<float> values)
    {
        float s = 0;
        foreach (var v in values) s += v;
        return s;
    }
}

float[] speeds = GetSpeeds();
float total = speeds.Sum();   // array -> ReadOnlySpan<float> as extension receiver
```

### 3.2 Extended examples

Generic inference through arrays:

```csharp
static T First<T>(ReadOnlySpan<T> items) => items[0];

int[] ids = [4, 8, 15];
int first = First(ids);        // C# 14: T = int inferred
```

One API for many inputs:

```csharp
public static int CountAbove(this ReadOnlySpan<int> values, int threshold)
{
    int n = 0;
    foreach (var v in values) if (v > threshold) n++;
    return n;
}

int[] arr = [1, 5, 9];
Span<int> span = arr;
List<int> list = [3, 7];

arr.CountAbove(4);                                  // array
span.CountAbove(4);                                 // span
CollectionsMarshal.AsSpan(list).CountAbove(4);      // list via span
```

Covariance of read-only spans:

```csharp
ReadOnlySpan<string> names = ["a", "b"];
ReadOnlySpan<object> objs = names;   // reference-type covariance
```

## 4. Before this feature

```csharp
float total = speeds.AsSpan().Sum();            // explicit AsSpan
int first = First<int>(ids);                    // explicit type argument
int first2 = First((ReadOnlySpan<int>)ids);     // or cast
```

## 5. Under the hood

The conversions compile to the same calls as before (`new ReadOnlySpan<T>(array)`, `MemoryExtensions.AsSpan`,
`Span<T>.op_Implicit`); only the language rules for **when** they apply changed. No new IL patterns.

## 6. Performance and allocation

No allocation; span-based overloads chosen by the new rules are often faster than `IEnumerable<T>` LINQ equivalents.

## 7. Pitfalls

- **Behaviour change on upgrade:** calls like `array.Contains(x)`, `array.Reverse()` or `array.SequenceEqual(...)` may bind to
  `MemoryExtensions` (span) instead of `System.Linq.Enumerable`. Usually faster and equivalent, but:
  - `array.Reverse()` now binds to `MemoryExtensions.Reverse(Span<T>)`, which reverses **in place** and returns `void`, instead of LINQ's lazy copy. Code like `foreach (var x in array.Reverse())` changes meaning or fails to compile.
  - Expression trees (EF Core queries, other LINQ providers) cannot contain span calls; queries that captured arrays may stop translating.
- Covariant `ReadOnlySpan<object>` from `string[]` is fine, but `Span<T>` covariance is not allowed (would be unsafe).

## 8. Concerns and limitations

- Review LINQ calls on arrays after upgrading to C# 14; prefer `Enumerable.Reverse(array)` explicitly where a copy is intended.
- Godot: Godot `Array`/`PackedXArray` types are not .NET arrays; these conversions apply to C# arrays (`float[]`, `Vector2[]`).

## 9. When to use: clean-code guidance

### 9.1 Use when
- Writing helpers: target `ReadOnlySpan<T>` (or `Span<T>` for writers) once and rely on the conversions.

### 9.2 Avoid when
- APIs that must accept lazy sequences: keep `IEnumerable<T>`.

### 9.3 Good versus bad

```csharp
// Good: one overload for arrays, spans, collection expressions
public static float Average(this ReadOnlySpan<float> xs) { ... }

// Bad (after C# 14): relying on LINQ Reverse on an array
foreach (var e in enemies.Reverse()) { }        // now in-place span Reverse; use Enumerable.Reverse(enemies)
```

## 10. Related features

- [extension-members.md](extension-members.md)
- [../12/collection-expressions.md](../12/collection-expressions.md)
- [../13/params-collections.md](../13/params-collections.md)
- [../13/overload-resolution-priority.md](../13/overload-resolution-priority.md)

## 11. References

- [What's new in C# 14: implicit span conversions](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-14#implicit-span-conversions)
- [Proposal: First-class span types](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-14.0/first-class-span-types)
- [Breaking changes in the compiler after .NET 9](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/breaking-changes/compiler%20breaking%20changes%20-%20dotnet%2010)
