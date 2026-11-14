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
modernizes: object parameters, marker interfaces, OneOf<...> packages, result-or-error tuples
---
# Union types (C# 15)

> **Abstract.** `union` declares a type whose value is exactly one of several **case types**:
> `public union Pet(Cat, Dog, Bird);`. The case types need **no common base type**, so unrelated or foreign types can be
> combined. Each case type converts implicitly to the union, and `switch` over a union is checked for exhaustiveness.
> A union is lowered to a struct holding an `object? Value`, so **value-type cases are boxed**.

| | |
|---|---|
| Introduced | C# 15 · .NET 11 (`net11.0`) |
| Status | Stable |
| Kind | Language (new type kind) + runtime/library support |
| Runtime support needed | Yes: `IUnion`, `[Union]` and library support in .NET 11 |
| Audience | Everyday code |
| Modernize category | CASE; never in hot paths with value-type cases, never across the Godot boundary |

## 1. Motivation

"This value is either A or B" had no first-class representation in C#:

- `object` loses all type information and needs casts;
- a marker interface or base class requires owning all the types;
- tuples like `(T? Value, Error? Error)` allow invalid states (both set or both null);
- libraries such as `OneOf<T0, T1, T2>` work but are not understood by the compiler's exhaustiveness checks or serializers.

## 2. The feature explained

### 2.1 Syntax

```csharp
public union Pet(Cat, Dog, Bird);

public union OneOrMore<T>(T, IEnumerable<T>)       // generic union with members
{
    public IEnumerable<T> AsEnumerable() => Value switch
    {
        IEnumerable<T> list => list,
        T value => [value],
    };
}
```

### 2.2 Semantics and rules

1. A union lists its case types in parentheses; any types are allowed (classes, structs, interfaces, generic parameters).
2. Implicit conversion exists from each case type to the union.
3. Pattern matching (`is`, `switch`) on a union tests its **contained value**; `switch` expressions are checked for exhaustiveness over the case types.
4. A union is not a `record`; `record union` is not supported. Unions cannot declare instance fields or auto-properties; they may declare methods and computed members and implement interfaces.
5. `default(MyUnion).Value` is `null`. Exhaustiveness may therefore still warn about an unhandled `null`.
6. Custom (hand-written) unions: mark a class or struct with `[Union]` (`System.Runtime.CompilerServices`), provide one constructor per case type and an `object? Value` property.
   Optional non-boxing members `bool HasValue` and `bool TryGetValue(out T value)` (one per case type) are preferred by pattern matching.

## 3. Code examples

### 3.1 Minimal example

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

### 3.2 Extended examples

Result type for loading a save game:

```csharp
public sealed record SaveGame(int Version, PlayerData Player);
public sealed record SaveNotFound(string Slot);
public sealed record SaveCorrupted(string Slot, string Reason);
public union LoadResult(SaveGame, SaveNotFound, SaveCorrupted);

LoadResult Load(string slot)
{
    if (!File.Exists(PathFor(slot))) return new SaveNotFound(slot);
    try { return Parse(File.ReadAllText(PathFor(slot))); }
    catch (JsonException e) { return new SaveCorrupted(slot, e.Message); }
}

string msg = Load("slot1") switch
{
    SaveGame g      => Start(g),
    SaveNotFound n  => $"No save in {n.Slot}",
    SaveCorrupted c => $"Save corrupted: {c.Reason}",
};
```

Either a node or a position (unrelated types, one of them foreign):

```csharp
public union AttackTarget(Node3D, Vector3);   // Vector3 is boxed: fine per order, not per frame

Vector3 Aim(AttackTarget t) => t switch
{
    Node3D n  => n.GlobalPosition,
    Vector3 p => p,
};
```

Generic result:

```csharp
public sealed record Error(string Code, string Message);
public union Result<T>(T, Error);

Result<int> ParsePort(string s) =>
    int.TryParse(s, out var p) && p is > 0 and < 65536 ? p : new Error("port", $"invalid port '{s}'");
```

Union members:

