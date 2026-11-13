---
id: cs13-lock-type
csharp: 13
tfm: net9.0
status: stable
needs-runtime: true (System.Threading.Lock, .NET 9)
tags: [lock, threading, synchronization]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-13
---
# New `System.Threading.Lock` type (C# 13 / .NET 9)

If the target of a `lock` statement is a `System.Threading.Lock`, the compiler uses
`Lock.EnterScope()` (a `ref struct` disposed at the end) instead of `Monitor`.

## Use when
- Any new `lock`: declare a `Lock` instead of `object`. Only the field type changes.

```csharp
private readonly Lock _gate = new();

public void Add(Item item)
{
    lock (_gate)
    {
        _items.Add(item);
    }
}
```

## Pitfalls
- If the `Lock` is converted to another type (e.g. `object`), the compiler falls back to `Monitor`-based code. Keep the static type `Lock`.
- Godot: engine APIs are mostly main-thread only; locks protect your own data shared with worker threads (`WorkerThreadPool`, `Task.Run`).
