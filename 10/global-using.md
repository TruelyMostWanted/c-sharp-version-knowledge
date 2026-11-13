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

A `global using` applies to every file in the project.

```csharp
// GlobalUsings.cs
global using System;
global using System.Collections.Generic;
global using Godot;
global using static System.Math;
```

## Rules
- Must come before any non-global `using` and before namespace or type declarations in that file.
- SDK feature `<ImplicitUsings>enable</ImplicitUsings>` adds a standard set automatically (generated file in `obj/`).

## Conventions for agents
- Keep all global usings in **one** file (`GlobalUsings.cs`); do not scatter them.
- Do not add a `using` to a file if a global using already covers it.
