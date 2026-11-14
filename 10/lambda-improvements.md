---
id: cs10-lambda-improvements
csharp: 10
tfm: net6.0
status: stable
tags: [lambda, var, natural-type, attributes, return-type]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/lambda-expressions
modernizes: Func/Action declarations written only to give a lambda a type
---
# Lambda improvements (C# 10)

> **Abstract.** Lambdas and method groups gained a **natural type**: `var f = (int x) => x * 2;` now infers
> `Func<int, int>`. Lambdas may declare an **explicit return type** (`object (bool b) => ...`) and carry
> **attributes** on themselves, their parameters and their return value. Together these make lambdas usable as
> first-class values and power ASP.NET Core Minimal APIs.

| | |
|---|---|
| Introduced | C# 10 · .NET 6 (`net6.0`) |
| Status | Stable |
| Kind | Syntax + type inference |
| Runtime support needed | No |
| Audience | Everyday code |
| Modernize category | PREFER |

## 1. Motivation

Before C# 10 a lambda had no type of its own; it needed a target type:

```csharp
var f = x => x * 2;                     // error: cannot infer
Func<int, int> g = x => x * 2;          // OK, but verbose
object o = (Func<int, int>)(x => x * 2);// cast needed for object / Delegate targets
```

Frameworks that accept `Delegate` (Minimal APIs) also had no way to read attributes from a lambda's parameters.

## 2. The feature explained

### 2.1 Syntax

```csharp
var square = (int x) => x * x;                       // natural type
var choose = object (bool b) => b ? 1 : "one";       // explicit return type
var handler = [Obsolete] (int x) => x;               // attribute on the lambda
var parse = ([FromQuery] int page) => page;          // attribute on a parameter
var r = [return: NotNull] (string? s) => s ?? "";    // attribute on the return value
```

### 2.2 Semantics and rules

1. A lambda has a natural type if **all parameter types are explicit** and the return type can be inferred (or is explicit).
2. The natural type is `Func<...>` / `Action<...>` when one fits; otherwise (e.g. `ref` / `out` parameters, more than 16 parameters)
   the compiler synthesizes an internal anonymous delegate type.
3. A **method group** has a natural type if it has exactly one applicable overload (refined in C# 13).
4. Lambdas convert to `Delegate`, `MulticastDelegate`, `object` and `Expression` / `LambdaExpression` via the natural type.
5. With an attribute list, the parameter list must be parenthesized: `[A] (x) => x`, not `[A] x => x`.
6. An explicit return type requires a parenthesized parameter list.

## 3. Code examples

### 3.1 Minimal example

```csharp
var heal = (int amount) => amount * 2;   // Func<int, int>
var parse = int.Parse;                   // error in C# 10: int.Parse has several overloads
var log = Console.WriteLine;             // error as well (many overloads)
var isEven = (int n) => n % 2 == 0;      // Func<int, bool>
```

### 3.2 Extended examples

Minimal APIs (the main consumer of these features):

```csharp
var app = WebApplication.Create(args);

app.MapGet("/units/{id}", ([FromRoute] int id, UnitStore store) => store.Find(id));
app.MapPost("/units", async (Unit unit, UnitStore store) =>
{
    await store.AddAsync(unit);
    return Results.Created($"/units/{unit.Id}", unit);
});
```

Lambdas as values in a table:

```csharp
var rules = new Dictionary<string, Func<int, bool>>
{
    ["positive"] = (int x) => x > 0,
    ["even"]     = (int x) => x % 2 == 0,
};
```

Explicit return type where inference fails:

```csharp
var pick = object (bool useNumber) => useNumber ? 42 : "none";
var maybe = int? (string s) => int.TryParse(s, out var v) ? v : null;
```

Natural type to `Delegate` and to expression trees:

```csharp
Delegate d = (string s) => s.Length;                     // Func<string, int>
Expression e = (int x) => x + 1;                          // Expression<Func<int, int>>
```

## 4. Before this feature

```csharp
Func<int, int> heal = amount => amount * 2;
Func<bool, object> choose = b => b ? (object)1 : "one";
app.MapGet("/units/{id}", (Func<int, UnitStore, Unit?>)((id, store) => store.Find(id)));
```

## 5. Under the hood

The lambda body still compiles to a method (in a closure class if it captures, otherwise in the cached `<>c` class):

```csharp
// var heal = (int amount) => amount * 2;
[CompilerGenerated]
private sealed class <>c
{
    public static readonly <>c <>9 = new();
    public static Func<int, int> <>9__0_0;
    internal int <Main>b__0_0(int amount) => amount * 2;
}

Func<int, int> heal = <>c.<>9__0_0 ??= new Func<int, int>(<>c.<>9.<Main>b__0_0);
```

Attributes written on the lambda are emitted on the generated method `<Main>b__0_0` and its parameters,
which is how frameworks read them through `Delegate.Method`. When no `Func`/`Action` fits, the compiler emits
an internal delegate type such as `<>f__AnonymousDelegate0`.

## 6. Performance and allocation

- Same as before: non-capturing lambdas are cached in a static field; capturing lambdas allocate a closure and a delegate.
- `var` does not change allocation behaviour.

## 7. Pitfalls

- `var f = x => x;` still fails: parameter types must be explicit for a natural type.
- Overloaded method groups have no natural type in C# 10 (`var p = int.Parse;` fails).
- `[Attr] x => x` does not parse; write `[Attr] (x) => x`.
- Anonymous delegate types are not `Func<>`; code that expects `Func<...>` must declare the type explicitly.

## 8. Concerns and limitations

- Readability: `var` hides the delegate type. Use it when the lambda's parameter list makes the type obvious.
- Breaking change risk (C# 10): overload resolution between `Delegate` / `Expression` and other parameter types
  can pick a different overload than before because lambdas now convert to `Delegate`. Rare, but check APIs with such overload pairs.

## 9. When to use: clean-code guidance

### 9.1 Use when
- Local helper lambdas with explicit parameter types.
- Minimal APIs and other `Delegate`-accepting frameworks.

### 9.2 Avoid when
- The delegate type documents intent better (`Predicate<Unit>`, a custom `delegate`): declare it explicitly.
- A local function would be clearer (recursion, multiple statements, no need for a delegate).

### 9.3 Good versus bad

```csharp
// Good
var isAlive = (Unit u) => u.Hp > 0;

// Bad: delegate variable for a one-off helper that should be a local function
var compute = (int a, int b) => { /* 20 lines */ return a + b; };
int Compute(int a, int b) { /* better */ return a + b; }
```

## 10. Related features

- [../12/lambda-default-parameters.md](../12/lambda-default-parameters.md): default values (C# 12).
- [../13/method-group-natural-type.md](../13/method-group-natural-type.md): better method group inference (C# 13).
- [../14/lambda-parameter-modifiers.md](../14/lambda-parameter-modifiers.md): `out` / `ref` without types (C# 14).
- [../11/method-group-delegate-caching.md](../11/method-group-delegate-caching.md): delegate caching.

## 11. References

- [Lambda expressions](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/lambda-expressions)
- [Proposal: Lambda improvements](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-10.0/lambda-improvements)
