---
id: cs10-asyncmethodbuilder-on-methods
csharp: 10
tfm: net6.0
status: stable
tags: [async, performance, ValueTask, pooling]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/attributes/general#asyncmethodbuilder-attribute
---
# `[AsyncMethodBuilder]` on individual methods (C# 10)

> **Abstract.** Since C# 10 the `[AsyncMethodBuilder]` attribute can be placed on a single `async` method
> instead of only on the returned task-like type. The method then uses a different builder for its state machine,
> typically `PoolingAsyncValueTaskMethodBuilder`, which reuses state-machine boxes instead of allocating one per
> asynchronous completion. It is a measured optimization for hot async paths, not a style choice.

| | |
|---|---|
| Introduced | C# 10 · .NET 6 (`net6.0`) |
| Status | Stable |
| Kind | Compiler behaviour |
| Runtime support needed | The pooling builders ship with .NET 6 (`System.Runtime.CompilerServices`) |
| Audience | Library authors, networking and I/O hot paths |
| Modernize category | n/a (only after profiling) |

## 1. Motivation

Every `async` method is compiled into a state machine. When the method completes **asynchronously**,
the state machine is moved to the heap ("boxed") so it can survive until the continuation runs.
For a method called millions of times (reading packets, parsing frames) these boxes are a measurable
source of garbage. Before C# 10, the only way to change the builder was to put `[AsyncMethodBuilder]` on a
**custom task-like type**, which forced every method returning that type to use the same builder.
C# 10 lets you opt in per method while still returning plain `ValueTask` / `ValueTask<T>`.

## 2. The feature explained

### 2.1 Syntax

```csharp
[AsyncMethodBuilder(typeof(PoolingAsyncValueTaskMethodBuilder<>))]   // for ValueTask<T>
public async ValueTask<int> ReadAsync() { ... }

[AsyncMethodBuilder(typeof(PoolingAsyncValueTaskMethodBuilder))]     // for ValueTask
public async ValueTask FlushAsync() { ... }
```

### 2.2 Semantics and rules

1. The attribute is applied to an `async` method, local function or lambda; it overrides the builder of the return type.
2. The builder type must follow the async builder pattern (`Create`, `Start`, `SetResult`, `SetException`, `Task`, `AwaitOnCompleted`, ...).
3. For generic return types the attribute names the **open** generic builder (`<>`); the compiler closes it.
4. The method's signature and callers are unchanged; only the generated state machine differs.
5. If the method completes synchronously, no box is allocated with any builder; pooling only matters for asynchronous completions.

## 3. Code examples

### 3.1 Minimal example

```csharp
using System.Runtime.CompilerServices;

public sealed class PacketReader
{
    private readonly Stream _stream;
    private readonly byte[] _header = new byte[4];

    public PacketReader(Stream stream) => _stream = stream;

    [AsyncMethodBuilder(typeof(PoolingAsyncValueTaskMethodBuilder<>))]
    public async ValueTask<int> ReadLengthAsync(CancellationToken ct)
    {
        int read = 0;
        while (read < _header.Length)
            read += await _stream.ReadAsync(_header.AsMemory(read), ct);   // (end-of-stream check omitted)
        return BitConverter.ToInt32(_header);
    }
}
```

### 3.2 Extended examples

Pooling a tight server loop (one call per message):

```csharp
public sealed class Connection
{
    private readonly PipeReader _reader;

    public Connection(PipeReader reader) => _reader = reader;

    [AsyncMethodBuilder(typeof(PoolingAsyncValueTaskMethodBuilder<>))]
    public async ValueTask<Message?> ReceiveAsync(CancellationToken ct)
    {
        while (true)
        {
            ReadResult result = await _reader.ReadAsync(ct);
            ReadOnlySequence<byte> buffer = result.Buffer;

            if (MessageParser.TryParse(ref buffer, out Message? message))
            {
                _reader.AdvanceTo(buffer.Start);
                return message;
            }

            _reader.AdvanceTo(buffer.Start, buffer.End);
            if (result.IsCompleted) return null;
        }
    }
}
```

Non-generic variant:

```csharp
[AsyncMethodBuilder(typeof(PoolingAsyncValueTaskMethodBuilder))]
public async ValueTask SendAsync(ReadOnlyMemory<byte> payload, CancellationToken ct)
{
    await _stream.WriteAsync(payload, ct);
    await _stream.FlushAsync(ct);
}
```

