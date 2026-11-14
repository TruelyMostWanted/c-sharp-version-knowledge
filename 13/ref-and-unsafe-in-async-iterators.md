---
id: cs13-ref-and-unsafe-in-async-iterators
csharp: 13
tfm: net9.0
status: stable
tags: [async, iterator, span, ref-struct, unsafe]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-13#ref-and-unsafe-in-iterators-and-async-methods
modernizes: copying spans to arrays only because the method is async or an iterator
---
# `ref` locals, ref structs and `unsafe` in async methods and iterators (C# 13)

> **Abstract.** `async` methods and iterators may now declare `ref` locals and locals of `ref struct` types
> (`Span<T>`, `ReadOnlySpan<T>`, `Utf8JsonReader`, ...), as long as those locals are **not alive across an
> `await` or `yield`**. Iterators may also contain `unsafe` code. Span-based parsing inside async methods no longer
> needs helper methods or array copies.

| | |
|---|---|
| Introduced | C# 13 · .NET 9 (`net9.0`) |
| Status | Stable |
| Kind | Semantics (ref safety) |
| Runtime support needed | No |
| Audience | Everyday code doing I/O and parsing |
| Modernize category | CASE |

## 1. Motivation

Async methods and iterators are rewritten into state machines whose locals become fields. Ref structs cannot be fields
of a class, so C# banned them entirely in such methods, even when a span was used only between two awaits.
Developers moved span code into separate synchronous helper methods or converted to arrays.

## 2. The feature explained

### 2.1 Syntax

No new syntax; previously forbidden declarations are now allowed.

### 2.2 Semantics and rules

1. In `async` methods: `ref` locals and ref-struct locals are allowed, but must not be **used after an `await`** that follows their declaration (they cannot be hoisted).
2. In iterators: the same rule applies to `yield return`.
3. Iterators may contain `unsafe` blocks, but each `yield return` / `yield break` must be in a **safe** context.
4. `using` declarations of ref struct disposables follow the same rule: the resource must not be alive across a suspension point.

## 3. Code examples

### 3.1 Minimal example

```csharp
public async Task<int> CountLinesAsync(Stream s)
{
    byte[] data = await ReadAllAsync(s);
    ReadOnlySpan<byte> span = data;          // fine: no await while span is alive
    return span.Count((byte)'\n');
}
```

### 3.2 Extended examples

Parsing JSON with `Utf8JsonReader` after an async read:

```csharp
public async Task<int> ReadVersionAsync(string path)
{
    byte[] bytes = await File.ReadAllBytesAsync(path);

    var reader = new Utf8JsonReader(bytes);     // ref struct local, used only before the next await
    while (reader.Read())
        if (reader.TokenType == JsonTokenType.PropertyName && reader.ValueTextEquals("version"u8))
        {
            reader.Read();
            return reader.GetInt32();
        }
    return 0;
}
```

Iterator using spans between yields:

```csharp
public IEnumerable<int> ParseNumbers(string csv)
{
    int start = 0;
    while (start < csv.Length)
    {
        int end = csv.IndexOf(',', start);
        if (end < 0) end = csv.Length;

        int value;
        {
            ReadOnlySpan<char> token = csv.AsSpan(start, end - start).Trim();   // scoped to this block
            value = int.Parse(token);
        }

        yield return value;    // span is not alive here
        start = end + 1;
    }
}
```

Error example (still not allowed):

```csharp
async Task Bad(Stream s)
{
    Span<byte> buf = stackalloc byte[16];
    await s.FlushAsync();
    buf[0] = 1;              // error: span used across await
}
```

## 4. Before this feature

```csharp
public async Task<int> ReadVersionAsync(string path)
{
    byte[] bytes = await File.ReadAllBytesAsync(path);
    return ParseVersion(bytes);                  // separate sync helper only to use Utf8JsonReader
}

private static int ParseVersion(byte[] bytes) { var reader = new Utf8JsonReader(bytes); ... }
```

## 5. Under the hood

The compiler's hoisting analysis decides which locals become state-machine fields. Ref and ref-struct locals that are
not live across a suspension point stay as real locals of `MoveNext()`; if one would need hoisting, the compiler reports an error.

## 6. Performance and allocation

Enables allocation-free span code inside async methods and iterators (no `ToArray()` copies).

## 7. Pitfalls

- Refactoring that adds an `await` between a span's declaration and its use causes compile errors; restructure with blocks or helper methods.
- `stackalloc` inside a loop in an iterator still grows the stack per iteration; prefer pooled buffers.

## 8. Concerns and limitations

- None for compatibility; previously invalid code becomes valid.
- Godot: `async` methods awaiting `ToSignal(...)` can now parse span data between awaits without helper methods.

## 9. When to use: clean-code guidance

### 9.1 Use when
- Parsing or processing data with spans inside async methods or iterators, between suspension points.

### 9.2 Avoid when
- The span logic is substantial: a separate synchronous method is still easier to test.

### 9.3 Good versus bad

```csharp
// Good: short span section between awaits
var data = await LoadAsync();
int checksum = Crc32.HashToUInt32(data.AsSpan());
await SaveAsync(checksum);

// Bad: long-lived span across awaits (does not compile) "fixed" with ToArray copies everywhere
```

## 10. Related features

- [allows-ref-struct.md](allows-ref-struct.md)
- [lock-type.md](lock-type.md)
- [../11/ref-fields-scoped.md](../11/ref-fields-scoped.md)

## 11. References

- [What's new in C# 13: ref and unsafe in iterators and async methods](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-13#ref-and-unsafe-in-iterators-and-async-methods)
- [Proposal: ref and unsafe in iterators and async](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-13.0/ref-unsafe-in-iterators-async)
