---
id: cs11-ref-fields-scoped
csharp: 11
tfm: net7.0
status: stable
needs-runtime: true (.NET 7, RuntimeFeature.ByRefFields)
tags: [ref-field, scoped, ref-struct, span, low-level]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/statements/declarations#scoped-ref
---
# `ref` fields and `scoped` (C# 11)

> **Abstract.** A `ref struct` may declare **`ref` fields**: fields that hold a managed reference to a variable,
> which is exactly how `Span<T>` is implemented. The `scoped` modifier restricts a `ref` or `ref struct`
> parameter or local so that it cannot escape the current method, which lets the compiler accept more safe code.
> Both are tools for allocation-free, low-level helpers; application code rarely needs them directly.

| | |
|---|---|
| Introduced | C# 11 · .NET 7 (`net7.0`) |
| Status | Stable |
| Kind | Language (ref safety) + runtime |
| Runtime support needed | Yes: .NET 7 for `ref` fields |
| Audience | Low-level and library code |
| Modernize category | n/a |

## 1. Motivation

`Span<T>` was implemented in the runtime with an internal "ByReference<T>" trick because C# had no way to store a
`ref` in a field. Library authors could not build their own span-like types (cursors, writers, ref-holding wrappers).
At the same time the ref-safety rules were conservative: a method taking a `Span<T>` parameter was assumed to possibly
store it, so callers could not pass stack memory in many cases.

## 2. The feature explained

### 2.1 Syntax

```csharp
public ref struct Ref<T>
{
    private ref T _value;                       // ref field
    private ref readonly T _ro;                 // ref to readonly
    private readonly ref T _fixedTarget;        // ref cannot be reassigned
    public Ref(ref T value) => _value = ref value;
}

static void Fill(scoped Span<byte> buffer) { }  // scoped parameter
scoped Span<int> tmp = stackalloc int[16];      // scoped local
```

### 2.2 Semantics and rules

1. `ref` fields may only be declared in a `ref struct`.
2. A `ref` field is assigned with `= ref expr` (ref assignment); a plain `=` writes through the reference.
3. A default-initialized ref field is a **null ref**; reading it throws `NullReferenceException`. Check with `Unsafe.IsNullRef(ref field)`.
4. `scoped` on a parameter or local: the value may be used but cannot be returned or stored beyond the method.
5. `ref` parameters of `ref struct` types and `this` in struct methods are implicitly `scoped` in certain positions (C# 11 rule changes);
   `[UnscopedRef]` (`System.Diagnostics.CodeAnalysis`) opts out, for example to return a ref to a struct field.
6. `out` parameters are implicitly `scoped`.

## 3. Code examples

### 3.1 Minimal example

```csharp
public ref struct Cursor
{
    private ref int _value;
    public Cursor(ref int value) { _value = ref value; }
    public void Increment() => _value++;
}

int counter = 0;
var c = new Cursor(ref counter);
c.Increment();   // counter == 1
```

### 3.2 Extended examples

A span-backed writer:

```csharp
public ref struct SpanWriter
{
    private readonly Span<byte> _buffer;
    private int _pos;

    public SpanWriter(Span<byte> buffer) { _buffer = buffer; _pos = 0; }

    public void WriteInt32(int value)
    {
        BinaryPrimitives.WriteInt32LittleEndian(_buffer[_pos..], value);
        _pos += 4;
    }

    public int Written => _pos;
}

Span<byte> mem = stackalloc byte[64];
var w = new SpanWriter(mem);
w.WriteInt32(42);
```

`scoped` lets callers pass stack memory:

```csharp
static int Checksum(scoped ReadOnlySpan<byte> data)
{
    int sum = 0;
    foreach (byte b in data) sum += b;
    return sum;              // returning data would be an error
}

Span<byte> local = stackalloc byte[32];
int cs = Checksum(local);
```

Returning a reference to a struct field with `[UnscopedRef]`:

```csharp
public struct Stats
{
    private int _hp;
    [UnscopedRef] public ref int Hp => ref _hp;
}
```

## 4. Before this feature

```csharp
// No ref fields: store an index plus the array instead of a reference
public struct ArrayCursor
{
    private readonly int[] _array;
    private readonly int _index;
    public ArrayCursor(int[] array, int index) { _array = array; _index = index; }
    public void Increment() => _array[_index]++;
}
```

Or use `unsafe` pointers (requires pinning and `unsafe` context).

## 5. Under the hood

A `ref` field is stored in metadata as a by-ref field type (`int32&`). The runtime's GC tracks it as an interior
pointer, which is why it can only live inside `ref struct`s (stack-only). `scoped` is encoded with
`[ScopedRef]` on parameters; the assembly is marked with `[RefSafetyRules(11)]` so other compilers apply C# 11 rules.

## 6. Performance and allocation

- Zero allocation; enables span-like abstractions with no overhead compared with raw pointers.
- The JIT treats ref fields like `Span<T>`'s internal reference: fast and GC-safe.

## 7. Pitfalls

- Null refs: a `default` ref struct has null ref fields. Guard with `Unsafe.IsNullRef`.
- C# 11 ref-safety changes can make previously compiling code error (or vice versa) when upgrading; typical fix: add `scoped` or `[UnscopedRef]`.
- Ref structs cannot be boxed, stored in classes, captured by lambdas or (before C# 13) used in async methods.

## 8. Concerns and limitations

- Requires .NET 7+ runtime support.
- Complexity: the ref-safety rules are hard; prefer `Span<T>`, `Memory<T>` and BCL helpers.
- Godot: useful for allocation-free parsing of save files or network packets; engine APIs do not take ref structs.

## 9. When to use: clean-code guidance

### 9.1 Use when
- Building allocation-free parsers, cursors, writers, enumerators over spans.
- Accepting stack memory in helper methods (`scoped`).

### 9.2 Avoid when
- Ordinary application code: use `Span<T>` and existing APIs.

### 9.3 Good versus bad

```csharp
// Good: small, encapsulated, documented
public ref struct TokenReader { private ReadOnlySpan<char> _rest; ... }

// Bad: ref fields to "save copies" of small structs in business logic
public ref struct OrderRef { public ref Order Order; }
```

## 10. Related features

- [../12/ref-readonly-parameters.md](../12/ref-readonly-parameters.md)
- [../13/allows-ref-struct.md](../13/allows-ref-struct.md), [../13/ref-struct-interfaces.md](../13/ref-struct-interfaces.md)
- [../13/ref-and-unsafe-in-async-iterators.md](../13/ref-and-unsafe-in-async-iterators.md)

## 11. References

- [scoped ref](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/statements/declarations#scoped-ref)
- [ref struct types](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/ref-struct)
- [Proposal: Low-level struct improvements](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-11.0/low-level-struct-improvements)
