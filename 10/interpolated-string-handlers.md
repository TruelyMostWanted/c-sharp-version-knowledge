---
id: cs10-interpolated-string-handlers
csharp: 10
tfm: net6.0
status: stable
tags: [string-interpolation, performance, logging, allocation]
source: https://learn.microsoft.com/en-us/dotnet/csharp/advanced-topics/performance/interpolated-string-handler
---
# Interpolated string handlers (C# 10)

A type marked `[InterpolatedStringHandler]` can receive an interpolated string as a sequence of
`AppendLiteral` / `AppendFormatted` calls instead of a finished `string`. This lets APIs skip formatting entirely
(e.g. when a log level is disabled) or write into a buffer without allocations.

## What it means in practice
- Since .NET 6, `$"..."` in normal code is compiled via `DefaultInterpolatedStringHandler` and is faster
  than `string.Format`. Prefer interpolation over `string.Format` and `+` concatenation.
- APIs such as `StringBuilder.Append($"...")`, `Debug.Assert(cond, $"...")` and `span.TryWrite($"...")` take handlers.

```csharp
Span<char> buffer = stackalloc char[64];
if (buffer.TryWrite($"HP {hp}/{maxHp}", out int written))
    label.Text = new string(buffer[..written]);
```

## Use when
- Writing your own logging or diagnostics API: add a custom handler with a `bool` "enabled" out parameter so
  disabled messages are never formatted. Most application code only benefits implicitly.
