---
id: cs10-interpolated-string-handlers
csharp: 10
tfm: net6.0
status: stable
needs-runtime: true (DefaultInterpolatedStringHandler, .NET 6)
tags: [string-interpolation, performance, logging, allocation]
source: https://learn.microsoft.com/en-us/dotnet/csharp/advanced-topics/performance/interpolated-string-handler
modernizes: string.Format and + concatenation
---
# Interpolated string handlers (C# 10)

> **Abstract.** An interpolated string `$"..."` can now be passed to a **handler type** instead of being turned
> into a `string` first. The compiler emits a sequence of `AppendLiteral` / `AppendFormatted` calls on the handler.
> The BCL uses this to make ordinary interpolation faster than `string.Format`, to write into spans and builders
> without intermediate strings, and to skip formatting entirely when a message is not needed (disabled asserts or logs).

| | |
|---|---|
| Introduced | C# 10 · .NET 6 (`net6.0`) |
| Status | Stable |
| Kind | Compiler behaviour + library pattern |
| Runtime support needed | `DefaultInterpolatedStringHandler` and handler-aware APIs (.NET 6) |
| Audience | Everyday code (implicitly); library authors (custom handlers) |
| Modernize category | PREFER interpolation over `string.Format` / `+` |

## 1. Motivation

Before C# 10, `$"HP {hp}/{maxHp}"` compiled to `string.Format("HP {0}/{1}", hp, maxHp)`:

- the format string was parsed at runtime on every call,
- value-type arguments were **boxed** into an `object[]`,
- the result was always a new `string`, even when the callee only wanted to append it to a buffer
  or discard it (log level disabled, assert passed).

## 2. The feature explained

### 2.1 Syntax

Consumers write ordinary interpolation. Handler authors write a type with the pattern:

```csharp
[InterpolatedStringHandler]
public ref struct MyHandler
{
    public MyHandler(int literalLength, int formattedCount /*, optional extra args, out bool shouldAppend */) { }
    public void AppendLiteral(string s) { }
    public void AppendFormatted<T>(T value) { }              // overloads with alignment/format optional
}
```

### 2.2 Semantics and rules

1. If the target parameter type is a handler (`[InterpolatedStringHandler]`), the compiler constructs it and calls the `Append*` methods in order instead of producing a string.
2. `[InterpolatedStringHandlerArgument("paramName")]` on the handler parameter passes other arguments (e.g. the logger, the `StringBuilder`, the `Span<char>`) into the handler constructor.
3. A trailing `out bool shouldAppend` constructor parameter lets the handler say "skip"; then **no hole expression is evaluated**.
4. `AppendLiteral` / `AppendFormatted` may return `bool`; returning `false` stops further appends.
5. When the target is `string`, the compiler uses `DefaultInterpolatedStringHandler` (if all holes allow it), otherwise falls back to `string.Format` / `string.Concat`.

## 3. Code examples

### 3.1 Minimal example

```csharp
Span<char> buffer = stackalloc char[64];
if (buffer.TryWrite($"HP {hp}/{maxHp}", out int written))
    label.Text = new string(buffer[..written]);
```

### 3.2 Extended examples

Appending to a `StringBuilder` without intermediate strings:

```csharp
var sb = new StringBuilder();
foreach (var unit in units)
    sb.Append($"{unit.Name,-12} {unit.Hp,4}/{unit.MaxHp}\n");   // StringBuilder.AppendInterpolatedStringHandler
```

Culture-explicit formatting (.NET 6):

```csharp
string s = string.Create(CultureInfo.InvariantCulture, $"{position.X:F2};{position.Y:F2}");
```

A custom handler that skips work when logging is disabled:

```csharp
[InterpolatedStringHandler]
public ref struct DebugLogHandler
{
    private DefaultInterpolatedStringHandler _inner;
    public readonly bool Enabled;

    public DebugLogHandler(int literalLength, int formattedCount, GameLog log, out bool shouldAppend)
    {
        Enabled = shouldAppend = log.IsDebugEnabled;
        _inner = shouldAppend ? new DefaultInterpolatedStringHandler(literalLength, formattedCount) : default;
    }

    public void AppendLiteral(string s) => _inner.AppendLiteral(s);
    public void AppendFormatted<T>(T value) => _inner.AppendFormatted(value);
    public string ToStringAndClear() => _inner.ToStringAndClear();
}

public sealed class GameLog
{
    public bool IsDebugEnabled { get; set; }

    public void Debug([InterpolatedStringHandlerArgument("")] ref DebugLogHandler message)
    {
        if (message.Enabled) Write(message.ToStringAndClear());
    }

    private static void Write(string s) => Console.WriteLine(s);
}

log.Debug($"Path for {unit} has {ComputeExpensiveStats()}");   // ComputeExpensiveStats() not called when disabled
```

