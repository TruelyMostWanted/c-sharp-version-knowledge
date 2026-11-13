---
id: cs12-ref-readonly-parameters
csharp: 12
tfm: net8.0
status: stable
tags: [ref, readonly, in, struct, performance, api-design]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-12
---
# `ref readonly` parameters (C# 12)

Passes a read-only reference; extends `in`. For APIs that need a **variable** (not a temporary) but do not modify it.

```csharp
public static float Determinant(ref readonly Matrix4x4 m) => m.GetDeterminant();

var mat = Matrix4x4.Identity;
float d = Determinant(ref mat);   // or `in mat`
```

## Rules
- Changing an existing `ref` parameter to `ref readonly` is **not** a breaking change for callers (changing to `in` is).
- Prefer `in` for new APIs that may also accept temporaries; use `ref readonly` when a variable is required
  or when migrating existing `ref` APIs.
