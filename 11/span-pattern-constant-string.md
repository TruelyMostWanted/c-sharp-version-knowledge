---
id: cs11-span-pattern-constant-string
csharp: 11
tfm: net7.0
status: stable
tags: [span, pattern-matching, string, parsing]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-11#pattern-match-spanchar-or-readonlyspanchar-on-a-constant-string
modernizes: span.SequenceEqual("text") or allocating ToString() before comparing
---
# Pattern match `Span<char>` / `ReadOnlySpan<char>` on a constant string (C# 11)

> **Abstract.** A `Span<char>` or `ReadOnlySpan<char>` can be matched against **constant strings** with `is` and
> `switch`: `key is "width"`, `key switch { "width" => ..., "height" => ... }`. The comparison is ordinal and
> allocation-free, which makes span-based parsers as readable as string-based ones.

| | |
|---|---|
| Introduced | C# 11 · .NET 7 (`net7.0`) |
| Status | Stable |
| Kind | Semantics (pattern matching) |
| Runtime support needed | `MemoryExtensions.SequenceEqual` (available since .NET Core 2.1) |
| Audience | Parsing and performance-sensitive code |
| Modernize category | AUTO |

## 1. Motivation

High-performance parsers slice input into `ReadOnlySpan<char>` to avoid substrings. Comparing a slice with a keyword
required `span.SequenceEqual("width")` chains or `span.ToString() == "width"`, which allocates and defeats the purpose.
`switch` on spans was impossible.

## 2. The feature explained

### 2.1 Syntax

```csharp
span is "text"
span is "a" or "b"
span switch { "a" => 1, "b" => 2, _ => 0 }
```

### 2.2 Semantics and rules

1. Constant string patterns are allowed when the input type is `Span<char>` or `ReadOnlySpan<char>`.
2. Comparison is **ordinal** (exact chars), like `string ==`.
3. `null` constant patterns are not allowed against spans; an empty span matches `""`.
4. Can be combined with `or`, `and`, `not` and list patterns.

## 3. Code examples

### 3.1 Minimal example

```csharp
ReadOnlySpan<char> key = line.AsSpan(0, line.IndexOf('='));

int value = key switch
{
    "width"  => 1,
    "height" => 2,
    _        => 0,
};

if (key is "fullscreen") { }
```

### 3.2 Extended examples

Config parser without allocations:

```csharp
static void ApplySetting(ReadOnlySpan<char> line, Settings s)
{
    int eq = line.IndexOf('=');
    if (eq < 0) return;

    ReadOnlySpan<char> key = line[..eq].Trim();
    ReadOnlySpan<char> val = line[(eq + 1)..].Trim();

    switch (key)
    {
        case "width":       s.Width = int.Parse(val); break;
        case "height":      s.Height = int.Parse(val); break;
        case "vsync":       s.VSync = val is "true" or "1"; break;
        case "fullscreen" or "fs": s.Fullscreen = bool.Parse(val); break;
    }
}
```

Tokenizer keywords:

```csharp
TokenKind Classify(ReadOnlySpan<char> word) => word switch
{
    "if" or "else" or "while" => TokenKind.Keyword,
    "true" or "false"         => TokenKind.Boolean,
    []                        => TokenKind.Empty,          // list pattern: empty span
    _                         => TokenKind.Identifier,
};
```

Case-insensitive comparisons need a different API:

```csharp
bool isYes = answer.Equals("yes", StringComparison.OrdinalIgnoreCase);   // patterns are ordinal only
```

## 4. Before this feature

```csharp
if (key.SequenceEqual("width")) { ... }
else if (key.SequenceEqual("height")) { ... }

switch (key.ToString())     // allocates a string per call
{
    case "width": ...
}
```

## 5. Under the hood

Each constant test becomes a call to `MemoryExtensions.SequenceEqual(span, "literal")` (the string converts to
`ReadOnlySpan<char>`). In `switch` statements over many constants, the compiler may first branch on `span.Length`
and individual characters before comparing, similar to string switches.

## 6. Performance and allocation

- No allocations; vectorized `SequenceEqual`.
- Faster than `ToString()` + `switch` in all cases; comparable to a hand-written chain.

## 7. Pitfalls

- Case-sensitive only. `"Width"` does not match `"width"`.
- Does not apply to `Memory<char>`; use `.Span`.
- `char[]` is not matched directly (convert with `AsSpan()`).

## 8. Concerns and limitations

- None for compatibility.
- Godot: parse `FileAccess.GetLine()` results or `ConfigFile` text via spans to avoid per-line garbage.

## 9. When to use: clean-code guidance

### 9.1 Use when
- Any parser or tokenizer that already works with spans.

### 9.2 Avoid when
- You need culture-aware or case-insensitive matching.

### 9.3 Good versus bad

```csharp
// Good
if (cmd is "quit" or "exit") Quit();

// Bad: allocation per call
if (cmd.ToString() == "quit") Quit();
```

## 10. Related features

- [list-patterns.md](list-patterns.md)
- [utf8-string-literals.md](utf8-string-literals.md): byte-level parsing with `u8`.

## 11. References

- [What's new in C# 11: pattern match span on constant string](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-11#pattern-match-spanchar-or-readonlyspanchar-on-a-constant-string)
- [Proposal: Pattern match Span<char> on a constant string](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-11.0/pattern-match-span-of-char-on-string)
