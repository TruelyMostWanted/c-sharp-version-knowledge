---
id: cs13-escape-sequence-e
csharp: 13
tfm: net9.0
status: stable
tags: [escape, ansi, console, string]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/char#literals
modernizes: \u001b and \x1b
---
# `\e` escape sequence (C# 13)

> **Abstract.** `\e` in a character or string literal is the ESCAPE character (U+001B). It replaces `\u001b` and the
> error-prone `\x1b`. Its main use is ANSI terminal control sequences (colours, cursor movement).

| | |
|---|---|
| Introduced | C# 13 · .NET 9 (`net9.0`) |
| Status | Stable |
| Kind | Syntax (lexer) |
| Runtime support needed | No |
| Audience | Console tools, logging |
| Modernize category | AUTO |

## 1. Motivation

ANSI sequences start with ESC. C# had no short escape for it, and the common `\x1b` has a trap: `\x` takes **one to four**
hex digits, so `"\x1b1m"` is not ESC followed by `1m`; it becomes `\x1b1` (a different character) followed by `m`.

## 2. The feature explained

### 2.1 Syntax

```csharp
char esc = '\e';
string red = "\e[31m";
```

### 2.2 Semantics and rules

1. `\e` equals `\u001b` in `char` and `string` literals (regular and interpolated). Not in verbatim or raw strings (no escapes there).

## 3. Code examples

### 3.1 Minimal example

```csharp
Console.WriteLine("\e[31mError\e[0m");   // red text in ANSI terminals
```

### 3.2 Extended examples

A tiny colour helper:

```csharp
public static class Ansi
{
    public const string Reset  = "\e[0m";
    public const string Red    = "\e[31m";
    public const string Green  = "\e[32m";
    public const string Bold   = "\e[1m";

    public static string Color(string text, string code) => $"{code}{text}{Reset}";
}

Console.WriteLine(Ansi.Color("Build OK", Ansi.Green));
```

Cursor control in a progress display:

```csharp
Console.Write($"\e[2K\rProgress: {percent,3}%");   // clear line, return to column 0
```

Raw strings have no escapes; use interpolation of a constant:

```csharp
const string Esc = "\e";
string banner = $"""{Esc}[1mTitle{Esc}[0m""";
```

## 4. Before this feature

```csharp
Console.WriteLine("\u001b[31mError\u001b[0m");
Console.WriteLine("\x1b[31mError\x1b[0m");   // works here, but "\x1b1m" would not
```

## 5. Under the hood

Lexer-only. The literal contains the character U+001B; IL is identical to `\u001b`.

## 6. Performance and allocation

None.

## 7. Pitfalls

- Windows terminals older than Windows 10 1511 (and some redirected outputs) do not interpret ANSI codes; output becomes garbage. Check `Console.IsOutputRedirected`.

## 8. Concerns and limitations

- Godot's output panel does not render ANSI colours; use `GD.PrintRich("[color=red]...[/color]")` there.

## 9. When to use: clean-code guidance

### 9.1 Use when
- Always instead of `\u001b` / `\x1b` on C# 13+.

### 9.2 Avoid when
- Not applicable.

### 9.3 Good versus bad

```csharp
// Good
const string Bold = "\e[1m";

// Bad
const string Bold2 = "\x1b[1m";
```

## 10. Related features

- [../11/raw-string-literals.md](../11/raw-string-literals.md)

## 11. References

- [What's new in C# 13: new escape sequence](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-13#new-escape-sequence)
- [char literals and escape sequences](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/char#literals)
