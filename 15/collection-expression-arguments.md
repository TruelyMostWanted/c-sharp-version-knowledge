---
id: cs15-collection-expression-arguments
csharp: 15
tfm: net11.0
status: stable
tags: [collection-expressions, capacity, comparer, with]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-15
---
# Collection expression arguments (C# 15)

A `with(...)` element as the **first** element of a collection expression passes arguments
to the collection's constructor or factory method.

## Use when
- You know the final size up front (pools, buffers, path lists) and want `capacity`.
- A set or collection needs a comparer (case-insensitive IDs, commands, tags).

## Syntax
```csharp
string[] values = ["one", "two", "three"];

List<string> names = [with(capacity: values.Length * 2), .. values];

HashSet<string> set = [with(StringComparer.OrdinalIgnoreCase), "Hello", "HELLO", "hello"];
// set.Count == 1
```

## Rules
- `with(...)` must be the first element.
- Arguments bind to the target collection's constructor or factory method like a normal call (named arguments allowed).

## Pitfalls
- Without `with(...)` behaviour is unchanged (C# 12 collection expressions).
- Prefer this over `new List<T>(n) { ... }` when the project is on C# 15.

## Example (game)
```csharp
private List<Projectile> _active = [];
public override void _Ready() => _active = [with(capacity: MaxProjectiles)];

HashSet<string> cheats = [with(StringComparer.OrdinalIgnoreCase), "godmode", "noclip"];
```
