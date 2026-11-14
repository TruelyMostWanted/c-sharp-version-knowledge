---
id: cs11-method-group-delegate-caching
csharp: 11
tfm: net7.0
status: stable
tags: [delegate, method-group, allocation, performance]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-11#improved-method-group-conversion-to-delegate
---
# Improved method group conversion to delegate (C# 11)

> **Abstract.** When a **static** method group is converted to a delegate (`list.Sort(CompareByDistance)`,
> `items.Select(Transform)`), the compiler may now cache the delegate instance in a static field, exactly as it
> already did for non-capturing lambdas. Repeated conversions no longer allocate. Instance method groups still
> allocate one delegate per conversion.

| | |
|---|---|
| Introduced | C# 11 · .NET 7 (`net7.0`) |
| Status | Stable |
| Kind | Compiler behaviour (code generation) |
| Runtime support needed | No |
| Audience | Everyday code, performance-sensitive code |
| Modernize category | n/a (automatic; undo old workarounds) |

## 1. Motivation

Before C# 11 the specification required a **new** delegate instance for every method group conversion.
`items.Select(Transform)` allocated a `Func<,>` on every call, while `items.Select(x => Transform(x))` did not
(lambdas without captures were cached). Performance guides recommended the awkward lambda wrapper.

## 2. The feature explained

### 2.1 Syntax

No new syntax.

### 2.2 Semantics and rules

1. The specification now **permits** reusing a delegate instance for a method group conversion of a static method.
2. The Roslyn compiler caches conversions of **static** methods in a compiler-generated static field.
3. Conversions of **instance** methods (`this.Method`, `obj.Method`) still create a new delegate each time (the target object differs).
4. Generic static methods are cached per instantiation.
5. Behavioural consequence: reference equality between two converted delegates may now be `true`. Code must not rely on either outcome.

## 3. Code examples

### 3.1 Minimal example

```csharp
static int CompareByHp(Unit a, Unit b) => a.Hp.CompareTo(b.Hp);

void SortEveryFrame(List<Unit> units) => units.Sort(CompareByHp);   // C# 11: no delegate allocation per call
```

### 3.2 Extended examples

LINQ with static helpers:

```csharp
static bool IsAlive(Unit u) => u.Hp > 0;
static string NameOf(Unit u) => u.Name;

var names = units.Where(IsAlive).Select(NameOf).ToList();   // delegates cached
```

Event subscriptions (static handlers):

```csharp
static void OnUnhandled(object? s, UnhandledExceptionEventArgs e) => Log(e.ExceptionObject);
AppDomain.CurrentDomain.UnhandledException += OnUnhandled;
```

Instance methods still allocate; cache them yourself if it matters:

```csharp
private readonly Comparison<Unit> _byDistance;
public Squad() => _byDistance = CompareByDistanceToLeader;   // allocate once

void Update() => _members.Sort(_byDistance);
```

## 4. Before this feature

```csharp
units.Sort((a, b) => CompareByHp(a, b));   // workaround: cached lambda instead of method group
```

## 5. Under the hood

```csharp
// units.Sort(CompareByHp);   (C# 11)
[CompilerGenerated]
internal static class <>O
{
    public static Comparison<Unit> <0>__CompareByHp;
}

units.Sort(<>O.<0>__CompareByHp ??= new Comparison<Unit>(CompareByHp));
```

The lazily initialized static field is the same technique used for static lambdas (`<>c.<>9__0_0`).

## 6. Performance and allocation

- Saves one delegate allocation (~64 bytes) per conversion of a static method group.
- Instance method groups and capturing lambdas are unchanged.

## 7. Pitfalls

- Unsubscribing events with a **new** delegate still works (delegate equality is by target and method, not reference).
- Code that compared delegates with `ReferenceEquals` may observe different results. Such code was fragile anyway.

## 8. Concerns and limitations

- Applies when compiling with C# 11 or newer; no runtime requirement.
- Godot: `Callable.From(StaticMethod)` wraps the delegate; caching the `Callable` yourself is still useful for frequently connected signals.

## 9. When to use: clean-code guidance

### 9.1 Use when
- Prefer method groups of static methods; they are now both readable and allocation-free.
- When modernizing, you may revert `x => Static(x)` wrappers that existed only for performance.

### 9.2 Avoid when
- Not applicable. For instance methods in hot paths, cache the delegate in a field.

### 9.3 Good versus bad

```csharp
// Good (C# 11+)
var alive = units.Where(IsAlive);

// Unnecessary on C# 11+
var alive2 = units.Where(u => IsAlive(u));
```

## 10. Related features

- [../10/lambda-improvements.md](../10/lambda-improvements.md)
- [../13/method-group-natural-type.md](../13/method-group-natural-type.md)

## 11. References

- [What's new in C# 11: improved method group conversion](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-11#improved-method-group-conversion-to-delegate)
- [Delegate types: method group conversions](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/delegates/using-delegates)
