---
id: cs10-line-pragma-format
csharp: 10
tfm: net6.0
status: stable
tags: [preprocessor, line, source-generator, razor]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/preprocessor-directives#error-and-warning-information
---
# New `#line` pragma format (C# 10)

`#line` can map a span (start line/column to end line/column) of generated code to the original source:
`#line (1, 1) - (5, 60) 10 "partial-class.razor"`.

Only relevant for authors of code generators and templating systems (Razor, DSL compilers). Do not use in application code.
