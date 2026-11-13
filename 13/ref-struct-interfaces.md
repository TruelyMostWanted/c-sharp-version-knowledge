---
id: cs13-ref-struct-interfaces
csharp: 13
tfm: net9.0
status: stable
tags: [ref-struct, interface, generics]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-13
related: allows-ref-struct.md
---
# `ref struct` types can implement interfaces (C# 13)

## Rules
- A `ref struct` can implement interfaces.
- It cannot be converted to the interface type (that would box and break ref safety).
- Explicit interface implementations are reachable only through a type parameter that `allows ref struct`.
- It must implement **all** interface members, including those with default implementations.

## Use when
- Combined with `allows ref struct` to write generic algorithms over span-like ref structs via interface constraints.

```csharp
interface IVisitor { void Visit(int value); }

ref struct SumVisitor : IVisitor
{
    public int Sum;
    public void Visit(int value) => Sum += value;
}

static void Walk<T>(ReadOnlySpan<int> data, ref T visitor) where T : IVisitor, allows ref struct
{
    foreach (var v in data) visitor.Visit(v);
}
```
