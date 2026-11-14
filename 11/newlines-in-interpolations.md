---
id: cs11-newlines-in-interpolations
csharp: 11
tfm: net7.0
status: stable
tags: [string-interpolation, formatting]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-11#newlines-in-string-interpolation-expressions
---
# Newlines in string interpolation expressions (C# 11)

> **Abstract.** The expression inside an interpolation hole `{ ... }` may span several lines, also in normal
> (non-verbatim) interpolated strings. This makes `switch` expressions, LINQ queries and long calls inside holes
> possible. Use it sparingly; a named local is usually clearer.

| | |
|---|---|
| Introduced | C# 11 · .NET 7 (`net7.0`) |
| Status | Stable |
| Kind | Syntax (lexer) |
| Runtime support needed | No |
| Audience | Everyday code |
| Modernize category | n/a |

## 1. Motivation

In C# 10 and earlier, a non-verbatim interpolated string `$"..."` had to fit on one line, including its holes.
A `switch` expression or a multi-line lambda inside a hole was impossible unless the whole string was verbatim (`$@"..."`),
which changes escape handling.

## 2. The feature explained

### 2.1 Syntax

```csharp
$"Text {expression
        spanning lines} more text"
```

### 2.2 Semantics and rules

1. Newlines are allowed **inside holes** only. The literal text parts of a non-verbatim string still cannot contain raw newlines (use `\n` or a raw string literal).
2. Any expression is allowed, including `switch` expressions, lambdas and query expressions.
3. Format and alignment specifiers follow the expression as usual (`{expr,10:F2}`).

## 3. Code examples

### 3.1 Minimal example

```csharp
string label = $"State: {state switch
{
    PlayerState.Idle    => "idle",
    PlayerState.Running => "running",
    _                   => "other",
}}";
```

### 3.2 Extended examples

LINQ inside a hole:

```csharp
string summary = $"Top: {string.Join(", ", units
    .Where(u => u.Hp > 0)
    .OrderByDescending(u => u.Score)
    .Take(3)
    .Select(u => u.Name))}";
```

Long method call with named arguments:

```csharp
string tooltip = $"Damage: {CalculateDamage(
    attacker: player,
    target: enemy,
    includeCrit: false):F1}";
```

## 4. Before this feature

```csharp
string stateText = state switch
{
    PlayerState.Idle => "idle",
    PlayerState.Running => "running",
    _ => "other",
};
string label = $"State: {stateText}";
```

This is still often the better version.

## 5. Under the hood

Pure lexer change. The generated code is the same as for a single-line hole (see interpolated string handlers).

## 6. Performance and allocation

None beyond the normal interpolation cost.

## 7. Pitfalls

- Newlines in holes do **not** put newlines into the resulting string. Only the literal parts produce output.
- Nested quotes and braces across lines are hard to read and easy to break when editing.

## 8. Concerns and limitations

- Readability is the main concern; formatters (`dotnet format`) may indent multi-line holes awkwardly.

## 9. When to use: clean-code guidance

### 9.1 Use when
- A short `switch` expression is the natural content of a hole and a local would only add a name with no meaning.

### 9.2 Avoid when
- The hole contains logic worth naming or testing. Extract a local or a method.

### 9.3 Good versus bad

```csharp
// Good: short, obvious mapping
$"Difficulty: {level switch { 1 => "easy", 2 => "normal", _ => "hard" }}"

// Bad: business logic hidden in a string
$"Price: {items
    .Where(i => i.Category == cat && !i.Discontinued)
    .Sum(i => i.Price * (1 - DiscountFor(i, customer, DateTime.Now))):C}"
```

## 10. Related features

- [raw-string-literals.md](raw-string-literals.md): multi-line literal text.
- [../10/interpolated-string-handlers.md](../10/interpolated-string-handlers.md)

## 11. References

- [What's new in C# 11: newlines in interpolations](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-11#newlines-in-string-interpolation-expressions)
- [String interpolation](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/tokens/interpolated)
