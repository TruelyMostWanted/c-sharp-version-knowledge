---
id: cs11-span-pattern-constant-string
csharp: 11
tfm: net7.0
status: stable
tags: [span, pattern-matching, string, parsing]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-11
modernizes: span.SequenceEqual("text") or allocating ToString() before comparing
---
# Pattern match `Span<char>` / `ReadOnlySpan<char>` on a constant string (C# 11)

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
No allocation for the comparison; useful in config, CSV and command parsers.
