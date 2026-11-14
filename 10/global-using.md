---
id: cs10-global-using
csharp: 10
tfm: net6.0
status: stable
tags: [using, global-using, implicit-usings, project-setup]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/using-directive
modernizes: the same using block repeated in every file
---
# `global using` directives (C# 10)

> **Abstract.** A `using` directive prefixed with `global` applies to **every file of the compilation**.
> Combined with the SDK switch `<ImplicitUsings>`, it removes the identical block of `using` lines from the top
> of most files. Keep all global usings in one place so readers know where names come from.

| | |
|---|---|
| Introduced | C# 10 · .NET 6 (`net6.0`) |
| Status | Stable |
| Kind | Syntax |
| Runtime support needed | No (compile-time only) |
| Audience | Everyday code, project setup |
| Modernize category | PREFER |

## 1. Motivation

Most files in a project start with the same five to fifteen `using` lines (`System`, `System.Linq`,
`System.Collections.Generic`, the project's core namespace, the engine namespace). They add noise,
cause merge conflicts and drift between files.

## 2. The feature explained

### 2.1 Syntax

```csharp
global using System.Text.Json;              // namespace
global using static System.Math;            // static members
global using Vec = System.Numerics.Vector3; // alias
```

MSBuild equivalents in the project file:

```xml
<ItemGroup>
  <Using Include="System.Text.Json" />
  <Using Include="System.Math" Static="true" />
  <Using Include="System.Numerics.Vector3" Alias="Vec" />
  <Using Remove="System.Net.Http" />           <!-- remove one implicit using -->
</ItemGroup>
```

### 2.2 Semantics and rules

1. A `global using` may appear in any file but must come **before** all non-global `using` directives and before any namespace or type declaration.
2. Its scope is the whole compilation (the project), not other projects that reference it.
3. All three forms are allowed: namespace, `static`, alias.
4. `<ImplicitUsings>enable</ImplicitUsings>` makes the SDK generate `obj/.../<Project>.GlobalUsings.g.cs` with a set
   that depends on the SDK (`Microsoft.NET.Sdk`: `System`, `System.IO`, `System.Linq`, `System.Net.Http`,
   `System.Threading`, `System.Threading.Tasks`, `System.Collections.Generic`; Web SDK adds ASP.NET namespaces).
5. Duplicate global and local usings of the same namespace are allowed but produce a hidden diagnostic (redundant using).

## 3. Code examples

### 3.1 Minimal example

```csharp
// GlobalUsings.cs
global using System;
global using System.Collections.Generic;
global using Godot;
global using static System.Math;
```

### 3.2 Extended examples

A project-level `GlobalUsings.cs` with grouping and comments:

```csharp
// GlobalUsings.cs: the only file with global usings in this project.

// BCL
global using System;
global using System.Collections.Generic;
global using System.Diagnostics;
global using System.Linq;

// Engine
global using Godot;

// Project core
global using MyGame.Core;
global using MyGame.Core.Events;

// Disambiguation: Godot and System.Numerics both have Vector2
global using Vector2 = Godot.Vector2;
```

Test projects:

```csharp
// Tests/GlobalUsings.cs
global using Xunit;
global using FluentAssertions;
global using MyGame.Core;
```

## 4. Before this feature

```csharp
// Archer.cs
using System;
using System.Collections.Generic;
using System.Linq;
using Godot;
using MyGame.Core;

namespace MyGame.Units;
...
// Mage.cs: the same five lines again
```

## 5. Under the hood

Nothing reaches IL. Usings only affect name lookup during compilation. A `global using` is treated as if it
appeared at the top of every file. Implicit usings are an ordinary generated source file in `obj/`.

## 6. Performance and allocation

None at runtime. Compile time is unaffected in practice.

## 7. Pitfalls

- **Name clashes.** A global `using Godot;` plus `using System.Numerics;` in one file makes `Vector2` ambiguous.
  Resolve with a global alias or fully qualified names.
- Scattered `global using` lines in random files are hard to find. Keep them in one file.
- `ImplicitUsings` brings `System.Net.Http` and `System.Threading.Tasks` even into projects that never use them;
  harmless, but remove with `<Using Remove="..."/>` if it causes ambiguity.

## 8. Concerns and limitations

- Readability: readers no longer see the imports at the top of a file. IDEs show them; plain-text readers
  (code review, agents) must know where `GlobalUsings.cs` is.
- Global usings do not flow to referencing projects; each project defines its own.
- Godot: Godot C# projects can use `ImplicitUsings`; add `global using Godot;` yourself if desired.

## 9. When to use: clean-code guidance

### 9.1 Use when
- A namespace is used in a large majority of files (BCL basics, engine, project core).

### 9.2 Avoid when
- A namespace is used by a few files only, or contains names that clash with others (keep it local).

### 9.3 Good versus bad

```csharp
// Good: one file, grouped, short
// GlobalUsings.cs
global using Godot;
global using MyGame.Core;

// Bad: hidden at the top of an unrelated file
// Enemy.cs
global using System.Text.RegularExpressions;
using Godot;
```

Agent rules: keep all global usings in **one** file; do not add a local `using` that a global using already covers.

## 10. Related features

- [file-scoped-namespace.md](file-scoped-namespace.md): the other half of shorter file headers.
- [../12/alias-any-type.md](../12/alias-any-type.md): global aliases for tuples and arrays.

## 11. References

- [using directive](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/using-directive)
- [Implicit using directives](https://learn.microsoft.com/en-us/dotnet/core/project-sdk/overview#implicit-using-directives)
