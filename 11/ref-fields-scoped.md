---
id: cs11-ref-fields-scoped
csharp: 11
tfm: net7.0
status: stable
needs-runtime: true (.NET 7)
tags: [ref-field, scoped, ref-struct, span, low-level]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/statements/declarations#scoped-ref
---
# `ref` fields and `scoped` (C# 11)

A `ref struct` can declare `ref` fields (how `Span<T>` is built). `scoped` limits a ref or ref struct parameter
or local so it cannot escape the current method.

```csharp
public ref struct Cursor
{
    private ref int _value;
    public Cursor(ref int value) { _value = ref value; }
    public void Increment() => _value++;
}

static int Parse(scoped ReadOnlySpan<char> text) { /* text may not be stored or returned */ return 0; }
```

## Use when
- Low-level, allocation-free helpers (parsers, cursors, writers over spans).
- Application code rarely needs this; prefer `Span<T>` and existing APIs.
