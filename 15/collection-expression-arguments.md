---
id: cs15-collection-expression-arguments
csharp: 15
tfm: net11.0
status: stable
tags: [collection-expressions, capacity, comparer, with]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-15
related: ../12/collection-expressions.md
modernizes: new List<T>(n) or new HashSet<T>(comparer) followed by adds
---
# Collection expression arguments (C# 15)

> **Abstract.** A `with(...)` element as the **first** element of a collection expression passes arguments to the target
> collection's constructor or collection-builder method: `[with(capacity: n), .. items]`,
> `[with(StringComparer.OrdinalIgnoreCase), "a", "b"]`. It closes the last gap of C# 12 collection expressions:
> capacity and comparers.

| | |
|---|---|
| Introduced | C# 15 · .NET 11 (`net11.0`) |
| Status | Stable |
| Kind | Syntax |
| Runtime support needed | No |
| Audience | Everyday code |
| Modernize category | PREFER |

## 1. Motivation

C# 12 collection expressions could not express constructor arguments. Whenever a collection needed a comparer or an
initial capacity, code fell back to `new HashSet<string>(StringComparer.OrdinalIgnoreCase) { ... }` or `new List<T>(n)`
plus a loop, mixing two styles in the same codebase.

## 2. The feature explained

### 2.1 Syntax

```csharp
List<T>      a = [with(capacity: 64)];
HashSet<T>   b = [with(comparer), x, y];
Dictionary<K, V> c = [with(StringComparer.Ordinal)];
List<T>      d = [with(capacity: src.Length), .. src];
```

### 2.2 Semantics and rules

1. `with(...)` must be the **first** element; at most one.
2. Arguments bind to the target type's constructor (or to the collection-builder method's extra parameters) like a normal call: overload resolution, named and optional arguments.
3. Without `with(...)` behaviour is unchanged (C# 12 rules).
4. For targets without a matching constructor (arrays, spans), `with(...)` is an error.
5. Elements and spreads after `with(...)` are added as usual.

## 3. Code examples

### 3.1 Minimal example

```csharp
string[] values = ["one", "two", "three"];

List<string> names = [with(capacity: values.Length * 2), .. values];

HashSet<string> set = [with(StringComparer.OrdinalIgnoreCase), "Hello", "HELLO", "hello"];
// set.Count == 1
```

### 3.2 Extended examples

Pre-sized pool:

```csharp
private List<Projectile> _active = [];

public override void _Ready() => _active = [with(capacity: MaxProjectiles)];
```

Case-insensitive lookups:

```csharp
HashSet<string> cheats = [with(StringComparer.OrdinalIgnoreCase), "godmode", "noclip"];
Dictionary<string, Command> commands = [with(StringComparer.OrdinalIgnoreCase)];
```

Sorted set with custom ordering:

```csharp
SortedSet<Unit> byHp = [with(Comparer<Unit>.Create((a, b) => a.Hp.CompareTo(b.Hp))), .. units];
```

Named arguments:

```csharp
Dictionary<int, string> map = [with(capacity: 128, comparer: EqualityComparer<int>.Default)];
```

## 4. Before this feature

```csharp
var names = new List<string>(values.Length * 2);
names.AddRange(values);

var set = new HashSet<string>(StringComparer.OrdinalIgnoreCase) { "Hello", "HELLO", "hello" };
```

## 5. Under the hood

`[with(StringComparer.OrdinalIgnoreCase), "a", "b"]` targeting `HashSet<string>` lowers to:

```csharp
var tmp = new HashSet<string>(StringComparer.OrdinalIgnoreCase);
tmp.Add("a");
tmp.Add("b");
```

For collection-builder types, the `with` arguments are passed to the builder method before the span of elements.

## 6. Performance and allocation

- Capacity avoids repeated growth and copying for large collections.
- Otherwise identical to the hand-written constructor call.

## 7. Pitfalls

- `with(...)` not in first position is an error; so is using it with arrays or spans.
- Capacity is a hint; it does not limit the collection.
- Interface targets (`IEnumerable<T>`) let the compiler choose the concrete type; `with(...)` there is restricted (check docs) and the concrete type may not have the constructor you expect.

## 8. Concerns and limitations

- Requires C# 15; on C# 12–14 use `new T(args) { ... }`.
- Godot: works for .NET collections; Godot's `Array`/`Dictionary` constructors take different arguments.

## 9. When to use: clean-code guidance

### 9.1 Use when
- The final size is known up front (pools, buffers, path lists).
- A set or dictionary needs a comparer.

### 9.2 Avoid when
- Small collections where capacity does not matter: plain `[]` is clearer.

### 9.3 Good versus bad

```csharp
// Good
HashSet<string> tags = [with(StringComparer.OrdinalIgnoreCase), .. rawTags];

// Bad (on C# 15): mixing styles without need
var tags2 = new HashSet<string>(StringComparer.OrdinalIgnoreCase);
foreach (var t in rawTags) tags2.Add(t);
```

## 10. Related features

- [../12/collection-expressions.md](../12/collection-expressions.md)
- [../13/params-collections.md](../13/params-collections.md)

## 11. References

- [What's new in C# 15](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-15)
- [Collection expressions](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/collection-expressions)
