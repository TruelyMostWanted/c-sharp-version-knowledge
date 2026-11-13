---
id: cs11-raw-string-literals
csharp: 11
tfm: net7.0
status: stable
tags: [string, raw-string, json, multiline, interpolation]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/tokens/raw-string
modernizes: verbatim strings @"..." with doubled quotes, escaped JSON/XML/SQL/shader text
---
# Raw string literals (C# 11)

Start and end with at least three `"`. No escaping inside. Multi-line form: delimiters on their own lines;
the indentation of the closing `"""` is removed from every content line.

```csharp
string json = """
    {
      "name": "Ann",
      "tags": ["a", "b"]
    }
    """;

// Interpolated: number of $ = number of braces needed for a hole
string shader = $$"""
    shader_type canvas_item;
    uniform float speed = {{speed}};
    void fragment() { COLOR.a *= 0.5; }
    """;
```

## Rules
- Content containing `"""` needs a longer delimiter (`""""`).
- Multi-line: content lines must start with (at least) the closing delimiter's whitespace.
- With `$$"""`, single `{ }` are literal text and `{{expr}}` is a hole. Useful for JSON, GDShader, C-like code.

## Use when
- JSON, SQL, regex, GDShader or code templates in source, test fixtures, source generators.
