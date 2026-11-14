---
id: cs13-allows-ref-struct
csharp: 13
tfm: net9.0
status: stable
needs-runtime: true (.NET 9, byref-like generic parameters)
tags: [generics, ref-struct, span, anti-constraint]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-13.0/ref-struct-interfaces
related: ref-struct-interfaces.md
---
# `allows ref struct` anti-constraint (C# 13)

> **Abstract.** `where T : allows ref struct` lets a type parameter accept **ref struct** types such as `Span<T>`,
> `ReadOnlySpan<T>` or your own ref structs. It is an *anti*-constraint: it widens what `T` may be, and in return the
> compiler enforces ref-safety rules on every use of `T` inside the generic code. The BCL uses it for
> `Func<>`/`Action<>`, `IEnumerable<T>`-adjacent APIs and comparers so they can work with spans.

| | |
|---|---|
| Introduced | C# 13 · .NET 9 (`net9.0`) |
| Status | Stable |
| Kind | Language (generic constraints) + runtime |
| Runtime support needed | Yes: .NET 9 |
| Audience | Library authors, low-level generic code |
| Modernize category | CASE |

## 1. Motivation

Ref structs live only on the stack, so generic code was forbidden from using them as type arguments: a generic
method might box `T`, store it in a field or capture it in a lambda, all of which would break stack safety.
As a result `Func<ReadOnlySpan<char>, int>` was impossible, and every span-friendly algorithm needed its own
non-generic overload.

## 2. The feature explained

### 2.1 Syntax

```csharp
public static TResult Apply<T, TResult>(T value, Func<T, TResult> f)
    where T : allows ref struct
    => f(value);

public interface IParser<TInput> where TInput : allows ref struct { int Parse(TInput input); }
```

### 2.2 Semantics and rules

1. `allows ref struct` must come **last** in the constraint list.
2. Inside the generic code, values of type `T` follow ref-struct rules: no boxing (no conversion to `object` or interfaces),
   no storage in fields of non-ref types, no capture in lambdas or local functions, no use across `await` / `yield`, no arrays of `T`.
3. Ref structs passed as `T` still cannot be boxed; instance members of interfaces are called through constrained calls.
4. It can be combined with other constraints (`where T : IDisposable, allows ref struct`), then the ref struct must implement the interface (C# 13 allows that).
5. Existing BCL delegates (`Func<...>`, `Action<...>`) and many interfaces were annotated in .NET 9.

## 3. Code examples

### 3.1 Minimal example

```csharp
public class C<T> where T : allows ref struct
{
    public void M(scoped T p)
    {
        // p must follow ref safety rules: no boxing, no storing in fields, no capture in lambdas
    }
}
```

### 3.2 Extended examples

Spans with `Func<>`:

```csharp
Func<ReadOnlySpan<char>, int> countDigits = s =>
{
    int n = 0;
    foreach (char c in s) if (char.IsDigit(c)) n++;
    return n;
};

int digits = countDigits("a1b2c3");   // 3
```

Generic helper usable with spans:

```csharp
public static int CountWhere<TSpan>(TSpan items, Func<TSpan, int, bool> predicate, int length)
    where TSpan : allows ref struct
{
    int count = 0;
    for (int i = 0; i < length; i++)
        if (predicate(items, i)) count++;
    return count;
}
```

Generic interface over ref struct inputs (parsers):

```csharp
public interface ITokenParser<TInput> where TInput : allows ref struct
{
    bool TryParse(TInput input, out int consumed);
}

public sealed class IntParser : ITokenParser<ReadOnlySpan<char>>
{
    public bool TryParse(ReadOnlySpan<char> input, out int consumed)
    {
        consumed = 0;
        while (consumed < input.Length && char.IsDigit(input[consumed])) consumed++;
        return consumed > 0;
    }
}
```

## 4. Before this feature

```csharp
// Custom delegate per span type
public delegate int SpanFunc(ReadOnlySpan<char> s);
SpanFunc countDigits = s => ...;

// Or array-based APIs that force allocations
Func<char[], int> countDigits2 = arr => ...;
```

## 5. Under the hood

The generic parameter gets the `AllowByRefLike` flag (`GenericParameterAttributes.AllowByRefLike`) in metadata.
The .NET 9 runtime and JIT then permit byref-like instantiations and generate specialized code per ref struct type
(like value types). Constrained calls are used for interface members.

## 6. Performance and allocation

- Enables allocation-free generic APIs over spans; no boxing by construction.
- Each ref struct instantiation gets its own native code (code size).

## 7. Pitfalls

- Adding `allows ref struct` to an existing generic method can **break its body** (e.g. `object o = value;` or LINQ over `T` no longer compiles). Add it only where the body is already ref-safe.
- Removing it later is a breaking change for callers passing spans.

## 8. Concerns and limitations

- Requires .NET 9 runtime support.
- Readability: constraint-heavy APIs are harder to understand; document why.

## 9. When to use: clean-code guidance

### 9.1 Use when
- Writing generic utilities, delegates or interfaces that should work with `Span<T>` and other ref structs.

### 9.2 Avoid when
- Application code that does not handle spans generically.

### 9.3 Good versus bad

```csharp
// Good: body is ref-safe
static bool AnyMatch<T>(T input, Func<T, bool> pred) where T : allows ref struct => pred(input);

// Bad: body boxes T, so the anti-constraint makes it fail to compile
static object Box<T>(T value) where T : allows ref struct => value;   // error: T may be a ref struct
```

## 10. Related features

- [ref-struct-interfaces.md](ref-struct-interfaces.md)
- [ref-and-unsafe-in-async-iterators.md](ref-and-unsafe-in-async-iterators.md)
- [../11/ref-fields-scoped.md](../11/ref-fields-scoped.md)

## 11. References

- [What's new in C# 13: allows ref struct](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-13#allows-ref-struct)
- [Generic constraints: allows ref struct](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/where-generic-type-constraint)
- [Proposal: ref struct interfaces and allows ref struct](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-13.0/ref-struct-interfaces)
