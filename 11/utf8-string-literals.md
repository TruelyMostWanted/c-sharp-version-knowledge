---
id: cs11-utf8-string-literals
csharp: 11
tfm: net7.0
status: stable
tags: [utf8, string, span, bytes, networking]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/reference-types#utf-8-string-literals
modernizes: Encoding.UTF8.GetBytes("constant")
---
# UTF-8 string literals (C# 11)

> **Abstract.** The `u8` suffix turns a string literal into its UTF-8 bytes at **compile time**:
> `"GDSAVE01"u8` has type `ReadOnlySpan<byte>` and points to data embedded in the assembly. No encoding at runtime,
> no allocation. Ideal for protocol headers, magic numbers, HTTP and JSON tokens.

| | |
|---|---|
| Introduced | C# 11 · .NET 7 (`net7.0`) |
| Status | Stable |
| Kind | Syntax + compiler behaviour |
| Runtime support needed | No (uses `ReadOnlySpan<byte>`) |
| Audience | I/O, networking, serialization code |
| Modernize category | PREFER |

## 1. Motivation

Modern .NET I/O works in UTF-8 bytes (`Utf8JsonWriter`, `PipeWriter`, sockets). Constant byte sequences were written
as `Encoding.UTF8.GetBytes("text")` (runtime encoding plus an array allocation) or as unreadable `new byte[] { 0x47, 0x44, ... }`.

## 2. The feature explained

### 2.1 Syntax

```csharp
ReadOnlySpan<byte> a = "text"u8;
ReadOnlySpan<byte> b = @"C:\path"u8;
ReadOnlySpan<byte> c = """{"k":1}"""u8;
ReadOnlySpan<byte> d = "a"u8 + "b"u8;       // compile-time concatenation of u8 literals
```

### 2.2 Semantics and rules

1. The literal must be a constant string; interpolation (`$"..."u8`) is not allowed.
2. The type is `ReadOnlySpan<byte>`; there is no trailing null terminator in the span (the stored data has one, but `Length` excludes it).
3. It is **not** a compile-time constant: cannot be used for `const`, default parameter values or attribute arguments.
4. `+` between two `u8` literals is folded at compile time.
5. Invalid UTF-16 (lone surrogates) in the literal is an error.

## 3. Code examples

### 3.1 Minimal example

```csharp
ReadOnlySpan<byte> header = "GDSAVE01"u8;
stream.Write(header);
```

### 3.2 Extended examples

Static property returning the span (allocation-free, reusable):

```csharp
public static class Protocol
{
    public static ReadOnlySpan<byte> Magic => "TMW\u0001"u8;
    public static ReadOnlySpan<byte> Ping  => "PING\r\n"u8;
}

if (packet.StartsWith(Protocol.Magic)) Handle(packet[Protocol.Magic.Length..]);
```

`Utf8JsonWriter` with pre-encoded property names:

```csharp
writer.WriteStartObject();
writer.WriteString("name"u8, unit.Name);
writer.WriteNumber("hp"u8, unit.Hp);
writer.WriteEndObject();
```

Matching UTF-8 tokens while reading JSON:

```csharp
while (reader.Read())
{
    if (reader.TokenType == JsonTokenType.PropertyName && reader.ValueTextEquals("hp"u8))
    {
        reader.Read();
        hp = reader.GetInt32();
    }
}
```

When a `byte[]` is really needed:

```csharp
private static readonly byte[] s_newline = "\n"u8.ToArray();   // one allocation, cached
```

## 4. Before this feature

```csharp
private static readonly byte[] s_header = Encoding.UTF8.GetBytes("GDSAVE01");
private static ReadOnlySpan<byte> Header2 => new byte[] { 0x47, 0x44, 0x53, 0x41, 0x56, 0x45, 0x30, 0x31 };
```

(The second form was already allocation-free thanks to a compiler optimization, but unreadable.)

## 5. Under the hood

The compiler encodes the string to UTF-8 and stores the bytes (plus a terminating `\0`) in the assembly's
`<PrivateImplementationDetails>` data section. The expression becomes:

```
ldsflda  valuetype '<PrivateImplementationDetails>'/'__StaticArrayInitTypeSize=9' '<PrivateImplementationDetails>'::'<hash>'
ldc.i4.8
newobj   instance void valuetype System.ReadOnlySpan`1<uint8>::.ctor(void*, int32)
```

That is, a span over static read-only memory. No runtime encoding, no heap allocation.

## 6. Performance and allocation

- Zero allocation, zero encoding work.
- `ToArray()` allocates a new array **every** time; cache the result if you need an array.

## 7. Pitfalls

- Cannot be stored in a field of a class or normal struct (span type). Use a static property `=> "..."u8`, or a cached `byte[]`.
- Not usable in `async` methods across `await` (span), except as a short-lived local (C# 13 relaxations).
- `"é"u8.Length` is 2: UTF-8 lengths differ from `string.Length`.
- Escape `\x01` in a regular string literal is variable-length hex; prefer `\u0001` to avoid surprises.

## 8. Concerns and limitations

- No interpolation; build dynamic UTF-8 with `Utf8.TryWrite` (.NET 8) or `Utf8Formatter`.
- Godot: `FileAccess.StoreBuffer` takes `byte[]`; convert with `ToArray()` once and cache, or use .NET `FileStream` for span-based I/O.

## 9. When to use: clean-code guidance

### 9.1 Use when
- Constant bytes for protocols, file headers, JSON property names, HTTP tokens.

### 9.2 Avoid when
- The API needs `string` or `byte[]` anyway and is not performance-sensitive.

### 9.3 Good versus bad

```csharp
// Good
output.Write("OK\r\n"u8);

// Bad: encodes and allocates on every call
output.Write(Encoding.UTF8.GetBytes("OK\r\n"));
```

## 10. Related features

- [raw-string-literals.md](raw-string-literals.md): `"""..."""u8`.
- [span-pattern-constant-string.md](span-pattern-constant-string.md)
- [../14/implicit-span-conversions.md](../14/implicit-span-conversions.md)

## 11. References

- [UTF-8 string literals](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/reference-types#utf-8-string-literals)
- [Proposal: UTF-8 string literals](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-11.0/utf8-string-literals)
