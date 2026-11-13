---
id: cs12-experimental-attribute
csharp: 12
tfm: net8.0
status: stable
tags: [attribute, experimental, api-design, library]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-12
---
# `[Experimental]` attribute (C# 12)

`System.Diagnostics.CodeAnalysis.ExperimentalAttribute` marks types, members or whole assemblies as experimental.
The compiler reports a diagnostic (with the given ID) wherever they are used; callers must suppress it consciously.

```csharp
[Experimental("TMW0001")]
public sealed class NewPathfinder { ... }

#pragma warning disable TMW0001
var pf = new NewPathfinder();
#pragma warning restore TMW0001
```

## Rules
- If the assembly is marked, all its types are experimental.
- Use a stable, project-specific diagnostic ID prefix.
