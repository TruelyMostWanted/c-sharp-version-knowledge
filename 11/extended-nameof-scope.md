---
id: cs11-extended-nameof-scope
csharp: 11
tfm: net7.0
status: stable
tags: [nameof, attributes, nullable]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-11
modernizes: string literals with parameter names inside attributes
---
# Extended `nameof` scope (C# 11)

Inside attributes on a method or its parameters, `nameof` can refer to that method's parameters and type parameters.

```csharp
[return: NotNullIfNotNull(nameof(path))]
public static string? Normalize(string? path) => path?.Replace('\\', '/');
```
Before: `[return: NotNullIfNotNull("path")]`, which breaks silently on rename.
