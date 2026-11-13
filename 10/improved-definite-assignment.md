---
id: cs10-improved-definite-assignment
csharp: 10
tfm: net6.0
status: stable
tags: [definite-assignment, nullable, warnings]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-10
---
# Improved definite assignment and null-state analysis (C# 10)

Definite assignment and null-state warnings are more accurate, e.g. with `?.` combined with `==` or `is`,
and with constant `true`/`false` comparisons. No new syntax.

```csharp
if (dict?.TryGetValue(key, out var value) == true)
{
    Use(value);   // C# 10: value is definitely assigned here
}
```
Consequence: workarounds (dummy initializations, `!`) added for older compilers can often be removed.
