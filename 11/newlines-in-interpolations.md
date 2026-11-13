---
id: cs11-newlines-in-interpolations
csharp: 11
tfm: net7.0
status: stable
tags: [string-interpolation, formatting]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-11
---
# Newlines in string interpolation expressions (C# 11)

The expression inside `{ }` of an interpolated string may span several lines, also in non-verbatim strings.

```csharp
string label = $"State: {state switch
{
    PlayerState.Idle    => "idle",
    PlayerState.Running => "running",
    _                   => "other",
}}";
```

Use sparingly; a separate local variable is often clearer.
