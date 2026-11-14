---
id: cs15-memory-safety-preview
csharp: 15
tfm: net11.0
status: preview
needs: LangVersion=preview; feature flag updated-memory-safety-rules for the new rules; AllowUnsafeBlocks for unsafe(...)
tags: [unsafe, pointers, memory-safety, fixed, stackalloc]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-15
---
# Memory safety (C# 15, PREVIEW)

> **Abstract.** C# 15 starts a multi-release effort to tie `unsafe` to operations that actually **access** unmanaged
> memory, instead of to the mere presence of pointer types. Declaring pointers, taking addresses, `fixed`, `stackalloc`
> to pointers and `sizeof` no longer need an `unsafe` context; dereferencing still does. An `unsafe(expr)` expression
> form and stricter "requires-unsafe" member rules (behind a feature flag) complete the picture.
> **Preview: do not use unless the project already opts in.**

| | |
|---|---|
| Introduced | C# 15 · .NET 11 (`net11.0`), preview |
| Status | Preview |
| Kind | Language (safety rules) |
| Runtime support needed | `MemorySafetyRulesAttribute` (.NET 11) for the updated rules |
| Audience | Low-level and interop code; early adopters |
| Modernize category | NEVER-AUTO (preview) |

## 1. Motivation

`unsafe` marked whole methods or blocks as soon as a pointer **type** appeared, even if the code only passed addresses
around. Reviews could not tell where memory was actually read or written, and large `unsafe` regions hid the dangerous
lines. Other ecosystems (Rust) show the value of narrowly scoped unsafety; .NET aims for audit-friendly code where
`unsafe` marks exactly the risky operations and the APIs that require them.

## 2. The feature explained

### 2.1 Syntax

```csharp
int* p = &number;                  // no unsafe needed (preview)
unsafe { *p = 5; }                 // dereference still needs unsafe
static readonly int Sig = unsafe(ReadSignature());   // unsafe expression
```

### 2.2 Semantics and rules

Pointer relaxations (`LangVersion` preview): no `unsafe` context needed for

1. declaring pointer types and taking an address with `&`,
2. the `fixed` statement,
3. converting `stackalloc` to a pointer,
4. `sizeof` on any unmanaged type.

Still requires `unsafe`: `*p`, `p->member`, `p[i]`, function pointer invocation.

`unsafe(expression)`: an unsafe context for a single expression where a block cannot appear (field initializer, constructor
initializer, `catch` filter). Needs `AllowUnsafeBlocks`.

Updated rules (feature flag `updated-memory-safety-rules`):

5. `unsafe` on a member marks it **requires-unsafe**: callers must call it from an unsafe context.
6. New contextual keyword `safe` marks `extern` members and fields in explicit or extended layout types as safe to use.
7. The assembly is marked with `System.Runtime.CompilerServices.MemorySafetyRulesAttribute`.

## 3. Code examples

### 3.1 Minimal example

```csharp
int number = 42;
int* pointer = &number;              // allowed without unsafe (preview)

int[] numbers = [10, 20, 30];
fixed (int* first = numbers)
{
    // dereferencing still needs unsafe
    unsafe { Console.WriteLine(*first); }
}
```

### 3.2 Extended examples

Narrow unsafe region around the one dangerous line:

```csharp
static int SumRaw(int[] data)
{
    int sum = 0;
    fixed (int* p = data)                // fine without unsafe
    {
        for (int i = 0; i < data.Length; i++)
            sum += unsafe(p[i]);         // only the read is unsafe
    }
    return sum;
}
```

Requires-unsafe API (with the feature flag):

```csharp
public static class NativeBuffer
{
    public static unsafe void CopyRaw(byte* src, byte* dst, int count)   // callers need unsafe
        => Buffer.MemoryCopy(src, dst, count, count);
}

unsafe { NativeBuffer.CopyRaw(a, b, 16); }
```

Project setup:

```xml
<PropertyGroup>
  <LangVersion>preview</LangVersion>
  <AllowUnsafeBlocks>true</AllowUnsafeBlocks>
  <Features>$(Features);updated-memory-safety-rules</Features>
</PropertyGroup>
```

## 4. Before this feature

```csharp
static unsafe int SumRaw(int[] data)    // whole method unsafe because of pointer types
{
    int sum = 0;
    fixed (int* p = data)
        for (int i = 0; i < data.Length; i++) sum += p[i];
    return sum;
}
```

## 5. Under the hood

The relaxations change only where the compiler demands an unsafe context; generated IL for pointer operations is the
same. With the updated rules, requires-unsafe members are marked in metadata so that other assemblies' callers are
checked, and the assembly carries `MemorySafetyRulesAttribute`.

## 6. Performance and allocation

None; this is a safety and auditing feature.

## 7. Pitfalls

- Semantics may change in later releases. Keep it out of production and out of benchmark code that must be reproducible.
- Taking an address without `fixed` of a movable object is still invalid; the relaxation does not make pointers safe.

## 8. Concerns and limitations

- Preview applies project-wide and unlocks other preview features.
- `Span<T>` is usually the better choice than raw pointers, including for structure-of-arrays code.
- Godot: Godot C# projects rarely need pointers; interop goes through generated bindings.

## 9. When to use: clean-code guidance

### 9.1 Use when
- Only in projects that already use `LangVersion=preview` and want to experiment with narrower unsafe regions.

### 9.2 Avoid when
- Everywhere else. Prefer spans, `MemoryMarshal`, `Unsafe` helpers in stable C#.

### 9.3 Good versus bad

```csharp
// Good (preview project): unsafe only around the dereference
sum += unsafe(p[i]);

// Bad: enabling preview in a production project just to drop `unsafe` keywords
```

## 10. Related features

- [../11/ref-fields-scoped.md](../11/ref-fields-scoped.md)
- [../12/inline-arrays.md](../12/inline-arrays.md)
- [../11/nint-nuint.md](../11/nint-nuint.md)

## 11. References

- [What's new in C# 15](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-15)
- [Unsafe code, pointer types and function pointers](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/unsafe-code)
