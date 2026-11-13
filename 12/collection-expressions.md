---
id: cs12-collection-expressions
csharp: 12
tfm: net8.0
status: stable
tags: [collection-expressions, spread, array, list, span]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-12
related: ../15/collection-expression-arguments.md
---
# Collection expressions (C# 12)

```csharp
int[] a = [1, 2, 3];
List<string> b = ["one", "two", "three"];
Span<char> c = ['a', 'b', 'c'];
int[][] twoD = [[1, 2, 3], [4, 5, 6]];

int[] row0 = [1, 2, 3];
int[] row1 = [4, 5, 6];
int[] single = [.. row0, .. row1];      // spread
List<int> empty = [];
```

## Rules
- `..e` spreads all elements of `e` (must be a collection).
- Targets: arrays, `Span<T>`, `ReadOnlySpan<T>`, and types supporting collection initializers (`List<T>`, ...).
  More types via `[CollectionBuilder]`.
- For constructor arguments (capacity, comparer) see C# 15 `with(...)`.

## Pitfalls
- Assigning to an interface type (`IEnumerable<T> x = [...]`) lets the compiler choose the concrete type; do not rely on it being a `List<T>`.
- In hot paths, `[]` to a new array or list still allocates. `ReadOnlySpan<T>` of constants may avoid it.
