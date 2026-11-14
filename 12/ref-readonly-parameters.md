---
id: cs12-ref-readonly-parameters
csharp: 12
tfm: net8.0
status: stable
tags: [ref, readonly, in, struct, performance, api-design]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/method-parameters#ref-readonly-modifier
modernizes: ref parameters that are never written
---
# `ref readonly` parameters (C# 12)

> **Abstract.** A `ref readonly` parameter passes a **read-only reference** to a variable. It sits between `ref`
> (callee may write, caller must pass a variable) and `in` (read-only, caller may pass temporaries).
> Use it for APIs that must receive a real variable (an address that matters) but never modify it, and to
> migrate existing `ref` APIs to read-only without breaking callers.

| | |
|---|---|
| Introduced | C# 12 · .NET 8 (`net8.0`) |
| Status | Stable |
| Kind | Syntax + semantics (parameter passing) |
| Runtime support needed | No (`RequiresLocationAttribute` is emitted by the compiler if missing) |
| Audience | Library authors, performance code with large structs |
| Modernize category | CASE (API) |

## 1. Motivation

`in` (C# 7.2) passes a read-only reference, but callers may pass rvalues (`M(GetMatrix())`, `M(5)`), for which the
compiler silently creates a temporary. APIs whose semantics depend on the **identity** of the variable (e.g.
`MemoryMarshal.CreateReadOnlySpan(ref readonly T, int)`, `Unsafe.AsRef`) could not use `in` safely and used `ref`
instead, which wrongly suggests mutation and forbids readonly fields as arguments.

## 2. The feature explained

### 2.1 Syntax

```csharp
void M(ref readonly Matrix4x4 m);

M(ref matrix);    // OK
M(in matrix);     // OK
M(matrix);        // warning: pass with ref or in
M(GetMatrix());   // warning: rvalue, a temporary is created
```

### 2.2 Semantics and rules

1. Inside the method the parameter is read-only (like `in`).
2. Callers should pass a variable with `ref` or `in`. Passing without modifier or passing an rvalue is a **warning**, not an error.
3. Readonly fields can be passed with `in` (not with `ref`).
4. Changing `ref` → `ref readonly` is **source and binary compatible** for callers (they still use `ref`).
5. Changing `in` → `ref readonly` is source compatible (with warnings for rvalues). Changing `ref readonly` → `in` is also mostly compatible.
6. Overloads cannot differ only by `ref` vs `ref readonly` vs `in`.

## 3. Code examples

### 3.1 Minimal example

```csharp
public static float Determinant(ref readonly Matrix4x4 m) => m.GetDeterminant();

var mat = Matrix4x4.Identity;
float d = Determinant(ref mat);   // or `in mat`
```

### 3.2 Extended examples

API that needs the variable's address:

```csharp
public static ReadOnlySpan<T> AsSingleSpan<T>(ref readonly T value)
    => MemoryMarshal.CreateReadOnlySpan(in value, 1);   // span over the caller's variable, not a temporary
```

Migrating an existing `ref` API without breaking callers:

```csharp
// v1
public static int Hash(ref LargeKey key) { ... }       // never wrote to key

// v2
public static int Hash(ref readonly LargeKey key) { ... }
// existing callers `Hash(ref k)` compile unchanged; now `Hash(in readonlyField)` also works
```

Large struct read in a hot loop:

```csharp
public readonly struct TerrainChunk { /* 256 bytes of data */ }

static float SampleHeight(ref readonly TerrainChunk chunk, int x, int z) { ... }

for (int i = 0; i < chunks.Length; i++)
    total += SampleHeight(ref chunks[i], x, z);   // no 256-byte copy
```

## 4. Before this feature

```csharp
public static float Determinant(in Matrix4x4 m) => ...;   // accepts temporaries silently
public static float Determinant2(ref Matrix4x4 m) => ...; // suggests mutation; readonly fields not accepted
```

## 5. Under the hood

In IL a `ref readonly` parameter is a managed pointer (`valuetype Matrix4x4&`) like `ref` and `in`, marked with
`[RequiresLocation]` (`System.Runtime.CompilerServices.RequiresLocationAttribute`) so compilers know callers should pass a location.
(`in` parameters are marked with `[IsReadOnly]` / `modreq(InAttribute)` on virtual methods.)
For rvalue arguments the compiler creates a hidden local and passes its address.

## 6. Performance and allocation

- Avoids copying large structs (same as `in`).
- For small structs (≤ 16 bytes) passing by value is usually as fast or faster; do not use refs for `int`, `Vector2`, ...
- **Defensive copies:** calling non-readonly members on a `ref readonly`/`in` parameter of a non-`readonly struct` copies the struct first. Make structs `readonly struct` (or members `readonly`).

## 7. Pitfalls

- Passing without `ref`/`in` is only a warning; treat it as an error in hot code.
- Defensive copies (see above) can make `ref readonly` slower than by-value.

## 8. Concerns and limitations

- API design: prefer `in` for new APIs that should also accept temporaries; prefer `ref readonly` when a variable is required or when migrating `ref` APIs.
- Godot: Godot math structs are small; pass them by value.

## 9. When to use: clean-code guidance

### 9.1 Use when
- The callee relies on the caller's variable identity (spans over it, address comparisons).
- Migrating `ref` parameters that are never written.

### 9.2 Avoid when
- Small structs and primitives.
- General APIs where `in` (accepting temporaries) is friendlier.

### 9.3 Good versus bad

```csharp
// Good
public static ReadOnlySpan<byte> Bytes<T>(ref readonly T value) where T : unmanaged
    => MemoryMarshal.AsBytes(MemoryMarshal.CreateReadOnlySpan(in value, 1));

// Bad: by-ref for a 4-byte value
static int Square(ref readonly int x) => x * x;
```

## 10. Related features

- [../11/ref-fields-scoped.md](../11/ref-fields-scoped.md)
- [../14/lambda-parameter-modifiers.md](../14/lambda-parameter-modifiers.md): `ref readonly` on lambda parameters.

## 11. References

- [ref readonly modifier](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/method-parameters#ref-readonly-modifier)
- [Proposal: ref readonly parameters](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-12.0/ref-readonly-parameters)
