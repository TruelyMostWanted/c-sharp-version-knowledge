---
id: cs13-ref-struct-interfaces
csharp: 13
tfm: net9.0
status: stable
needs-runtime: true (.NET 9)
tags: [ref-struct, interface, generics]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/ref-struct
related: allows-ref-struct.md
---
# `ref struct` types can implement interfaces (C# 13)

> **Abstract.** A `ref struct` may implement interfaces. It still can never be **converted** to the interface type
> (that would box it onto the heap), so interface members are reached only through a generic type parameter
> constrained to the interface **and** `allows ref struct`. Together the two features enable generic, allocation-free
> algorithms over span-like types.

| | |
|---|---|
| Introduced | C# 13 · .NET 9 (`net9.0`) |
| Status | Stable |
| Kind | Language (type system) |
| Runtime support needed | Yes: .NET 9 |
| Audience | Library authors, low-level code |
| Modernize category | n/a |

## 1. Motivation

Ref structs could not implement interfaces, so they could not participate in generic abstractions (`IDisposable`
pattern aside, which the compiler special-cased). `Span<T>` enumerators, readers and writers could not share a contract.

## 2. The feature explained

### 2.1 Syntax

```csharp
interface IVisitor { void Visit(int value); }

ref struct SumVisitor : IVisitor
{
    public int Sum;
    public void Visit(int value) => Sum += value;
}
```

### 2.2 Semantics and rules

1. A `ref struct` can list interfaces in its base list.
2. It cannot be converted to the interface type (no boxing): `IVisitor v = new SumVisitor();` is an error.
3. Explicit interface implementations are reachable only through a type parameter that `allows ref struct`.
4. It must implement **all** interface members, including those that have default implementations (default bodies cannot be used, because calling them would require boxing).
5. Use generic methods with `where T : IInterface, allows ref struct` to call interface members.

## 3. Code examples

### 3.1 Minimal example

```csharp
interface IVisitor { void Visit(int value); }

ref struct SumVisitor : IVisitor
{
    public int Sum;
    public void Visit(int value) => Sum += value;
}

static void Walk<T>(ReadOnlySpan<int> data, ref T visitor) where T : IVisitor, allows ref struct
{
    foreach (var v in data) visitor.Visit(v);
}

var sum = new SumVisitor();
Walk([1, 2, 3], ref sum);   // sum.Sum == 6
```

### 3.2 Extended examples

A writer contract implemented by a span writer and a stream writer:

```csharp
public interface IByteSink
{
    void Write(ReadOnlySpan<byte> data);
}

public ref struct SpanSink(Span<byte> target) : IByteSink
{
    private Span<byte> _target = target;
    private int _pos;
    public void Write(ReadOnlySpan<byte> data) { data.CopyTo(_target[_pos..]); _pos += data.Length; }
}

public sealed class StreamSink(Stream s) : IByteSink
{
    public void Write(ReadOnlySpan<byte> data) => s.Write(data);
}

static void WriteHeader<TSink>(ref TSink sink) where TSink : IByteSink, allows ref struct
{
    sink.Write("SAVE"u8);
    sink.Write([1, 0]);
}
```

`IDisposable` through generics:

```csharp
static void Use<T>(T resource) where T : IDisposable, allows ref struct
{
    using (resource) { /* ... */ }
}
```

## 4. Before this feature

Duplicate code per ref struct type, or delegates/function pointers to abstract over behaviour.

## 5. Under the hood

The interface implementation is recorded in metadata as for any struct. Calls through a constrained type parameter use
`constrained.` prefixes, and the JIT specializes the generic method per ref struct, so calls are direct.

## 6. Performance and allocation

No boxing; specialized code per instantiation, usually inlined.

## 7. Pitfalls

- Default interface members must still be implemented in the ref struct.
- Passing the ref struct **by value** to a generic method copies it; mutations are lost. Pass `ref T` when state matters (see `Walk`).
- Adding a member to the interface breaks every ref struct implementing it (no default bodies help).

## 8. Concerns and limitations

- Requires .NET 9.
- Complexity: combine with `allows ref struct`, `scoped` and ref rules; keep such code isolated.

## 9. When to use: clean-code guidance

### 9.1 Use when
- Generic algorithms over several span-like ref structs (readers, writers, visitors, enumerators).

### 9.2 Avoid when
- A normal struct or class would do; ref structs are restrictive.

### 9.3 Good versus bad

```csharp
// Good
static int Count<T>(ref T reader) where T : ITokenReader, allows ref struct { ... }

// Bad: tries to box
ITokenReader r = new SpanTokenReader(text);   // error
```

## 10. Related features

- [allows-ref-struct.md](allows-ref-struct.md)
- [../11/ref-fields-scoped.md](../11/ref-fields-scoped.md)

## 11. References

- [ref struct types](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/ref-struct)
- [What's new in C# 13: ref struct interfaces](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-13#ref-struct-interfaces)
- [Proposal: ref struct interfaces](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-13.0/ref-struct-interfaces)
