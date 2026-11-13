---
id: cs13-escape-sequence-e
csharp: 13
tfm: net9.0
status: stable
tags: [escape, ansi, console, string]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-13
---
# `\e` escape sequence (C# 13)

`\e` is the ESCAPE character (U+001B). Replaces `\u001b` or `\x1b`.

```csharp
Console.WriteLine("\e[31mError\e[0m");   // red text in ANSI terminals
```

Avoid `\x1b`: following characters that are valid hex digits become part of the escape.
