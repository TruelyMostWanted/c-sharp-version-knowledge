---
id: cs13-method-group-natural-type
csharp: 13
tfm: net9.0
status: stable
tags: [method-group, var, delegate, inference]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-13#method-group-natural-type
---
# Method group natural type improvements (C# 13)

> **Abstract.** When the compiler infers the natural delegate type of a method group (`var f = Parse;`), it now
> discards inapplicable candidates (wrong generic arity, unsatisfied constraints, extension methods that do not fit)
> **scope by scope**, and moves to an outer scope (for example extension methods) only if no candidate remained.
> Fewer method groups end up "ambiguous", so `var` with method groups works in more cases. No new syntax.

| | |
|---|---|
| Introduced | C# 13 · .NET 9 (`net9.0`) |
| Status | Stable |
| Kind | Semantics (type inference) |
| Runtime support needed | No |
| Audience | Everyday code |
| Modernize category | n/a (automatic) |

## 1. Motivation

C# 10 gave method groups a natural type when there is exactly one candidate. The algorithm collected **all**
candidates from every scope (instance methods, then all extension methods in scope) before deciding. A generic
method that could never apply, or an unrelated extension method with the same name, made the group ambiguous and
`var f = obj.M;` failed.

## 2. The feature explained

### 2.1 Syntax

No new syntax.

### 2.2 Semantics and rules

1. Candidates are considered scope by scope: instance methods first, then each extension method scope outward.
2. Within a scope, candidates that cannot apply are pruned: generic methods whose type arguments cannot be inferred or whose arity does not match, and those with unsatisfied constraints.
3. If exactly one candidate remains in a scope, that is the natural type; if none remain, the next scope is tried; if several remain, there is no natural type.
4. Applies wherever natural types of method groups are used (`var`, conversion to `Delegate`, `object`).

## 3. Code examples

### 3.1 Minimal example

```csharp
public sealed class Greeter
{
    public string Hello(string name) => $"Hello {name}";
}

public static class GreeterExtensions
{
    public static string Hello<T>(this Greeter g, T value, int times) where T : struct => "...";
}

var g = new Greeter();
var f = g.Hello;   // C# 13: Func<string, string> (the instance method); before: possible ambiguity error
```

### 3.2 Extended examples

Registering handlers by method group:

```csharp
var handlers = new Dictionary<string, Delegate>
{
    ["save"] = commands.Save,     // natural type found once irrelevant overloads are pruned
    ["load"] = commands.Load,
};
```

## 4. Before this feature

```csharp
Func<string, string> f = g.Hello;   // explicit delegate type needed
```

## 5. Under the hood

Compile-time inference only; the generated delegate creation is unchanged.

## 6. Performance and allocation

None. (Delegate allocation rules: see [../11/method-group-delegate-caching.md](../11/method-group-delegate-caching.md).)

## 7. Pitfalls

- Adding a new overload in the **same** scope can still make `var f = M;` ambiguous later; for public or widely used methods, declare the delegate type explicitly.

## 8. Concerns and limitations

- Minor breaking-change potential: code that previously picked an extension method may now pick an instance method found in an inner scope. Rare.

## 9. When to use: clean-code guidance

### 9.1 Use when
- Local `var` of a method group where the delegate type is obvious from context.

### 9.2 Avoid when
- Fields, properties and public APIs: spell out the delegate type.

### 9.3 Good versus bad

```csharp
// Good
var isValid = Validator.IsValidName;   // single obvious candidate

// Bad: the inferred type is not obvious to readers
var x = service.Process;
```

## 10. Related features

- [../10/lambda-improvements.md](../10/lambda-improvements.md)

## 11. References

- [What's new in C# 13: method group natural type](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-13#method-group-natural-type)
- [Proposal: Method group natural type improvements](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-13.0/method-group-natural-type-improvements)
