---
id: cs10-record-sealed-tostring
csharp: 10
tfm: net6.0
status: stable
tags: [record, ToString, sealed]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-10
---
# `sealed` `ToString` override in records (C# 10)

A record can declare `public sealed override string ToString()`. Derived records then keep this implementation
instead of the compiler generating a new one for each derived type.

```csharp
public abstract record GameEvent(int Round)
{
    public sealed override string ToString() => $"[{Round}] {GetType().Name}";
}
```
