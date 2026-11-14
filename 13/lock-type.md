---
id: cs13-lock-type
csharp: 13
tfm: net9.0
status: stable
needs-runtime: true (System.Threading.Lock, .NET 9)
tags: [lock, threading, synchronization]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/statements/lock
modernizes: private readonly object _lock = new();
---
# New `System.Threading.Lock` type (C# 13 / .NET 9)

> **Abstract.** .NET 9 introduces `System.Threading.Lock`, a dedicated mutual-exclusion type. When the expression of a
> `lock` statement has static type `Lock`, the C# 13 compiler emits `Lock.EnterScope()` with a `using`-like dispose
> instead of `Monitor.Enter`/`Exit`. The only code change is the field type: `object` → `Lock`.

| | |
|---|---|
| Introduced | C# 13 · .NET 9 (`net9.0`) |
| Status | Stable |
| Kind | Compiler behaviour + new BCL type |
| Runtime support needed | Yes: .NET 9 (`System.Threading.Lock`) |
| Audience | Everyday code with threads |
| Modernize category | AUTO if the object is only used in `lock` statements |

## 1. Motivation

`lock (obj)` has always used `Monitor` on an arbitrary object. Every object carries a sync-block slot, `Monitor` must
handle "thin" and "fat" locks, and nothing prevents locking on `this`, a `Type` or a string. A dedicated type is
clearer, type-safe (you cannot accidentally lock on something else) and faster.

## 2. The feature explained

### 2.1 Syntax

```csharp
private readonly Lock _gate = new();

lock (_gate) { ... }
```

### 2.2 Semantics and rules

1. If the `lock` expression's static type is `System.Threading.Lock`, the compiler uses `Lock.EnterScope()`.
2. Otherwise (any other type, including `object` holding a `Lock`), the old `Monitor` pattern is used.
3. Converting a `Lock` to `object` or another type produces warning **CS9216**, because such a lock would use `Monitor` and not interoperate with `EnterScope`.
4. `Lock` also has explicit APIs: `Enter()`, `TryEnter(timeout)`, `Exit()`, `IsHeldByCurrentThread`.
5. Same semantics as before: reentrant (the same thread may enter again), no `await` inside a `lock` body.

## 3. Code examples

### 3.1 Minimal example

```csharp
private readonly Lock _gate = new();
private readonly List<Item> _items = [];

public void Add(Item item)
{
    lock (_gate)
    {
        _items.Add(item);
    }
}
```

### 3.2 Extended examples

Try-enter with timeout:

```csharp
if (_gate.TryEnter(TimeSpan.FromMilliseconds(5)))
{
    try { Flush(); }
    finally { _gate.Exit(); }
}
```

Explicit scope (same as `lock`):

```csharp
using (_gate.EnterScope())
{
    _counter++;
}
```

Worker threads sharing results with the main thread (game):

```csharp
public sealed partial class ChunkGenerator : Node
{
    private readonly Lock _readyLock = new();
    private readonly Queue<ChunkData> _ready = new();

    private void GenerateOnWorker(Vector2I coord)          // runs via Task.Run / WorkerThreadPool
    {
        ChunkData data = Build(coord);
        lock (_readyLock) _ready.Enqueue(data);
    }

    public override void _Process(double delta)           // main thread
    {
        lock (_readyLock)
        {
            while (_ready.TryDequeue(out var chunk)) Spawn(chunk);
        }
    }
}
```

## 4. Before this feature

```csharp
private readonly object _gate = new();

lock (_gate) { _items.Add(item); }
```

## 5. Under the hood

With `Lock`:

```csharp
Lock.Scope scope = _gate.EnterScope();   // ref struct
try
{
    _items.Add(item);
}
finally
{
    scope.Dispose();                     // exits the lock
}
```

With `object` (classic):

```csharp
object o = _gate;
bool lockTaken = false;
try
{
    Monitor.Enter(o, ref lockTaken);
    _items.Add(item);
}
finally
{
    if (lockTaken) Monitor.Exit(o);
}
```

## 6. Performance and allocation

- `Lock` avoids sync-block lookups and is faster than `Monitor` in uncontended and moderately contended cases.
- `Lock.Scope` is a ref struct: no allocation.
- Allocation per lock object is similar to `new object()`.

## 7. Pitfalls

- If the `Lock` is converted to another type (e.g. `object`, generic `T`), the compiler falls back to `Monitor`-based code (CS9216 warns). Keep the static type `Lock`.
- Mixing `lock (_gate)` with `Monitor.Enter(_gate)` on the same `Lock` instance does not synchronize. Use one mechanism.
- Changing a **public or protected** lock object's type is an API change.

## 8. Concerns and limitations

- Requires .NET 9; on older runtimes the type does not exist.
- Not async-aware; for async code use `SemaphoreSlim.WaitAsync`.
- Godot: engine APIs are mostly main-thread only; locks protect your own data shared with worker threads (`WorkerThreadPool`, `Task.Run`). Never call engine APIs inside a lock held by a worker.

## 9. When to use: clean-code guidance

### 9.1 Use when
- Any new `lock`: declare a `Lock` instead of `object`.

### 9.2 Avoid when
- The lock object is exposed or passed as `object`.
- You need async waiting (`SemaphoreSlim`) or reader/writer semantics (`ReaderWriterLockSlim`).

### 9.3 Good versus bad

```csharp
// Good
private readonly Lock _sync = new();

// Bad: locking on public or shared objects
lock (this) { }
lock (typeof(Cache)) { }
```

## 10. Related features

- [ref-and-unsafe-in-async-iterators.md](ref-and-unsafe-in-async-iterators.md)

## 11. References

- [lock statement](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/statements/lock)
- [System.Threading.Lock](https://learn.microsoft.com/en-us/dotnet/api/system.threading.lock)
- [Proposal: Lock object](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-13.0/lock-object)
