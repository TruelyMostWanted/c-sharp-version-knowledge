---
id: cs14-lambda-parameter-modifiers
csharp: 14
tfm: net10.0
status: stable
tags: [lambda, ref, out, in, scoped]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/lambda-expressions#input-parameters-of-a-lambda-expression
modernizes: explicitly typed lambda parameter lists written only to add out/ref/in
---
# Simple lambda parameters with modifiers (C# 14)

> **Abstract.** Lambda parameters can carry the modifiers `scoped`, `ref`, `in`, `out` and `ref readonly` **without**
> stating their types: `(text, out result) => int.TryParse(text, out result)`. The types still come from the target
> delegate. Before C# 14 a single modifier forced you to write every parameter's type.

| | |
|---|---|
| Introduced | C# 14 · .NET 10 (`net10.0`) |
| Status | Stable |
| Kind | Syntax |
| Runtime support needed | No |
| Audience | Everyday code |
| Modernize category | AUTO |

## 1. Motivation

Delegates with `out` or `ref` parameters (`TryParse`-style callbacks, span visitors) required fully typed lambdas:

```csharp
TryParse<Dictionary<string, List<int>>> p =
    (string text, out Dictionary<string, List<int>> result) => ...;   // long, redundant type
```

## 2. The feature explained

### 2.1 Syntax

```csharp
(text, out result) => ...
(ref acc, value) => ...
(scoped span) => ...
(in matrix) => ...
```

### 2.2 Semantics and rules

1. Modifiers allowed on implicitly typed parameters: `ref`, `out`, `in`, `ref readonly`, `scoped`, `scoped ref`, ...
2. The lambda must be converted to a delegate type that determines the types; there is no natural type for such lambdas.
3. `params` still requires an explicitly typed parameter list.
4. Mixing explicitly and implicitly typed parameters in one list is still not allowed.
5. Attributes on parameters still require explicit types.

## 3. Code examples

### 3.1 Minimal example

```csharp
delegate bool TryParse<T>(string text, out T result);

TryParse<int> parse1 = (text, out result) => int.TryParse(text, out result);           // C# 14
TryParse<int> parse2 = (string text, out int result) => int.TryParse(text, out result); // before
```

### 3.2 Extended examples

Parser registry:

```csharp
var parsers = new Dictionary<Type, TryParse<object>>
{
    [typeof(int)]   = (s, out r) => { bool ok = int.TryParse(s, out var v); r = v; return ok; },
    [typeof(float)] = (s, out r) => { bool ok = float.TryParse(s, CultureInfo.InvariantCulture, out var v); r = v; return ok; },
};
```

Accumulating by reference:

```csharp
delegate void Accumulate<T>(ref T acc, T value);

Accumulate<Vector2> add = (ref acc, value) => acc += value;
var sum = Vector2.Zero;
foreach (var v in velocities) add(ref sum, v);
```

`scoped` span callbacks:

```csharp
delegate int SpanCounter(scoped ReadOnlySpan<char> text);

SpanCounter vowels = (scoped text) =>
{
    int n = 0;
    foreach (char c in text) if ("aeiou".Contains(c)) n++;
    return n;
};
```

## 4. Before this feature

```csharp
TryParse<int> parse = (string text, out int result) => int.TryParse(text, out result);
Accumulate<Vector2> add = (ref Vector2 acc, Vector2 value) => acc += value;
```

## 5. Under the hood

Purely syntactic: the generated lambda method has the same signature as the explicitly typed version.

## 6. Performance and allocation

None.

## 7. Pitfalls

- Readability: `(a, out b)` hides types; fine when the delegate type is right next to it, less so in long call chains.

## 8. Concerns and limitations

- No natural type: `var f = (s, out r) => ...;` does not compile.

## 9. When to use: clean-code guidance

### 9.1 Use when
- Lambdas for delegates with `out`/`ref`/`in`/`scoped` parameters where types are obvious from the target.

### 9.2 Avoid when
- The delegate type is far away and types aid understanding.

### 9.3 Good versus bad

```csharp
// Good
TryParse<Version> parseVersion = (s, out v) => Version.TryParse(s, out v);

// Unnecessary on C# 14
TryParse<Version> parseVersion2 = (string s, out Version v) => Version.TryParse(s, out v);
```

## 10. Related features

- [../10/lambda-improvements.md](../10/lambda-improvements.md)
- [../12/lambda-default-parameters.md](../12/lambda-default-parameters.md)
- [../12/ref-readonly-parameters.md](../12/ref-readonly-parameters.md)

## 11. References

- [Lambda expressions: input parameters](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/lambda-expressions#input-parameters-of-a-lambda-expression)
- [Proposal: Simple lambda parameters with modifiers](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-14.0/simple-lambda-parameters-with-modifiers)
