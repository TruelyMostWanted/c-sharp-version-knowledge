---
id: cs10-asyncmethodbuilder-on-methods
csharp: 10
tfm: net6.0
status: stable
tags: [async, performance, ValueTask, pooling]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/attributes/general#asyncmethodbuilder-attribute
---
# `[AsyncMethodBuilder]` on individual methods (C# 10)

The builder for an async method can be overridden per method, e.g. to pool the state machine of a hot `ValueTask` method.

```csharp
[AsyncMethodBuilder(typeof(PoolingAsyncValueTaskMethodBuilder<>))]
public async ValueTask<int> ReadPacketAsync() { ... }
```

## Use when
- Only after profiling shows allocations from a frequently called async method. Not for general application code.
