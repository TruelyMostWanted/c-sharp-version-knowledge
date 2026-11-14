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

> **Abstract.** `record struct` and `readonly record struct` declare **value-type records**. The compiler
> synthesizes typed equality (`IEquatable<T>`, `==`, `!=`), `GetHashCode`, a readable `ToString`, `Deconstruct`
> and `with` support. `readonly record struct` is the go-to form for small immutable values: IDs, coordinates,
> keys, messages.

| | |
|---|---|
| Introduced | C# 10 · .NET 6 (`net6.0`) |
| Status | Stable |
| Kind | Syntax + synthesized members |
| Runtime support needed | No |
| Audience | Everyday code |
| Modernize category | CASE (equality semantics may change) |

## 1. Motivation

A correct value struct needs a lot of boilerplate: `Equals(T)`, `Equals(object)`, `GetHashCode`, `==`, `!=`,
`IEquatable<T>`, often `ToString` and `Deconstruct`. Forgetting `IEquatable<T>` causes boxing in collections,
and the default `ValueType.Equals` uses **reflection** for structs with reference-type fields. C# 9 records solved
this only for reference types.

## 2. The feature explained

### 2.1 Syntax

```csharp
public readonly record struct GridPos(int X, int Y);   // positional, immutable
public record struct MutablePoint(float X, float Y);    // positional, mutable
public record struct Stats { public int Hp { get; init; } public int Mp { get; init; } }   // nominal
```

### 2.2 Semantics and rules

1. Positional parameters become **public properties**: read-write (`get; set;`) in `record struct`,
   init-only (`get; init;`) in `readonly record struct`.
2. Synthesized: a primary constructor, `Equals(T)`, `Equals(object)`, `GetHashCode()`, `operator ==`/`!=`,
   `IEquatable<T>`, `ToString()`, `PrintMembers`, `Deconstruct`.
3. Equality compares all **instance fields** (including backing fields) with `EqualityComparer<TField>.Default`.
4. `with` creates a copy and sets the listed members.
5. A parameterless `new GridPos()` / `default` is always available and zero-initializes everything.
6. `record` alone means `record class` (reference type).
7. Any synthesized member can be replaced by declaring it yourself (for example a custom `Equals` or `ToString`).

## 3. Code examples

### 3.1 Minimal example

```csharp
public readonly record struct GridPos(int X, int Y);

var p = new GridPos(3, 4);
var q = p with { Y = 5 };
bool same = p == new GridPos(3, 4);   // true
var (x, y) = p;                       // Deconstruct
Console.WriteLine(p);                 // "GridPos { X = 3, Y = 4 }"
```

### 3.2 Extended examples

Strongly typed IDs (avoid mixing up `int`s):

```csharp
public readonly record struct EntityId(int Value)
{
    public static readonly EntityId None = new(-1);
    public override string ToString() => $"#{Value}";
}

public readonly record struct ItemId(int Value);

void Give(EntityId to, ItemId item) { }   // Give(item, to) no longer compiles
```

Dictionary keys and set elements:

```csharp
var occupied = new HashSet<GridPos>();
occupied.Add(new GridPos(1, 2));
bool taken = occupied.Contains(new GridPos(1, 2));   // true, no boxing (IEquatable<GridPos>)
```

Validation in a positional record struct:

```csharp
public readonly record struct Percent(float Value)
{
    public float Value { get; init; } = Value is >= 0 and <= 100
        ? Value
        : throw new ArgumentOutOfRangeException(nameof(Value));
}
```

Messages / events passed through channels:

```csharp
public readonly record struct DamageDealt(EntityId Source, EntityId Target, int Amount);
channel.Writer.TryWrite(new DamageDealt(attacker, victim, 12));
```

## 4. Before this feature

```csharp
public readonly struct GridPos : IEquatable<GridPos>
{
    public GridPos(int x, int y) { X = x; Y = y; }
    public int X { get; }
    public int Y { get; }

    public bool Equals(GridPos other) => X == other.X && Y == other.Y;
    public override bool Equals(object? obj) => obj is GridPos other && Equals(other);
    public override int GetHashCode() => HashCode.Combine(X, Y);
    public static bool operator ==(GridPos a, GridPos b) => a.Equals(b);
    public static bool operator !=(GridPos a, GridPos b) => !a.Equals(b);
    public override string ToString() => $"GridPos {{ X = {X}, Y = {Y} }}";
    public void Deconstruct(out int x, out int y) { x = X; y = Y; }
}
```