## 4. Before this feature

```csharp
// Option A: accept the allocation per asynchronous completion
public async ValueTask<int> ReadLengthAsync(CancellationToken ct) { ... }

// Option B (complex): implement IValueTaskSource<T> by hand with a pooled, resettable source object,
// ManualResetValueTaskSourceCore<T>, version tokens and manual continuation handling.
```

Option B was correct but error-prone; the attribute gives most of the benefit with one line.

## 5. Under the hood

The compiler-generated state machine's builder field changes type:

```csharp
// default
private struct <ReadLengthAsync>d__1 : IAsyncStateMachine
{
    public AsyncValueTaskMethodBuilder<int> <>t__builder;
    ...
}

// with the attribute
private struct <ReadLengthAsync>d__1 : IAsyncStateMachine
{
    public PoolingAsyncValueTaskMethodBuilder<int> <>t__builder;
    ...
}

// kick-off method (both cases)
var sm = new <ReadLengthAsync>d__1 { <>4__this = this, ct = ct, <>1__state = -1 };
sm.<>t__builder = PoolingAsyncValueTaskMethodBuilder<int>.Create();
sm.<>t__builder.Start(ref sm);
return sm.<>t__builder.Task;
```

On the first asynchronous `await`, the pooling builder rents a box from a per-thread / per-core cache
instead of allocating it, and returns it when the `ValueTask` has been consumed.

## 6. Performance and allocation

- Saves one heap allocation per **asynchronous** completion of the method.
- Costs: pooled boxes stay alive (memory held), plus a small bookkeeping overhead per call.
  For rarely called methods this can be a net loss.
- Measure with BenchmarkDotNet (`[MemoryDiagnoser]`) or dotnet-counters (`gc-heap-alloc-rate`) before and after.

## 7. Pitfalls

- **`ValueTask` rules become critical.** A pooled `ValueTask` must be awaited **exactly once** and never after
  completion was observed. Awaiting twice, calling `.Result` before completion or storing it and awaiting later
  can read another call's result, because the box was returned to the pool and reused.
- Using `.AsTask()` once is fine; caching the `ValueTask` itself is not.
- Wrong builder for the return type (`PoolingAsyncValueTaskMethodBuilder` on `ValueTask<T>`) is a compile error,
  not a silent fallback.

## 8. Concerns and limitations

- Readability: an attribute on a method that changes memory behaviour is easy to overlook in review. Comment why.
- Only available for task-like return types that have a compatible builder; `Task` / `Task<T>` have no pooling builder in the BCL.
- Godot: game loop code rarely uses `ValueTask` hot paths; `await ToSignal(...)` returns a `SignalAwaiter`, not a `ValueTask`, so this attribute does not apply there.

## 9. When to use: clean-code guidance

### 9.1 Use when
- Profiling shows allocations from a specific `async ValueTask` method that is called very frequently and usually completes asynchronously (network, pipelines, channels).

### 9.2 Avoid when
- General application code, UI handlers, one-shot initialization.
- Callers might await the result more than once (public APIs consumed by unknown code).

### 9.3 Good versus bad

```csharp
// Good: hot, internal, consumed exactly once
[AsyncMethodBuilder(typeof(PoolingAsyncValueTaskMethodBuilder<>))]
internal async ValueTask<Frame> ReadFrameAsync(CancellationToken ct) { ... }

// Bad: "might be faster" on a public method whose ValueTask callers store and await twice
[AsyncMethodBuilder(typeof(PoolingAsyncValueTaskMethodBuilder<>))]
public async ValueTask<Settings> LoadSettingsAsync() { ... }
```

## 10. Related features

- [interpolated-string-handlers.md](interpolated-string-handlers.md): another builder-pattern feature for performance.
- [../13/ref-and-unsafe-in-async-iterators.md](../13/ref-and-unsafe-in-async-iterators.md): spans inside async methods.

## 11. References

- [AsyncMethodBuilder attribute](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/attributes/general#asyncmethodbuilder-attribute)
- [Proposal: AsyncMethodBuilder override](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-10.0/async-method-builders)
- [Understanding the whys, whats, and whens of ValueTask (.NET Blog)](https://devblogs.microsoft.com/dotnet/understanding-the-whys-whats-and-whens-of-valuetask/)
