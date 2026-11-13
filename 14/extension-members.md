---
id: cs14-extension-members
csharp: 14
tfm: net10.0
status: stable
tags: [extension, extension-property, static-extension, operator]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-14
related: ../15/extension-indexers.md
---
# Extension members (C# 14)

`extension` blocks inside a static class declare extension **properties**, **static** extension members
and **operators**, in addition to extension methods.

## Use when
- A type you don't own needs a computed property (`list.IsEmpty`) or a static factory or helper (`Vector2I.Manhattan(a, b)`).
- Grouping all extensions for one receiver type in one block.

## Syntax
```csharp
public static class Enumerable
{
    extension<TSource>(IEnumerable<TSource> source)      // named receiver: instance-style members
    {
        public bool IsEmpty => !source.Any();
        public IEnumerable<TSource> Where(Func<TSource, bool> predicate) { ... }
    }

    extension<TSource>(IEnumerable<TSource>)             // receiver type only: static members
    {
        public static IEnumerable<TSource> Identity => Enumerable.Empty<TSource>();
        public static IEnumerable<TSource> operator +(IEnumerable<TSource> left, IEnumerable<TSource> right)
            => left.Concat(right);
    }
}
```

## Rules
- Named receiver: call as instance member (`sequence.IsEmpty`).
- Receiver type only: call as static member (`IEnumerable<int>.Identity`).
- User-defined operators can be implemented as static extension members.
- Classic `this`-parameter extension methods keep working.
- Indexers in extension blocks come with C# 15.

## Pitfalls
- Extension properties cannot hold state (no backing field on the receiver).
- Static extension properties that build collections allocate on every access; cache them.

## Example (Godot)
```csharp
public static class VectorExtensions
{
    extension(Vector2 v)
    {
        public bool IsNearlyZero => v.LengthSquared() < 1e-6f;
    }
    extension(Vector2I)
    {
        public static int Manhattan(Vector2I a, Vector2I b) => Math.Abs(a.X - b.X) + Math.Abs(a.Y - b.Y);
    }
}

if (velocity.IsNearlyZero) PlayIdle();
int cost = Vector2I.Manhattan(from, to);
```
