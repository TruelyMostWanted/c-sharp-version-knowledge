---
id: cs11-list-patterns
csharp: 11
tfm: net7.0
status: stable
tags: [pattern-matching, list-pattern, slice, arrays, spans]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/patterns#list-patterns
modernizes: manual Length checks plus index access
---
# List patterns (C# 11)

Match arrays, lists and spans by shape. `..` is the slice pattern (zero or more elements), optionally captured.

```csharp
int[] numbers = [1, 2, 3];
bool a = numbers is [1, 2, 3];          // true
bool b = numbers is [_, > 1, ..];       // true
if (numbers is [var first, .. var rest]) { }

// Command parsing (debug console)
string result = args switch
{
    []                         => "no command",
    ["help"]                   => Help(),
    ["spawn", var name]        => Spawn(name, 1),
    ["spawn", var name, var n] => Spawn(name, int.Parse(n)),
    _                          => "unknown",
};
```

## Rules
- Works on types that are countable (`Length` or `Count`) and indexable; slices additionally need a range indexer or `Slice`.
- At most one slice pattern per list pattern.
