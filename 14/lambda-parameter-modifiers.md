---
id: cs14-lambda-parameter-modifiers
csharp: 14
tfm: net10.0
status: stable
tags: [lambda, ref, out, in, scoped]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-14
---
# Simple lambda parameters with modifiers (C# 14)

`scoped`, `ref`, `in`, `out` and `ref readonly` can be used on lambda parameters **without** writing their types.

```csharp
delegate bool TryParse<T>(string text, out T result);

TryParse<int> parse1 = (text, out result) => int.TryParse(text, out result);          // C# 14
TryParse<int> parse2 = (string text, out int result) => int.TryParse(text, out result); // before
```

## Rules
- `params` still needs an explicitly typed parameter list.
