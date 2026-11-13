---
id: cs13-allows-ref-struct
csharp: 13
tfm: net9.0
status: stable
tags: [generics, ref-struct, span, anti-constraint]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-13
---
# `allows ref struct` anti-constraint (C# 13)

Before C# 13, ref structs (`Span<T>`, ...) could not be generic type arguments.
`where T : allows ref struct` lets a type parameter accept them; the compiler then enforces ref safety on `T`.

```csharp
public class C<T> where T : allows ref struct
{
    public void M(scoped T p)
    {
        // p must follow ref safety rules: no boxing, no storing in fields, no capture in lambdas
    }
}
```

## Use when
- Generic utilities and interfaces that should also work with spans or other ref structs.
