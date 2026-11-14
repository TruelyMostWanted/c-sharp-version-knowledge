---
id: cs10-caller-argument-expression
csharp: 10
tfm: net6.0
status: stable
tags: [diagnostics, guard, assert, attributes]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/attributes/caller-information#argument-expressions
modernizes: assert/guard helpers that take a manual message string
---
# `[CallerArgumentExpression]` (C# 10)

> **Abstract.** `[CallerArgumentExpression("param")]` on an optional `string` parameter makes the compiler pass
> the **source text** of another argument at the call site. Guards and asserts can therefore report
> `"hp <= maxHp"` instead of a generic message. The .NET throw helpers (`ArgumentNullException.ThrowIfNull`,
> `ArgumentOutOfRangeException.ThrowIfNegative`, ...) are built on it; prefer them over hand-written checks.

| | |
|---|---|
| Introduced | C# 10 · .NET 6 (`net6.0`) |
| Status | Stable |
| Kind | Compiler behaviour (caller information) |
| Runtime support needed | Attribute type ships with .NET Core 3.0+ / .NET 6; can be declared manually for older targets |
| Audience | Everyday code (via BCL helpers), library authors (own guards) |
| Modernize category | AUTO for `ThrowIfNull` and friends |

## 1. Motivation

Argument validation used to repeat the parameter name and often the condition itself:

```csharp
if (player == null) throw new ArgumentNullException(nameof(player));
Debug.Assert(hp <= maxHp, "hp <= maxHp");      // condition written twice, drifts on refactoring
```

`[CallerFilePath]`, `[CallerLineNumber]` and `[CallerMemberName]` (C# 5) already let the compiler fill in
call-site information. C# 10 adds the missing piece: the expression text of an argument.

## 2. The feature explained

### 2.1 Syntax

```csharp
void Check(bool condition,
           [CallerArgumentExpression(nameof(condition))] string? expression = null);
```

### 2.2 Semantics and rules

1. The attribute names another parameter of the same method (use `nameof`, allowed since C# 11 in this position; in C# 10 write the string `"condition"`).
2. The annotated parameter must be optional (have a default value) and be of a type a `string` converts to.
3. If the caller passes the argument explicitly, the compiler does **not** override it.
4. The text is the argument as written in source, whitespace normalized only as the compiler sees it (comments stripped).
5. It works for constructors, methods, indexers and local functions; not for lambdas called through delegates (the call site is the delegate invocation).

## 3. Code examples

### 3.1 Minimal example

```csharp
public static class Guard
{
    public static void That(bool condition,
        [CallerArgumentExpression("condition")] string? expression = null)
    {
        if (!condition) throw new InvalidOperationException($"Check failed: {expression}");
    }
}

Guard.That(hp <= maxHp);   // InvalidOperationException: "Check failed: hp <= maxHp"
```

### 3.2 Extended examples

Built-in throw helpers (prefer these):

```csharp
public void Equip(Item item, int slot)
{
    ArgumentNullException.ThrowIfNull(item);                       // .NET 6
    ArgumentOutOfRangeException.ThrowIfNegative(slot);              // .NET 8
    ArgumentOutOfRangeException.ThrowIfGreaterThanOrEqual(slot, SlotCount); // .NET 8
    ArgumentException.ThrowIfNullOrEmpty(item.Id);                  // .NET 7
    ...
}
```

A reusable guard that carries the value and the expression:

```csharp
public static class Ensure
{
    public static T NotDefault<T>(T value,
        [CallerArgumentExpression("value")] string? expr = null) where T : struct, IEquatable<T>
    {
        if (value.Equals(default)) throw new ArgumentException($"'{expr}' must not be default.", expr);
        return value;
    }
}

_entity = Ensure.NotDefault(request.EntityId);   // "'request.EntityId' must not be default."
```

Test assertions that print the failing expression:

```csharp
public static void IsTrue(bool condition,
    [CallerArgumentExpression("condition")] string? expr = null,
    [CallerFilePath] string file = "", [CallerLineNumber] int line = 0)
{
    if (!condition) throw new AssertionException($"{file}({line}): expected true: {expr}");
}
```

## 4. Before this feature

```csharp
if (item == null) throw new ArgumentNullException(nameof(item));
if (slot < 0) throw new ArgumentOutOfRangeException(nameof(slot), slot, "must be >= 0");
Debug.Assert(hp <= maxHp, "hp <= maxHp");
```

After:

```csharp
ArgumentNullException.ThrowIfNull(item);
ArgumentOutOfRangeException.ThrowIfNegative(slot);
Guard.That(hp <= maxHp);      // own helper from 3.1
```

Note: whether a given BCL method (for example `Debug.Assert`) captures the expression depends on the
.NET version's API; check the method's signature for a `[CallerArgumentExpression]` parameter.

## 5. Under the hood

The attribute is resolved entirely at compile time. The call

```csharp
Guard.That(hp <= maxHp);
```

is compiled exactly as if you had written

```csharp
Guard.That(hp <= maxHp, "hp <= maxHp");
```

In IL the string is a plain `ldstr "hp <= maxHp"`. No reflection, no runtime lookup.

## 6. Performance and allocation

- None at runtime: the string is an interned literal.
- The BCL throw helpers are small and the throwing path is moved out of line (`[DoesNotReturn]` helper),
  which keeps the calling method small and inlinable. Hand-written `if (...) throw new ...` inlines the
  throw site into the caller; this is one reason to prefer the helpers in hot code.

## 7. Pitfalls

- Passing the expression parameter explicitly disables the feature for that call (sometimes done by accident when adding a new optional parameter after it).
- Wrapping a guard in another helper captures the **wrapper's** parameter name, not the original expression,
  unless the wrapper forwards its own `[CallerArgumentExpression]` parameter.
- The expression text can leak implementation details (variable names) into exception messages shown to users. Keep user-facing messages separate.

## 8. Concerns and limitations

- Binary compatibility: adding an optional `[CallerArgumentExpression]` parameter to a public method changes its signature (breaking for compiled callers). Add an overload instead.
- Older targets (`netstandard2.0`): declare `System.Runtime.CompilerServices.CallerArgumentExpressionAttribute` yourself (internal) or use a polyfill package; the compiler recognizes it by name.
- Godot: works as normal; `GD.PushError` / custom assert helpers can use it for clearer editor output.

## 9. When to use: clean-code guidance

### 9.1 Use when
- Validating arguments: use `ThrowIfNull`, `ThrowIfNullOrEmpty`, `ThrowIfNegative`, `ThrowIfZero`, `ThrowIfGreaterThan`, ...
- Writing your own guard, assert or test helper.

### 9.2 Avoid when
- The message is meant for end users (localize instead).

### 9.3 Good versus bad

```csharp
// Good
ArgumentNullException.ThrowIfNull(config);

// Bad: duplicated name, drifts on rename, larger method body
if (config is null) throw new ArgumentNullException("config");
```

## 10. Related features

- [../11/extended-nameof-scope.md](../11/extended-nameof-scope.md): `nameof(condition)` inside the attribute.
- [interpolated-string-handlers.md](interpolated-string-handlers.md): `Debug.Assert(cond, $"...")` message handlers.

## 11. References

- [Caller information: argument expressions](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/attributes/caller-information#argument-expressions)
- [Proposal: CallerArgumentExpression](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-10.0/caller-argument-expression)
- [ArgumentNullException.ThrowIfNull](https://learn.microsoft.com/en-us/dotnet/api/system.argumentnullexception.throwifnull)
