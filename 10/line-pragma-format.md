---
id: cs10-line-pragma-format
csharp: 10
tfm: net6.0
status: stable
tags: [preprocessor, line, source-generator, razor]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/preprocessor-directives#error-and-warning-information
---
# New `#line` span format (C# 10)

> **Abstract.** `#line` can map a **span** of generated C# (start and end line/column) to a span in the original
> source file, optionally with a character offset. Debuggers and diagnostics then point at the right place in
> Razor, templates or DSL files. Only authors of code generators need this; application code never does.

| | |
|---|---|
| Introduced | C# 10 · .NET 6 (`net6.0`) |
| Status | Stable |
| Kind | Preprocessor directive |
| Runtime support needed | No (affects PDB and diagnostics only) |
| Audience | Code-generator and template-engine authors |
| Modernize category | n/a |

## 1. Motivation

The classic `#line 42 "file.razor"` maps only **lines**. Generated code often places a user expression in the
middle of a line with different columns than in the source (`@model.Name` in Razor becomes
`__builder.AddContent(0, model.Name);`). Breakpoints, sequence points and error squiggles were therefore placed on
the wrong columns.

## 2. The feature explained

### 2.1 Syntax

```
#line (startLine, startChar) - (endLine, endChar) [charOffset] "file"
```

### 2.2 Semantics and rules

1. The span `(startLine, startChar) - (endLine, endChar)` (1-based) is the location in the **original** file.
2. The directive applies to the **next line** of generated code.
3. `charOffset` (optional) is the column in the generated line where the mapped text starts; the compiler maps
   columns relative to it.
4. `#line default` and `#line hidden` keep their meaning (restore mapping / hide from debugger).

## 3. Code examples

### 3.1 Minimal example

```csharp
#line (1, 1) - (5, 60) 10 "partial-class.razor"
        __builder.AddContent(0, someExpression);
#line default
```

### 3.2 Extended examples

A generator mapping one user expression:

```csharp
// Template source (unit.tmpl), line 3: "Hello {{ player.Name }}!"
// Generated:
#line (3, 10) - (3, 21) 27 "unit.tmpl"
        output.Append(player.Name);
#line default
```

Hiding generator plumbing from the debugger:

```csharp
#line hidden
        var __tmp = new TemplateContext();
#line default
```

## 4. Before this feature

```csharp
#line 3 "unit.tmpl"
        output.Append(player.Name);   // whole line mapped; columns wrong
#line default
```

## 5. Under the hood

`#line` changes no IL. It changes the **sequence points** written into the PDB and the locations of compiler
diagnostics. The debugger uses these to step and to place breakpoints in the original file.

## 6. Performance and allocation

None.

## 7. Pitfalls

- Off-by-one errors (1-based lines and columns) produce subtly wrong breakpoints; test in a debugger.
- Forgetting `#line default` maps all following generated code to the template.

## 8. Concerns and limitations

- Only meaningful when a debugger or user sees the original file (Razor, DSLs, templating).
- Source generators usually do not need it unless they translate user-written non-C# source.

## 9. When to use: clean-code guidance

### 9.1 Use when
- Writing a template engine, Razor-like compiler or DSL that emits C# from user files.

### 9.2 Avoid when
- Any hand-written application code. Never add `#line` manually.

### 9.3 Good versus bad

```csharp
// Good: generator output, mapped and restored
#line (12, 5) - (12, 18) 20 "menu.ui"
        node.Text = model.Title;
#line default

// Bad: hand-written code lying about its location
#line 1 "Other.cs"
public void M() { }
```

## 10. Related features

- [../12/interceptors-preview.md](../12/interceptors-preview.md): another generator-only feature.

## 11. References

- [Preprocessor directives: #line](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/preprocessor-directives#error-and-warning-information)
- [Proposal: Enhanced #line directives](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-10.0/enhanced-line-directives)
