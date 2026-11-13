---
id: cs10-constant-interpolated-strings
csharp: 10
tfm: net6.0
status: stable
tags: [const, string-interpolation]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-10
---
# Constant interpolated strings (C# 10)

A `const string` can be initialized with interpolation if **all** placeholders are constant strings.

```csharp
const string SaveDir  = "user://saves";
const string SaveFile = $"{SaveDir}/slot.json";
```

## Rules
- Placeholders must be `const string`; numeric constants are not allowed (their formatting depends on culture).
