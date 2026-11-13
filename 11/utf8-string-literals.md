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

The `u8` suffix produces the UTF-8 bytes of a constant string as `ReadOnlySpan<byte>`, without runtime encoding or allocation.

```csharp
ReadOnlySpan<byte> header = "GDSAVE01"u8;
stream.Write(header);

byte[] copy = "hello"u8.ToArray();   // only if an array is really needed
```

## Rules
- Not a compile-time `const`; cannot be used for default parameter values.
- Cannot be combined with string interpolation.
- Type is `ReadOnlySpan<byte>`; cannot be stored in fields of non-ref types (use `.ToArray()` or a static property returning the span).
