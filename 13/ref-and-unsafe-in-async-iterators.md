---
id: cs13-ref-and-unsafe-in-async-iterators
csharp: 13
tfm: net9.0
status: stable
tags: [async, iterator, span, ref-struct, unsafe]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-13
---
# `ref` locals, ref structs and `unsafe` in async methods and iterators (C# 13)

## Rules
- `async` methods may declare `ref` locals and `ref struct` locals (e.g. `Span<T>`), but they cannot be accessed **across an `await`**.
- Iterators may declare them too, but not access them across a `yield return`.
- Iterators may contain `unsafe` contexts, but every `yield return` / `yield break` must be in a safe context.

## Use when
- Parsing or processing with `Span<T>` inside an `async` method between awaits.

```csharp
public async Task<int> CountLinesAsync(Stream s)
{
    byte[] data = await ReadAllAsync(s);
    ReadOnlySpan<byte> span = data;          // fine: no await while span is alive
    return span.Count((byte)'\n');
}
```
