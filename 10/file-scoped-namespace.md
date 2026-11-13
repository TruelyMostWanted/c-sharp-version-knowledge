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

```csharp
namespace MyGame.Units;

public partial class Archer : CharacterBody2D { }
```
Applies to the rest of the file and removes one indentation level.

## Rules
- Only one file-scoped namespace per file; it cannot be mixed with block-scoped namespaces.
- Enforce with `.editorconfig`: `csharp_style_namespace_declarations = file_scoped:warning` (IDE0161).
