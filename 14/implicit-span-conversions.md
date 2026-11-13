---
id: cs14-implicit-span-conversions
csharp: 14
tfm: net10.0
status: stable
tags: [span, readonlyspan, array, conversion, performance]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-14
---
# First-class implicit span conversions (C# 14)

The language adds implicit conversions between `T[]`, `Span<T>` and `ReadOnlySpan<T>`.

## Rules
- Span types can be **receivers of extension methods**: an extension on `ReadOnlySpan<T>` is callable on an array.
- The conversions compose with other conversions and help generic type inference.
- Full list: "built-in types" article in the language reference.

## Use when
- Writing one API on `ReadOnlySpan<T>` that should accept arrays, spans and collection expressions
  without overloads.

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

## Pitfalls
- Overload resolution can now pick span-based overloads where it picked others before
  (e.g. `MemoryExtensions` instead of LINQ). In code that must stay an expression tree
  (EF Core queries), check that calls still translate after upgrading.
