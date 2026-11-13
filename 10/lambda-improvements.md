---
id: cs10-lambda-improvements
csharp: 10
tfm: net6.0
status: stable
tags: [lambda, var, natural-type, attributes, return-type]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/lambda-expressions
---
# Lambda improvements (C# 10)

Three related changes:

1. **Natural type**: the compiler infers a delegate type, so `var` works.
```csharp
var heal = (int amount) => amount * 2;      // Func<int, int>
var parse = int.Parse;                      // method group with a single overload: Func<string, int>
```
2. **Explicit return type** when inference fails:
```csharp
var choose = object (bool b) => b ? 1 : "one";
```
3. **Attributes on lambdas** and their parameters:
```csharp
var handler = [Obsolete] (int x) => x;
```

## Notes
- Minimal APIs (`app.MapGet("/", () => ...)`) rely on these.
- Lambda default parameter values come in C# 12; parameter modifiers without types in C# 14.
