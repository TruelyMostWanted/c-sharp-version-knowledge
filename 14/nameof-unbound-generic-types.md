---
id: cs14-nameof-unbound-generic-types
csharp: 14
tfm: net10.0
status: stable
tags: [nameof, generics]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/nameof
modernizes: nameof(List<int>) written only to get the name
---
# `nameof` with unbound generic types (C# 14)

> **Abstract.** `nameof` accepts **unbound** generic types: `nameof(List<>)` is `"List"`, `nameof(Dictionary<,>)` is
> `"Dictionary"`. Before C# 14 you had to invent type arguments (`nameof(List<int>)`) just to get a name.

| | |
|---|---|
| Introduced | C# 14 · .NET 10 (`net10.0`) |
| Status | Stable |
| Kind | Syntax |
| Runtime support needed | No |
| Audience | Everyday code, diagnostics, generators |
| Modernize category | AUTO |

## 1. Motivation

`typeof(List<>)` has always worked for unbound generics, but `nameof(List<>)` did not. Code that needed the name used
arbitrary type arguments, which looked meaningful but were not.

## 2. The feature explained

### 2.1 Syntax

```csharp
nameof(List<>)
nameof(Dictionary<,>)
nameof(Outer<>.Inner)
nameof(List<>.Count)     // member access on an unbound type
```

### 2.2 Semantics and rules

1. The result is the simple name without arity or type arguments (`"List"`, not ``"List`1"``).
2. Members of unbound generic types can be named (`nameof(List<>.Count)` → `"Count"`).
3. Partially bound types (`Dictionary<int,>`) are not allowed.
4. Like every `nameof`, the result is a compile-time constant.

## 3. Code examples

### 3.1 Minimal example

```csharp
nameof(List<>)          // "List"
nameof(Dictionary<,>)   // "Dictionary"
```

### 3.2 Extended examples

Diagnostics and logging:

```csharp
_logger.LogWarning("{Type} pool exhausted", nameof(ObjectPool<>));
```

Registries keyed by generic type name:

```csharp
private static readonly Dictionary<string, Type> s_systems = new()
{
    [nameof(MovementSystem<>)] = typeof(MovementSystem<>),
    [nameof(CombatSystem<>)]   = typeof(CombatSystem<>),
};
```

Constant strings (attributes, switch labels):

```csharp
const string RepoName = nameof(Repository<>);    // "Repository"
```

## 4. Before this feature

```csharp
nameof(List<int>)               // "List", but int is meaningless
typeof(List<>).Name             // "List`1", runtime, includes arity
```

## 5. Under the hood

Compile-time constant: `ldstr "List"`.

## 6. Performance and allocation

None.

## 7. Pitfalls

- The name does not include arity; `nameof(Action<>)` and `nameof(Action<,>)` are both `"Action"`. Do not use names as unique keys across arities.

## 8. Concerns and limitations

- None.

## 9. When to use: clean-code guidance

### 9.1 Use when
- You need the name of a generic type or one of its members.

### 9.2 Avoid when
- You need a unique identifier: use `typeof(T).FullName` or the `Type` itself.

### 9.3 Good versus bad

```csharp
// Good
throw new InvalidOperationException($"{nameof(Pool<>)} is empty");

// Bad (on C# 14)
throw new InvalidOperationException($"{nameof(Pool<object>)} is empty");
```

## 10. Related features

- [../11/extended-nameof-scope.md](../11/extended-nameof-scope.md)

## 11. References

- [nameof expression](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/nameof)
- [Proposal: Unbound generic types in nameof](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-14.0/unbound-generic-types-in-nameof)
