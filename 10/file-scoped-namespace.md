---
id: cs10-file-scoped-namespace
csharp: 10
tfm: net6.0
status: stable
analyzer: IDE0161
tags: [namespace, style, indentation]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/namespace
modernizes: namespace X { ... } block around the whole file
---
# File-scoped namespace declaration (C# 10)

> **Abstract.** `namespace X;` declares that all types in the rest of the file belong to namespace `X`.
> It replaces the block form `namespace X { ... }` and removes one level of indentation from every line.
> The generated assembly is identical; this is the safest modernization there is.

| | |
|---|---|
| Introduced | C# 10 · .NET 6 (`net6.0`) |
| Status | Stable |
| Kind | Syntax |
| Runtime support needed | No (compile-time only) |
| Audience | Everyday code |
| Modernize category | AUTO (IDE0161) |

## 1. Motivation

Almost every C# file contains exactly one namespace. The block form spends a pair of braces and four spaces of
indentation on every line for information that applies to the whole file.

## 2. The feature explained

### 2.1 Syntax

```csharp
namespace Company.Product.Feature;
```

### 2.2 Semantics and rules

1. Applies from the declaration to the end of the file.
2. At most **one** file-scoped namespace per file; it cannot be combined with block-scoped namespaces in the same file.
3. `using` directives must come **before** it (or are placed inside by style choice only in the block form).
   Usings after the declaration are not allowed.
4. `extern alias` must also come first.
5. Nested namespaces are expressed with dots (`namespace A.B;`), not nesting.

## 3. Code examples

### 3.1 Minimal example

```csharp
namespace MyGame.Units;

public partial class Archer : CharacterBody2D { }
```

### 3.2 Extended examples

Typical file layout:

```csharp
using System.Text.Json;
using MyGame.Core;

namespace MyGame.Saving;

public sealed class SaveService
{
    public void Save(SaveGame game) { ... }
}

internal static class SaveFormat
{
    public const int Version = 3;
}
```

Project-wide enforcement in `.editorconfig`:

```ini
[*.cs]
csharp_style_namespace_declarations = file_scoped:warning
```

Bulk conversion:

```
dotnet format style --diagnostics IDE0161 --severity warn
```

## 4. Before this feature

```csharp
using System;

namespace MyGame.Units
{
    public partial class Archer : CharacterBody2D
    {
        public override void _Ready() { }
    }
}
```

## 5. Under the hood

Nothing. Namespaces exist only as a prefix of type names in metadata (`MyGame.Units.Archer`).
Block-scoped and file-scoped declarations produce byte-identical IL.

## 6. Performance and allocation

None.

## 7. Pitfalls

- A file that really contains two namespaces must stay block-scoped (or be split into two files, which is preferable).
- `using` directives placed **inside** the old block are resolved relative to the namespace; moving them above the
  file-scoped declaration can change how a short name resolves in rare cases (conflicting type names). Build after converting.

## 8. Concerns and limitations

- None for compatibility. Large diffs: convert in a dedicated commit so `git blame` stays useful
  (add the commit to `.git-blame-ignore-revs`).
- Godot: script classes work unchanged; the class name must still match the file name.

## 9. When to use: clean-code guidance

### 9.1 Use when
- Always, for every new file and every file you touch, when the project is on C# 10 or newer.

### 9.2 Avoid when
- The file has multiple namespaces (split it instead).

### 9.3 Good versus bad

```csharp
// Good
namespace MyGame.Ai;

public sealed class Planner { }

// Bad on C# 10+: needless block and indentation
namespace MyGame.Ai
{
    public sealed class Planner { }
}
```

## 10. Related features

- [global-using.md](global-using.md): the other half of trimming file headers.

## 11. References

- [namespace keyword](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/namespace)
- [IDE0161](https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/style-rules/ide0160-ide0161)
- [Proposal: File-scoped namespaces](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-10.0/file-scoped-namespaces)
