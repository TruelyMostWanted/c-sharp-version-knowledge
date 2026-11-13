---
id: cs13-overload-resolution-priority
csharp: 13
tfm: net9.0
status: stable
needs-runtime: true (OverloadResolutionPriorityAttribute, .NET 9)
tags: [overload, library, attribute, api-design]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-13
---
# Overload resolution priority (C# 13)

`[OverloadResolutionPriority(n)]` (`System.Runtime.CompilerServices`) makes the compiler prefer overloads
with a higher priority.

## Use when
- **Library authors** adding a better (e.g. span-based, faster) overload without breaking or
  making existing calls ambiguous.

```csharp
public static class Log
{
    public static void Write(params string[] parts) { ... }

    [OverloadResolutionPriority(1)]
    public static void Write(params ReadOnlySpan<string> parts) { ... }
}
```

## Pitfalls
- Use sparingly; hidden priority rules confuse readers. Not for application code.
