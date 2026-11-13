---
id: cs12-inline-arrays
csharp: 12
tfm: net8.0
status: stable
needs-runtime: true (InlineArrayAttribute, .NET 8)
tags: [inline-array, fixed-buffer, struct, performance]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-12
---
# Inline arrays (C# 12)

A struct marked `[InlineArray(N)]` with exactly one field holds N elements inline (no heap array).
Performance is similar to an unsafe fixed-size buffer, but safe and usable as a span.

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

## Use when
- Small fixed-size data inside structs without allocation (e.g. 8 grid neighbours, 4 wheel contacts,
  a fixed slot array in a struct-of-data entity).
- Microsoft notes it is mainly for runtime and library authors; you often meet it as `Span<T>` from APIs.
