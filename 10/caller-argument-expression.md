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

Captures the **source text** of another argument as a string.

```csharp
public static class Guard
{
    public static void That(bool condition,
        [CallerArgumentExpression(nameof(condition))] string? expression = null)
    {
        if (!condition) throw new InvalidOperationException($"Check failed: {expression}");
    }
}

Guard.That(hp <= maxHp);   // message: "Check failed: hp <= maxHp"
```

## Rules
- The parameter needs a default value.
- Built-in helpers use it: `ArgumentNullException.ThrowIfNull(x)` (.NET 6), `ArgumentOutOfRangeException.ThrowIfNegative(x)` (.NET 8).
  Prefer these over hand-written guards.
