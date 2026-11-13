---
id: cs15-union-types
csharp: 15
tfm: net11.0
status: stable
needs-runtime: true
tags: [union, discriminated-union, pattern-matching, exhaustiveness, result-type]
source:
  - https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-15
  - https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-15.0/unions
---
# Union types (C# 15)

`union` declares a type whose value is exactly one of several **case types**.
The case types need **no common base type**: you can combine unrelated or foreign types.

## Use when
- Result types: success or one of several failures (load, parse, purchase, lookup).
- A parameter or field is "either A or B" (`Node3D` target or `Vector3` position).
- Messages or events where the variants are existing, unrelated types.
- Prefer a closed hierarchy instead when you own all variants and want shared base members
  (see [closed-hierarchies.md](closed-hierarchies.md)).

## Syntax
```csharp
public record class Cat(string Name);
public record class Dog(string Name);
public record class Bird(string Name);

public union Pet(Cat, Dog, Bird);

Pet pet = new Dog("Rex");          // implicit conversion from each case type

string name = pet switch           // compiler checks exhaustiveness
{
    Dog d  => d.Name,
    Cat c  => c.Name,
    Bird b => b.Name,
};
```

Generic unions and members are allowed:
```csharp
public union OneOrMore<T>(T, IEnumerable<T>)
{
    public IEnumerable<T> AsEnumerable() => Value switch
    {
        IEnumerable<T> list => list,
        T value => [value],
    };
}
```

## How it is lowered (important for performance)
- A `union` declaration becomes a **struct** with one field: `public object? Value { get; }`,
  one public constructor per case type, and an implementation of `IUnion`.
- **Value-type case types are boxed** on entry.
- `default(MyUnion).Value` is `null`. Exhaustiveness can still warn about unhandled `null`.
- Unions cannot declare instance fields or auto-properties; they may implement interfaces.
- Custom (hand-written) unions: mark a class or struct with `[Union]` (`System.Runtime.CompilerServices`),
  provide constructors per case type and an `object? Value`. Optional non-boxing members
  `bool HasValue` and `bool TryGetValue(out T value)` (one per case type) are preferred by pattern matching.

## Library and framework support (.NET 11)
- System.Text.Json serializes and deserializes C# unions.
- ASP.NET Core Minimal APIs and MVC accept unions in JSON request and response bodies; OpenAPI uses `anyOf`.
  Route, query, header and form binding are not covered.

## Pitfalls
- Do not use unions with value-type cases in hot paths (per entity, per frame); each assignment boxes.
- A union is not a `record`; `record union` is not supported.
- Godot: unions are not Variant-compatible. Do not use them for `[Export]`, signal parameters or `Variant`.

## Example (game)
```csharp
public sealed record SaveGame(int Version, PlayerData Player);
public sealed record SaveNotFound(string Slot);
public sealed record SaveCorrupted(string Slot, string Reason);
public union LoadResult(SaveGame, SaveNotFound, SaveCorrupted);

string msg = Load("slot1") switch
{
    SaveGame g      => Start(g),
    SaveNotFound n  => $"No save in {n.Slot}",
    SaveCorrupted c => $"Save corrupted: {c.Reason}",
};

public union AttackTarget(Node3D, Vector3);   // Vector3 is boxed: fine per order, not per frame
```
