---
id: cs11-nint-nuint
csharp: 11
tfm: net7.0
status: stable
needs-runtime: true (numeric IntPtr features, .NET 7)
tags: [nint, nuint, IntPtr, interop, native]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/integral-numeric-types#native-sized-integers
modernizes: IntPtr / UIntPtr in new code
---
# `nint` and `nuint` as aliases of `IntPtr` / `UIntPtr` (C# 11)

> **Abstract.** The native-sized integer keywords `nint` and `nuint` (C# 9) are, since C# 11 on .NET 7, exact
> aliases of `System.IntPtr` and `System.UIntPtr`, just as `int` is an alias of `System.Int32`. Arithmetic,
> conversions and `checked` behaviour are the same on both spellings. Prefer the keywords in new code.

| | |
|---|---|
| Introduced | C# 9 (keywords); C# 11 · .NET 7 (full aliasing) |
| Status | Stable |
| Kind | Semantics (type identity) |
| Runtime support needed | Yes: .NET 7 (`RuntimeFeature.NumericIntPtr`) |
| Audience | Interop and low-level code |
| Modernize category | PREFER |

## 1. Motivation

In C# 9 and 10, `nint` and `IntPtr` were the same runtime type but **different to the compiler**: `nint` supported
arithmetic operators, `IntPtr` did not, and conversions differed. This made interop code confusing. .NET 7 added numeric
operators to `IntPtr` itself, so the language could unify them.

## 2. The feature explained

### 2.1 Syntax

```csharp
nint offset = 16;
nuint size = (nuint)buffer.Length;
IntPtr handle = nint.Zero;   // same type
```

### 2.2 Semantics and rules

1. `nint` ≡ `System.IntPtr`, `nuint` ≡ `System.UIntPtr` (when the runtime reports `NumericIntPtr`).
2. Size is 32 bits on 32-bit processes and 64 bits on 64-bit processes.
3. All arithmetic, comparison, shift and bitwise operators work on both spellings; `checked` contexts detect overflow.
4. Constants: `nint.MaxValue` / `nint.MinValue` are not compile-time constants (they depend on the platform).
5. Implicit conversions: `int` → `nint`, `nint` → `long`; explicit the other way.

## 3. Code examples

### 3.1 Minimal example

```csharp
nint a = 40;
IntPtr b = 2;
nint sum = a + b;   // 42; IntPtr arithmetic works directly in C# 11 / .NET 7
```

### 3.2 Extended examples

Interop with native handles:

```csharp
[LibraryImport("native_lib")]
private static partial nint CreateContext(nuint flags);

[LibraryImport("native_lib")]
private static partial void DestroyContext(nint context);

nint ctx = CreateContext(0);
try { /* ... */ } finally { DestroyContext(ctx); }
```

Pointer-sized offsets:

```csharp
unsafe
{
    byte* basePtr = (byte*)NativeMemory.Alloc(1024);
    nint offset = 128;
    byte* p = basePtr + offset;
    NativeMemory.Free(basePtr);
}
```

## 4. Before this feature

```csharp
IntPtr handle = new IntPtr(ptrValue);
IntPtr next = IntPtr.Add(handle, 16);    // C# 10: no + operator on IntPtr
long raw = handle.ToInt64();
```

## 5. Under the hood

Both spellings compile to the IL type `native int` / `native unsigned int`. Arithmetic uses the native IL instructions
(`add`, `mul`, `conv.i`) directly. Prior to C# 11 the compiler attached `[NativeInteger]` attributes to distinguish
`nint` from `IntPtr` in signatures; with numeric `IntPtr` this distinction is gone.

## 6. Performance and allocation

None; native-sized arithmetic is a single machine instruction.

## 7. Pitfalls

- Values that fit in 64 bits may overflow in a 32-bit process. Do not use `nint` for persisted data or network formats.
- `nint` in public APIs makes the API's value range platform-dependent.

## 8. Concerns and limitations

- On runtimes older than .NET 7, `nint` and `IntPtr` are still distinct to the C# 11 compiler (no numeric `IntPtr`).
- Godot: native handles from GDExtension or interop are `nint`/`IntPtr`; game data should use fixed-size `int` / `long`.

## 9. When to use: clean-code guidance

### 9.1 Use when
- Native handles, pointer offsets and sizes in interop code.

### 9.2 Avoid when
- Domain values, serialized data, counters: use `int` / `long`.

### 9.3 Good versus bad

```csharp
// Good
nint handle = Native.Open(path);

// Bad: save-game field whose size depends on the process bitness
public nint Gold;
```

## 10. Related features

- [ref-fields-scoped.md](ref-fields-scoped.md): low-level memory.
- [../15/memory-safety-preview.md](../15/memory-safety-preview.md)

## 11. References

- [Native-sized integers](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/integral-numeric-types#native-sized-integers)
- [Proposal: Numeric IntPtr](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-11.0/numeric-intptr)
