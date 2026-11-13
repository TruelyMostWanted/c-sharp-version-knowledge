---
id: cs10-record-structs
csharp: 10
tfm: net6.0
status: stable
tags: [record, struct, value-equality, with]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/record
modernizes: hand-written Equals/GetHashCode/ToString on structs
---
# Record structs (C# 10)

Value-type records: the compiler generates value equality, `ToString`, deconstruction and `with` support.

```csharp
public readonly record struct GridPos(int X, int Y);    // immutable: init-only properties
public record struct MutablePoint(float X, float Y);     // positional properties are mutable

var p = new GridPos(3, 4);
var q = p with { Y = 5 };
bool same = p == new GridPos(3, 4);   // true
```

## Rules
- `record struct` positional properties are **read-write**; `readonly record struct` makes them init-only.
- `record class` (or plain `record`) remains the reference-type form.

## Use when
- Small value types used as keys, coordinates, IDs, messages (`readonly record struct EntityId(int Value)`).

## Pitfalls
- Generated equality compares all fields; avoid for large structs in hot equality checks, or write a custom `Equals`.
