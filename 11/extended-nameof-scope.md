---
id: cs11-extended-nameof-scope
csharp: 11
tfm: net7.0
status: stable
tags: [nameof, attributes, nullable]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/nameof
modernizes: string literals with parameter names inside attributes
---
# Extended `nameof` scope (C# 11)

> **Abstract.** In attributes on a method, its parameters or its return value, `nameof` can now refer to that
> method's **parameters and type parameters**. Nullable-analysis attributes like
> `[NotNullIfNotNull(nameof(path))]` stop relying on string literals that break silently on rename.

| | |
|---|---|
| Introduced | C# 11 · .NET 7 (`net7.0`) |
| Status | Stable |
| Kind | Semantics (name lookup) |
| Runtime support needed | No |
| Audience | Everyday code, library authors |
| Modernize category | AUTO |

## 1. Motivation

Attributes on a method are outside the method's body, so its parameters were not in scope and `nameof(param)`
did not compile. Nullable attributes (`NotNullIfNotNull`, `MemberNotNullWhen` ...), `CallerArgumentExpression`
and others take parameter names, which therefore had to be literal strings.

## 2. The feature explained

### 2.1 Syntax

```csharp
[return: NotNullIfNotNull(nameof(path))]
string? Normalize(string? path);

void M<T>([SomeAttr(nameof(T))] T value);
```

### 2.2 Semantics and rules

1. Inside an attribute on a method or local function, `nameof` may reference the method's parameters and type parameters.
2. Inside an attribute on a parameter, `nameof` may reference the other parameters of the same method.
3. Also applies to attributes on lambdas' parameters and return values.
4. Only `nameof` sees these names; using a parameter as a value inside an attribute is still not possible.

## 3. Code examples

### 3.1 Minimal example

```csharp
[return: NotNullIfNotNull(nameof(path))]
public static string? Normalize(string? path) => path?.Replace('\\', '/');
```

### 3.2 Extended examples

`CallerArgumentExpression` with `nameof`:

```csharp
public static void Require(bool condition,
    [CallerArgumentExpression(nameof(condition))] string? expr = null)
{
    if (!condition) throw new InvalidOperationException(expr);
}
```

`TryX` pattern with `NotNullWhen` and `MaybeNullWhen`:

```csharp
public bool TryGetUnit(int id, [NotNullWhen(true)] out Unit? unit) { ... }

[MemberNotNullWhen(true, nameof(Target))]
public bool HasTarget => Target is not null;
public Node3D? Target { get; private set; }
```

Generic type parameter names:

```csharp
[Description($"Converts values of {nameof(T)}")]   // "Converts values of T"
public static T Convert<T>(object value) => (T)value;
```

## 4. Before this feature

```csharp
[return: NotNullIfNotNull("path")]                         // breaks silently on rename
public static string? Normalize(string? path) => ...;

public static void Require(bool condition,
    [CallerArgumentExpression("condition")] string? expr = null) { }
```

## 5. Under the hood

`nameof(path)` is a compile-time constant `"path"`. The attribute blob in metadata is identical to the string-literal form.

## 6. Performance and allocation

None.

## 7. Pitfalls

- A local or member with the same name as the parameter can shadow the lookup in unusual scopes; the compiler resolves according to normal rules. In practice: none.

## 8. Concerns and limitations

- None. Safe automatic refactoring (`dotnet format` / IDE quick fix).

## 9. When to use: clean-code guidance

### 9.1 Use when
- Always replace parameter-name string literals in attributes with `nameof`.

### 9.2 Avoid when
- Never.

### 9.3 Good versus bad

```csharp
// Good
[return: NotNullIfNotNull(nameof(input))]

// Bad
[return: NotNullIfNotNull("input")]
```

## 10. Related features

- [../10/caller-argument-expression.md](../10/caller-argument-expression.md)
- [../14/nameof-unbound-generic-types.md](../14/nameof-unbound-generic-types.md)

## 11. References

- [nameof expression](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/nameof)
- [Proposal: Extended nameof scope](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-11.0/extended-nameof-scope)
- [Nullable static analysis attributes](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/attributes/nullable-analysis)