## 5. Under the hood

`readonly record struct GridPos(int X, int Y)` is roughly:

```csharp
public readonly struct GridPos : IEquatable<GridPos>
{
    public GridPos(int X, int Y) { this.X = X; this.Y = Y; }
    public int X { get; init; }
    public int Y { get; init; }

    public bool Equals(GridPos other) =>
        EqualityComparer<int>.Default.Equals(<X>k__BackingField, other.<X>k__BackingField) &&
        EqualityComparer<int>.Default.Equals(<Y>k__BackingField, other.<Y>k__BackingField);
    public override bool Equals(object obj) => obj is GridPos g && Equals(g);
    public override int GetHashCode() =>
        EqualityComparer<int>.Default.GetHashCode(<X>k__BackingField) * -1521134295 +
        EqualityComparer<int>.Default.GetHashCode(<Y>k__BackingField);
    public static bool operator ==(GridPos l, GridPos r) => l.Equals(r);
    public static bool operator !=(GridPos l, GridPos r) => !(l == r);
    public override string ToString() { /* "GridPos { " + PrintMembers + " }" via StringBuilder */ }
    private bool PrintMembers(StringBuilder builder) { ... }
    public void Deconstruct(out int X, out int Y) { X = this.X; Y = this.Y; }
}
```

`p with { Y = 5 }` on a struct is just a copy plus a setter call: `var tmp = p; tmp.Y = 5;` (the init accessor is allowed here).
Unlike record classes there is no `EqualityContract` and no `<Clone>$` method.

## 6. Performance and allocation

- No heap allocation; equality is strongly typed (no boxing in `Dictionary`/`HashSet`).
- `EqualityComparer<T>.Default` calls are devirtualized and inlined by the JIT for primitives.
- `ToString()` allocates (StringBuilder); keep it out of hot paths.
- Large record structs are copied by value on every pass and assignment; keep them small (≤ 16–32 bytes is a common guideline) or pass with `in`.

## 7. Pitfalls

- `record struct` (without `readonly`) has **mutable** public properties: surprising for a "record". Prefer `readonly record struct`.
- Equality compares **all** fields, including arrays by reference. `record struct Path(int[] Points)` is equal only if both refer to the same array.
- `default(GridPos)` bypasses validation in constructors or property initializers.
- Floating-point members: `NaN != NaN` with `==` on `float`, but `EqualityComparer<float>.Default.Equals(NaN, NaN)` is `true`; record equality follows the latter.

## 8. Concerns and limitations

- Changing an existing `struct` to `record struct` changes `Equals`, `==` availability and `ToString` output: review serializers and tests (CASE).
- Godot: record structs are not Variant-compatible. Do not use them as `[Export]` fields or signal parameters; keep them in the C# logic layer.
- Serialization: System.Text.Json handles positional record structs via the constructor.

## 9. When to use: clean-code guidance

### 9.1 Use when
- Small immutable values: IDs, coordinates, ranges, money, keys, messages.
- You want value equality and readable `ToString` without boilerplate.

### 9.2 Avoid when
- The type is large or contains collections (equality semantics surprise).
- Identity matters (entities with lifecycle): use a class.
- Per-frame hot loops that only need raw fields: a plain struct with public fields may be simpler (SoA/AoS data).

### 9.3 Good versus bad

```csharp
// Good
public readonly record struct Money(decimal Amount, string Currency);

// Bad: mutable, contains a reference-type collection, compared by reference
public record struct Inventory(List<Item> Items, int Gold);
```

## 10. Related features

- [struct-improvements.md](struct-improvements.md): parameterless constructors and `with` for all structs.
- [record-sealed-tostring.md](record-sealed-tostring.md)
- [../12/primary-constructors.md](../12/primary-constructors.md): primary constructors for non-record types.

## 11. References

- [Records (C# reference)](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/record)
- [Proposal: Record structs](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-10.0/record-structs)