(`""` in `InterpolatedStringHandlerArgument` refers to the receiver, `this`.)

## 4. Before this feature

```csharp
string a = string.Format("HP {0}/{1}", hp, maxHp);     // parses format, boxes ints
string b = "HP " + hp + "/" + maxHp;                   // ToString calls, Concat
if (log.IsDebugEnabled) log.Debug($"...");              // manual guard to avoid formatting
```

## 5. Under the hood

`string s = $"HP {hp}/{maxHp}";` is lowered to roughly:

```csharp
var h = new DefaultInterpolatedStringHandler(literalLength: 4, formattedCount: 2);
h.AppendLiteral("HP ");
h.AppendFormatted(hp);       // generic: no boxing, uses ISpanFormattable when available
h.AppendLiteral("/");
h.AppendFormatted(maxHp);
string s = h.ToStringAndClear();
```

`DefaultInterpolatedStringHandler` rents a buffer from `ArrayPool<char>.Shared`, formats directly into it
and returns it on `ToStringAndClear`. With a custom handler and `shouldAppend == false`, the lowering is

```csharp
var h = new DebugLogHandler(32, 2, log, out bool shouldAppend);
if (shouldAppend)
{
    h.AppendLiteral("Path for ");
    h.AppendFormatted(unit);
    ...
}
log.Debug(ref h);
```

so the hole expressions are inside the `if`.

## 6. Performance and allocation

- Ordinary `$"..."`: no boxing for value types implementing `ISpanFormattable`, no format parsing; only the final string allocates.
- `TryWrite` into a stack buffer or `StringBuilder.Append($"...")`: no intermediate string at all.
- Custom handlers with `shouldAppend`: zero work when disabled.

## 7. Pitfalls

- **Conditional evaluation.** With a handler that can skip, hole expressions with side effects (`$"{counter++}"`)
  may not run. Never put required side effects into interpolation holes.
- Changing a logging API from `string` to a handler parameter is source compatible but changes evaluation semantics for callers (see above).
- `ref struct` handlers cannot be used across `await` or captured in lambdas.

## 8. Concerns and limitations

- Writing a correct handler is advanced; most code should use the BCL ones (`StringBuilder`, `Span<char>.TryWrite`, `string.Create`, `Debug.Assert`).
- Microsoft.Extensions.Logging uses message templates (`"{Id}"`), not handlers. Do not use `$"..."` with `ILogger.LogX`;
  it formats eagerly and loses structured properties.
- Godot: `GD.Print` takes `params object[]`; pass a single interpolated string to avoid boxing every argument.

## 9. When to use: clean-code guidance

### 9.1 Use when
- Always prefer `$"..."` over `string.Format` and `+` chains.
- Formatting into existing buffers: `TryWrite`, `StringBuilder.Append($"...")`.
- Writing your own diagnostics API where disabled messages must cost nothing.

### 9.2 Avoid when
- Structured logging with `ILogger` (use templates or the `[LoggerMessage]` source generator).
- Culture-sensitive output where the current culture is wrong: use `string.Create(CultureInfo.InvariantCulture, $"...")`.

### 9.3 Good versus bad

```csharp
// Good
_logger.LogInformation("Unit {UnitId} spawned at {Position}", id, pos);
label.Text = $"Gold: {gold}";

// Bad
_logger.LogInformation($"Unit {id} spawned at {pos}");   // eager, unstructured
label.Text = string.Format("Gold: {0}", gold);           // boxing, runtime parse
```

## 10. Related features

- [constant-interpolated-strings.md](constant-interpolated-strings.md)
- [../11/newlines-in-interpolations.md](../11/newlines-in-interpolations.md)
- [../11/raw-string-literals.md](../11/raw-string-literals.md)

## 11. References

- [Tutorial: interpolated string handlers](https://learn.microsoft.com/en-us/dotnet/csharp/advanced-topics/performance/interpolated-string-handler)
- [Proposal: Improved interpolated strings](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-10.0/improved-interpolated-strings)
- [String interpolation in C# 10 and .NET 6 (.NET Blog)](https://devblogs.microsoft.com/dotnet/string-interpolation-in-c-10-and-net-6/)
