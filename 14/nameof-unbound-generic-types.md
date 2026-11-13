---
id: cs14-nameof-unbound-generic-types
csharp: 14
tfm: net10.0
status: stable
tags: [nameof, generics]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-14
---
# `nameof` with unbound generic types (C# 14)

```csharp
nameof(List<>)          // "List"
nameof(Dictionary<,>)   // "Dictionary"
```

Before C# 14 a closed type was required (`nameof(List<int>)`).
Useful in logging, diagnostics, registries and source generators.
