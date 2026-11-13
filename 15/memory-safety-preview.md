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

Start of a multi-release effort: `unsafe` should be tied to operations that actually access
unmanaged memory, not to the mere presence of pointer types.
**Do not use unless the project already opts into preview.**

## Pointer relaxations (LangVersion preview)
No `unsafe` context needed any more for:
- declaring pointer types and taking an address with `&`
- the `fixed` statement
- converting `stackalloc` to a pointer
- `sizeof` on any unmanaged type

Still requires `unsafe`: `*p`, `p->member`, `p[i]`, function pointer invocation.

```csharp
int number = 42;
int* pointer = &number;              // allowed without unsafe (preview)

int[] numbers = [10, 20, 30];
fixed (int* first = numbers)
{
    // dereferencing still needs unsafe
}
```

## `unsafe(expression)`
Unsafe context for a single expression where a block cannot appear
(field initializer, constructor initializer, `catch` filter). Needs `AllowUnsafeBlocks`.
```csharp
static readonly int Signature = unsafe(ReadSignature());
```

## Updated rules (feature flag `updated-memory-safety-rules`)
- `unsafe` on a member marks it *requires-unsafe*: callers must call it from an unsafe context.
- New contextual keyword `safe` marks `extern` members and fields in explicit or extended layout types as safe.
- The assembly is marked with `System.Runtime.CompilerServices.MemorySafetyRulesAttribute`.

## Pitfalls
- Semantics may change in later releases. Keep it out of production and out of benchmark code that must be reproducible.
- `Span<T>` is usually the better choice than raw pointers for structure-of-arrays code.
