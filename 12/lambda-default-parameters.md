---
id: cs12-lambda-default-parameters
csharp: 12
tfm: net8.0
status: stable
tags: [lambda, default-parameters, params]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/lambda-expressions#input-parameters-of-a-lambda-expression
---
# Default parameter values (and `params`) in lambdas (C# 12)

> **Abstract.** Lambda parameters may declare **default values** (`(int count = 1) => ...`) and a `params` array,
> with the same rules as methods and local functions. Because `Func<>` / `Action<>` cannot carry defaults, such a
> lambda's natural type is a compiler-synthesized delegate type. Minimal APIs read the defaults as optional parameters.

| | |
|---|---|
| Introduced | C# 12 · .NET 8 (`net8.0`) |
| Status | Stable |
| Kind | Syntax |
| Runtime support needed | No |
| Audience | Everyday code |
| Modernize category | PREFER (instead of overloads or wrapper lambdas) |

## 1. Motivation

Local functions and methods could have optional parameters, lambdas could not. Code that stored callbacks with sensible
defaults had to wrap them or turn the lambda into a local function. Frameworks that inspect lambdas
(Minimal APIs) could not express optional route or query parameters inline.

## 2. The feature explained

### 2.1 Syntax

```csharp
var f = (string name, int count = 1) => $"{count}x {name}";
var g = (params int[] values) => values.Sum();
```

### 2.2 Semantics and rules

1. Default values must be compile-time constants (or `default`), and appear after required parameters.
2. A `params` parameter must be last and be an array type in C# 12 (`params` collections come in C# 13, but lambdas require explicit types).
3. The natural type of such a lambda is an internal synthesized delegate type with the defaults, not `Func<...>`.
4. When converting to an existing delegate type, the **delegate's** defaults apply at call sites; the lambda's defaults are then ignored (warning CS9099 if they differ).
5. `Delegate.Method.GetParameters()[i].DefaultValue` exposes the value via reflection.

## 3. Code examples

### 3.1 Minimal example

```csharp
var spawn = (string enemy, int count = 1) => $"{count}x {enemy}";
spawn("Goblin");      // "1x Goblin"
spawn("Orc", 3);      // "3x Orc"
```

### 3.2 Extended examples

Minimal API with an optional query parameter:

```csharp
app.MapGet("/units", (int page = 1, int size = 20) => store.Page(page, size));
// GET /units            -> page 1, size 20
// GET /units?page=3     -> page 3, size 20
```

`params` in a lambda:

```csharp
var sum = (params int[] xs) => xs.Sum();
int total = sum(1, 2, 3);   // 6
```

Debug console commands with optional arguments:

```csharp
var commands = new Dictionary<string, Delegate>
{
    ["heal"]  = (int amount = 100) => player.Heal(amount),
    ["spawn"] = (string id, int count = 1) => Spawn(id, count),
};
```

## 4. Before this feature

```csharp
Func<string, int, string> spawnCore = (enemy, count) => $"{count}x {enemy}";
string Spawn(string enemy, int count = 1) => spawnCore(enemy, count);   // wrapper for the default

// or a local function
string SpawnLocal(string enemy, int count = 1) => $"{count}x {enemy}";
```

## 5. Under the hood

```csharp
// var spawn = (string enemy, int count = 1) => ...;
internal delegate TResult <>f__AnonymousDelegate0<T1, T2, TResult>(T1 arg1, T2 arg2 = 1);   // synthesized

<>f__AnonymousDelegate0<string, int, string> spawn = <>c.<>9__0_0 ??= new(<>c.<>9.<Main>b__0_0);
```

The lambda method `<Main>b__0_0` gets `[Optional, DefaultParameterValue(1)]` on `count`. At the call site
`spawn("Goblin")` the compiler inserts the constant: `spawn.Invoke("Goblin", 1)`.

## 6. Performance and allocation

Same as other lambdas. The synthesized delegate type adds no runtime cost.

## 7. Pitfalls

- Assigning to `Func<string, int, string>` drops the default: `Func<...> f = (string s, int c = 1) => ...; f("x");` does not compile.
- `var` is required to keep the synthesized type; this type cannot be named in signatures.

## 8. Concerns and limitations

- Synthesized delegate types are not interchangeable with `Func<>`; passing them to APIs that want `Func<>` requires a conversion of the lambda, not of the variable.

## 9. When to use: clean-code guidance

### 9.1 Use when
- Minimal API handlers with optional parameters.
- Local callback tables where defaults simplify calls.

### 9.2 Avoid when
- The delegate is part of a public API: declare a named delegate type with defaults instead.

### 9.3 Good versus bad

```csharp
// Good
app.MapGet("/search", (string q, int limit = 10) => Search(q, limit));

// Bad: default silently ignored because the target type has none
Func<int, int> twice = (int x = 5) => x * 2;   // warning CS9099
```

## 10. Related features

- [../10/lambda-improvements.md](../10/lambda-improvements.md)
- [../14/lambda-parameter-modifiers.md](../14/lambda-parameter-modifiers.md)
- [../13/params-collections.md](../13/params-collections.md)

## 11. References

- [Lambda expressions: input parameters](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/lambda-expressions#input-parameters-of-a-lambda-expression)
- [Proposal: Lambda optional parameters](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-12.0/lambda-method-group-defaults)
