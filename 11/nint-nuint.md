---
id: cs11-nint-nuint
csharp: 11
tfm: net7.0
status: stable
tags: [nint, nuint, IntPtr, interop, native]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/integral-numeric-types#native-sized-integers
---
# `nint` and `nuint` as aliases of `IntPtr` / `UIntPtr` (C# 11)

`nint` and `nuint` (native-sized integers, since C# 9) are now plain aliases for `System.IntPtr` and `System.UIntPtr`.
Arithmetic and conversions work the same on both spellings.

Prefer `nint` / `nuint` in new interop and low-level code; `IntPtr.Zero` and `nint.Zero` are the same.
