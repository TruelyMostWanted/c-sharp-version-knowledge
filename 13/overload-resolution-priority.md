---
id: cs13-overload-resolution-priority
csharp: 13
tfm: net9.0
status: stable
needs-runtime: true (OverloadResolutionPriorityAttribute, .NET 9 or polyfill)
tags: [overload, library, attribute, api-design]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/attributes/general#overloadresolutionpriority-attribute
---
# Overload resolution priority (C# 13)

> **Abstract.** `[OverloadResolutionPriority(n)]` (`System.Runtime.CompilerServices`) makes the compiler prefer
> overloads with a higher priority **before** applying the normal "better function member" rules. Library authors
> use it to steer existing call sites to a new, better overload (often span-based) without ambiguity errors and
> without removing the old overload (binary compatibility).

| | |
|---|---|
| Introduced | C# 13 · .NET 9 (`net9.0`) |
| Status | Stable |
| Kind | Compiler-recognized attribute |
| Runtime support needed | Attribute type from .NET 9 (can be polyfilled) |
| Audience | Library authors only |
| Modernize category | n/a |

## 1. Motivation

Adding an overload to a published API is risky: existing calls may become ambiguous, or keep binding to the old,
slower overload because normal rules prefer it. Removing the old overload breaks binaries. Library authors needed a
way to say "if both apply, pick this one".

## 2. The feature explained

### 2.1 Syntax

```csharp
[OverloadResolutionPriority(1)]
public static void Write(params ReadOnlySpan<string> parts) { }
```

### 2.2 Semantics and rules

1. Default priority is 0. Higher wins; negative values push overloads back.
2. Priorities are compared only among candidates from the **same type** (not across a type and its base, nor across extension types).
3. After filtering by priority, normal overload resolution runs on the remaining candidates.
4. Applies to methods, constructors and indexers. Ignored on overrides (the original declaration's priority counts).
5. Older compilers ignore the attribute.

## 3. Code examples

### 3.1 Minimal example

```csharp
public static class Log
{
    public static void Write(params string[] parts) { /* allocates the array */ }

    [OverloadResolutionPriority(1)]
    public static void Write(params ReadOnlySpan<string> parts) { /* no allocation */ }
}

Log.Write("a", "b");   // binds to the span overload
```

### 3.2 Extended examples

Deprecating an overload softly:

```csharp
public sealed class Inventory
{
    [OverloadResolutionPriority(-1)]
    [Obsolete("Use Add(ItemId, int)")]
    public void Add(string itemName, int count) { }

    public void Add(ItemId id, int count) { }
}
```

Resolving an ambiguity between interface-based overloads:

```csharp
public static class Serializer
{
    public static void Write(IEnumerable<int> values) { }

    [OverloadResolutionPriority(1)]
    public static void Write(IReadOnlyList<int> values) { }   // preferred for List<int>, arrays
}
```

## 4. Before this feature

Library authors either lived with the old overload being chosen, renamed the new method (`WriteSpan`), or hid the old one with `[EditorBrowsable(Never)]` while existing calls still bound to it.

## 5. Under the hood

The attribute is stored in metadata; the compiler reads it during overload resolution. No IL difference at the call site except which method is called.

## 6. Performance and allocation

None by itself; typically used to route calls to allocation-free overloads.

## 7. Pitfalls

- Recompiling consumers silently changes which overload they call; behaviour must be equivalent.
- Priorities do not apply across inheritance or extension containers; tests are essential.

## 8. Concerns and limitations

- Hidden rules confuse readers ("why does this call go there?"). Document priorities in XML docs.
- Not for application code.

## 9. When to use: clean-code guidance

### 9.1 Use when
- Evolving a public library API with a better overload that should win for existing call sites.

### 9.2 Avoid when
- Application code, or to paper over badly designed overload sets (redesign instead).

### 9.3 Good versus bad

```csharp
// Good: equivalent semantics, better performance
[OverloadResolutionPriority(1)] public static int Sum(params ReadOnlySpan<int> xs) { ... }

// Bad: different behaviour hidden behind priority
[OverloadResolutionPriority(1)] public static void Save(object o) { /* different format! */ }
```

## 10. Related features

- [params-collections.md](params-collections.md)
- [../14/implicit-span-conversions.md](../14/implicit-span-conversions.md)
- [../12/experimental-attribute.md](../12/experimental-attribute.md)

## 11. References

- [OverloadResolutionPriority attribute](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/attributes/general#overloadresolutionpriority-attribute)
- [Proposal: Overload resolution priority](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-13.0/overload-resolution-priority)