```csharp
public union Shape(Circle, Rect)
{
    public float Area => Value switch
    {
        Circle c => MathF.PI * c.R * c.R,
        Rect r   => r.W * r.H,
        _        => 0,          // default(Shape) has Value == null
    };
}
```

JSON round-trip (System.Text.Json in .NET 11):

```csharp
string json = JsonSerializer.Serialize<LoadResult>(new SaveNotFound("slot2"));
LoadResult back = JsonSerializer.Deserialize<LoadResult>(json)!;
```

## 4. Before this feature

```csharp
// Marker interface (requires owning every type)
public interface ILoadResult;
public sealed record SaveGame(...) : ILoadResult;
...
var msg = result switch { SaveGame g => ..., SaveNotFound n => ..., _ => throw new UnreachableException() };

// object (no safety)
object target = node;   // or a Vector3, boxed, any type accepted

// OneOf package
OneOf<SaveGame, SaveNotFound, SaveCorrupted> r = ...;
r.Switch(g => ..., n => ..., c => ...);
```

## 5. Under the hood

A `union` declaration becomes a **struct** roughly like:

```csharp
[Union]
public struct Pet : IUnion
{
    public Pet(Cat value)  => Value = value;
    public Pet(Dog value)  => Value = value;
    public Pet(Bird value) => Value = value;
    public object? Value { get; }
    // implicit conversions from Cat, Dog, Bird
}
```

- One field holding `object? Value`; one public constructor per case type; implements `IUnion`.
- **Value-type case types are boxed** on entry.
- `pet is Dog d` lowers to a type test on `pet.Value` (or to `TryGetValue(out Dog)` if a custom union provides it).

## 6. Performance and allocation

- The union itself is a small struct (one reference). Reference-type cases: no extra allocation.
- Value-type cases (`int`, `Vector3`, struct records) are **boxed** on every assignment: one heap allocation each.
- Type tests are cheap (`isinst`). For allocation-free custom unions, implement `TryGetValue(out T)` members with `[Union]`.

## 7. Pitfalls

- Do not use unions with value-type cases in hot paths (per entity, per frame); each assignment boxes.
- `default(MyUnion)` holds `null`; uninitialized fields or array elements of a union type are "none of the cases".
- Overlapping case types (`union U(Animal, Dog)`) make `switch` arm order matter; avoid overlapping cases.
- A union is not a `record`: no value equality or `with` is synthesized beyond what the struct provides.

## 8. Concerns and limitations

- Library and framework support (.NET 11): System.Text.Json serializes and deserializes C# unions; ASP.NET Core Minimal APIs and MVC accept unions in JSON request and response bodies, with OpenAPI `anyOf`. Route, query, header and form binding are not covered.
- API evolution: adding a case type breaks exhaustive switches of consumers (warnings).
- Godot: unions are not Variant-compatible. Do not use them for `[Export]`, signal parameters or `Variant`.

## 9. When to use: clean-code guidance

### 9.1 Use when
- Result types: success or one of several failures (load, parse, purchase, lookup).
- A parameter or field is "either A or B" with unrelated or foreign types (`Node3D` target or `Vector3` position).
- Messages or events where the variants are existing, unrelated types.

### 9.2 Avoid when
- You own all variants and want shared base members: use a [closed hierarchy](closed-hierarchies.md).
- Hot paths with value-type cases.
- Data crossing into the engine (Godot) or other non-.NET boundaries.

### 9.3 Good versus bad

```csharp
// Good
public union PurchaseResult(Purchased, NotEnoughGold, ItemSoldOut);

// Bad: boxing every frame
public union FrameInput(Vector2, float, bool);
_input = new Vector2(x, y);   // allocates per frame
```

## 10. Related features

- [closed-hierarchies.md](closed-hierarchies.md)
- [../11/list-patterns.md](../11/list-patterns.md), [../10/extended-property-patterns.md](../10/extended-property-patterns.md)

## 11. References

- [What's new in C# 15](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-15)
- [Proposal: Unions](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-15.0/unions)
