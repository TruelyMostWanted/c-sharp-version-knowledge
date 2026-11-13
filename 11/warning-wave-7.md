---
id: cs11-warning-wave-7
csharp: 11
tfm: net7.0
status: stable
tags: [warnings, diagnostics]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/compiler-messages/warning-waves
---
# Warning wave 7 (C# 11)

New warnings enabled with `<AnalysisLevel>` 7 or higher (default for net7.0 and later).
Best known: **CS8981**: a type name consisting only of lowercase ASCII letters may clash with future keywords.

Rule for agents: never name types in all lowercase (`class record`, `class field`). This is exactly how
`field` (C# 14) and other contextual keywords break old code.
