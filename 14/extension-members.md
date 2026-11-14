---
id: cs14-extension-members
csharp: 14
tfm: net10.0
status: stable
tags: [extension, extension-property, static-extension, operator]
source: https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/extension-methods
related: ../15/extension-indexers.md
modernizes: GetX() extension methods that are conceptually properties; static helper classes for a type
---
# Extension members (C# 14)

> **Abstract.** `extension` blocks inside a static class declare **extension properties**, **static extension
> members** (methods, properties, operators) and classic extension methods for a receiver type. Instance-style members
> use a named receiver (`extension(Vector2 v)`), static members use a receiver type only (`extension(Vector2I)`).
> Existing `this`-parameter extension methods keep working and are binary compatible with the new form.

| | |
|---|---|
| Introduced | C# 14 · .NET 10 (`net10.0`) |
| Status | Stable |
| Kind | Syntax + semantics (member lookup) |
| Runtime support needed | No |
| Audience | Everyday code, library authors |
| Modernize category | PREFER for new code; do not rewrite existing `this` extension methods |

## 1. Motivation

Extension methods (C# 3) only covered instance **methods**. Common needs had awkward workarounds:

- computed properties had to be methods (`list.IsEmpty()`, `GetLength()`),
- "static extensions" (`Vector2I.Manhattan(a, b)`) had to live in an unrelated helper class (`VectorUtil.Manhattan`),
- operators for foreign types were impossible.

## 2. The feature explained

### 2.1 Syntax

```csharp
public static class MyExtensions
{
    extension<T>(IEnumerable<T> source)          // named receiver: instance-style members
    {
        public bool IsEmpty => !source.Any();
        public IEnumerable<T> WhereNotNull() => source.Where(x => x is not null);
    }

    extension<T>(IEnumerable<T>)                 // receiver type only: static members
    {
        public static IEnumerable<T> Empty => [];
        public static IEnumerable<T> operator +(IEnumerable<T> a, IEnumerable<T> b) => a.Concat(b);
    }
}
```

### 2.2 Semantics and rules

1. Extension blocks live in **non-generic, top-level static classes** (same as classic extension methods).
2. Named receiver → members are called like instance members (`seq.IsEmpty`); the receiver name is in scope.
3. Receiver type only → members are called like static members of the receiver type (`IEnumerable<int>.Empty`).
4. Members: methods, properties (get/set), operators (static), and, from C# 15, indexers. No fields, no auto-properties with state, no events, no constructors.
5. Receiver modifiers: `ref`, `ref readonly`, `in` for value-type receivers (`extension(ref Counter c)`), enabling mutation of structs.
6. Generic extension blocks: type parameters on the block (`extension<T>(...)`) and on members.
7. Classic `static R M(this T x)` methods and members in extension blocks can coexist and are interchangeable for callers.
8. Lookup: real members on the type always win over extensions.

## 3. Code examples

### 3.1 Minimal example

```csharp
public static class StringExtensions
{
    extension(string s)
    {
        public bool IsBlank => string.IsNullOrWhiteSpace(s);
    }
}

if (input.IsBlank) return;
```

### 3.2 Extended examples

Godot vector helpers (instance and static):

```csharp
public static class VectorExtensions
{
    extension(Vector2 v)
    {
        public bool IsNearlyZero => v.LengthSquared() < 1e-6f;
        public Vector2 WithY(float y) => new(v.X, y);
    }

    extension(Vector2I)
    {
        public static int Manhattan(Vector2I a, Vector2I b) => Math.Abs(a.X - b.X) + Math.Abs(a.Y - b.Y);
    }
}

if (velocity.IsNearlyZero) PlayIdle();
int cost = Vector2I.Manhattan(from, to);
```

Extension property with a setter on a mutable receiver:

```csharp
public static class TransformExtensions
{
    extension(Node2D node)
    {
        public float PositionX
        {
            get => node.Position.X;
            set => node.Position = node.Position with { X = value };
        }
    }
}

enemy.PositionX += 10f;
```

`ref` receiver for structs:

```csharp
public struct Counter { public int Value; }

public static class CounterExtensions
{
    extension(ref Counter c)
    {
        public void Increment() => c.Value++;   // mutates the caller's struct
    }
}

var counter = new Counter();
counter.Increment();   // counter.Value == 1
```

Operators for a type you don't own:

```csharp
public static class VersionExtensions
{
    extension(Version)
    {
        public static Version operator +(Version v, int builds)
            => new(v.Major, v.Minor, Math.Max(v.Build, 0) + builds);
    }
}

var next = new Version(1, 4, 2) + 1;   // 1.4.3
```

(Only for operators the type does not define itself; a real operator on the type always wins.)

Static factory next to the type:

```csharp
public static class TimeSpanExtensions
{
    extension(TimeSpan)
    {
        public static TimeSpan FromFrames(int frames, int fps = 60) => TimeSpan.FromSeconds(frames / (double)fps);
    }
}

var t = TimeSpan.FromFrames(30);
```

## 4. Before this feature

```csharp
public static class VectorUtil
{
    public static bool IsNearlyZero(this Vector2 v) => v.LengthSquared() < 1e-6f;   // method, not property
    public static int Manhattan(Vector2I a, Vector2I b) => ...;                       // VectorUtil.Manhattan(a, b)
}

if (velocity.IsNearlyZero()) PlayIdle();
int cost = VectorUtil.Manhattan(from, to);
```

## 5. Under the hood

Each extension member is compiled to an ordinary **static method** in the containing static class, with the receiver as
first parameter (for instance-style members). Properties become accessor methods:

```csharp
// extension(string s) { public bool IsBlank => ...; }
public static class StringExtensions
{
    [Extension]
    public static bool get_IsBlank(string s) => string.IsNullOrWhiteSpace(s);
    // plus compiler-generated nested metadata types describing the extension block (names are internal)
}
```

Static extension members become static methods without a receiver parameter. Because instance-style extension methods
compile to the same shape as classic `this` methods, switching between the two syntaxes is binary compatible.

## 6. Performance and allocation

- Same as static method calls; no allocation, inlinable.
- Extension properties recompute on every access (no backing storage). Static extension properties that build collections allocate per access; cache them.

## 7. Pitfalls

- Extension properties cannot hold state; there is no field on the receiver. Use a `ConditionalWeakTable` if you truly need attached state.
- Properties look cheap; do not hide expensive work (I/O, LINQ over large sets) behind them.
- Name clashes: if the receiver type later adds a real member with the same name, it silently wins.
- Value-type receivers without `ref` operate on a copy; setters then modify the copy.

## 8. Concerns and limitations

- Discoverability: readers may not know where `Vector2I.Manhattan` comes from; keep extensions in well-named classes and namespaces.
- Tooling: older analyzers or decompilers may display extension members as plain static methods.
- Godot: extension members are compile-time only and work on engine types; they are not visible to GDScript or the editor.

## 9. When to use: clean-code guidance

### 9.1 Use when
- A foreign type needs a computed property, a static factory or an operator.
- Grouping all extensions for one receiver in a block.

### 9.2 Avoid when
- You own the type (add the member directly).
- The "property" is expensive or has side effects (use a method).

### 9.3 Good versus bad

```csharp
// Good
extension(Rect2 r) { public Vector2 Center => r.Position + r.Size / 2; }

// Bad: hidden expensive work behind a property
extension(Node n) { public int DescendantCount => n.FindChildren("*").Count; }
```

## 10. Related features

- [../15/extension-indexers.md](../15/extension-indexers.md)
- [user-defined-compound-assignment.md](user-defined-compound-assignment.md)
- [implicit-span-conversions.md](implicit-span-conversions.md): span receivers.

## 11. References

- [Extension members](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/extension-methods)
- [extension keyword](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/extension)
- [Proposal: Extensions](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-14.0/extensions)
