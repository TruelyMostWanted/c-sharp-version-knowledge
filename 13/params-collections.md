---
id: cs13-params-collections
csharp: 13
tfm: net9.0
status: stable
tags: [params, span, allocation, performance]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-13
---
# `params` collections (C# 13)

`params` works with any recognized collection type, not only arrays: `Span<T>`, `ReadOnlySpan<T>`,
types implementing `IEnumerable<T>` with an `Add` method, and the interfaces `IEnumerable<T>`,
`IReadOnlyCollection<T>`, `IReadOnlyList<T>`, `ICollection<T>`, `IList<T>`.

## Use when
- Variadic helpers on hot paths: `params ReadOnlySpan<T>` lets the compiler avoid a heap array.

```csharp
public void Concat<T>(params ReadOnlySpan<T> items)
{
    for (int i = 0; i < items.Length; i++)
        Console.Write($"{items[i]} ");
}

Concat(1, 2, 3);
```

## Rules
- With an interface type, the compiler synthesizes the storage.

## Pitfalls
- Adding a `params ReadOnlySpan<T>` overload next to `params T[]` changes which overload existing calls bind to (usually the span one). Intended, but check API behaviour.
