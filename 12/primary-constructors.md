---
id: cs12-primary-constructors
csharp: 12
tfm: net8.0
status: stable
analyzer: IDE0290
tags: [constructor, primary-constructor, dependency-injection]
source: https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/instance-constructors#primary-constructors
modernizes: constructors that only copy parameters into private fields
---
# Primary constructors for classes and structs (C# 12)

> **Abstract.** Any `class` or `struct` (not only records) can declare parameters directly after its name:
> `class OrderService(IOrderRepository repo, ILogger log)`. The parameters are in scope in the **whole type body**.
> Used in members, they are captured as hidden fields; used only in initializers, they cost nothing.
> Unlike records, **no public properties** are generated, and captured parameters are **mutable**.

| | |
|---|---|
| Introduced | C# 12 · .NET 8 (`net8.0`) |
| Status | Stable |
| Kind | Syntax + semantics |
| Runtime support needed | No |
| Audience | Everyday code (services, DI, small types) |
| Modernize category | CASE (IDE0290); never on Godot `GodotObject` scripts |

## 1. Motivation

Dependency-injected services repeat every dependency three times:

```csharp
public sealed class OrderService
{
    private readonly IOrderRepository _repo;           // 1
    public OrderService(IOrderRepository repo)          // 2
    {
        _repo = repo;                                   // 3
    }
}
```

Records (C# 9) had positional parameters, but they generate public properties and value equality, which services do not want.

## 2. The feature explained

### 2.1 Syntax

```csharp
public class C(int a, string b) : Base(a)
{
    private readonly string _b = b;      // used in an initializer
    public int Twice => a * 2;           // captured
}

public struct Point(double x, double y)
{
    public double Length => Math.Sqrt(x * x + y * y);
}
```

### 2.2 Semantics and rules

1. Parameters are in scope in member bodies, field/property initializers and the base-class argument list.
2. If a parameter is used only in initializers or the base call, **no field** is generated.
3. If used in a member body, the compiler captures it in a hidden, **mutable** field (unspeakable name).
4. Every explicitly declared constructor must chain to the primary constructor via `: this(...)`.
5. A class with a primary constructor gets **no** implicit parameterless constructor.
6. A struct still has the implicit parameterless constructor (`new S()` / `default`), which zero-initializes everything, including captured parameters.
7. Parameters are not members: `this.repo` does not exist; they are not accessible from derived types.
8. Attributes for the primary constructor itself are written on the type declaration with the `method:` target:
   `[method: SetsRequiredMembers] public class C(int a) { ... }`.
9. Warnings: CS9124 (parameter captured **and** used to initialize a field → stored twice), CS9107 (parameter captured and also passed to base).

## 3. Code examples

### 3.1 Minimal example

```csharp
public class OrderService(IOrderRepository repo, ILogger<OrderService> log)
{
    public Order Get(int id)
    {
        log.LogDebug("Loading {Id}", id);
        return repo.Find(id);
    }
}
```

### 3.2 Extended examples

Readonly by explicit field (recommended when immutability matters):

```csharp
public sealed class SaveService(IFileSystem fs, ISerializer serializer)
{
    private readonly IFileSystem _fs = fs;
    private readonly ISerializer _serializer = serializer;

    public void Save(string path, SaveGame game) => _fs.WriteAllBytes(path, _serializer.Serialize(game));
}
```

Validation in initializers:

```csharp
public sealed class Pool<T>(int capacity) where T : new()
{
    private readonly T[] _items = capacity > 0
        ? new T[capacity]
        : throw new ArgumentOutOfRangeException(nameof(capacity));
}
```

Passing to a base class:

```csharp
public abstract class Unit(string id, int maxHp)
{
    public string Id { get; } = id;
    public int MaxHp { get; } = maxHp;
}

public sealed class Archer(string id) : Unit(id, maxHp: 40);
```

Additional constructors chain to the primary one:

```csharp
public sealed class Timer(float duration, bool loop)
{
    public Timer(float duration) : this(duration, loop: false) { }
    public float Duration { get; } = duration;
    public bool Loop { get; } = loop;
}
```

Struct with primary constructor:

```csharp
public readonly struct Range1D(float min, float max)
{
    public float Min { get; } = min;
    public float Max { get; } = max;
    public bool Contains(float v) => v >= Min && v <= Max;
}
```

## 4. Before this feature

```csharp
public sealed class OrderService
{
    private readonly IOrderRepository _repo;
    private readonly ILogger<OrderService> _log;

    public OrderService(IOrderRepository repo, ILogger<OrderService> log)
    {
        _repo = repo;
        _log = log;
    }
}
```

## 5. Under the hood

```csharp
// public class OrderService(IOrderRepository repo, ILogger<OrderService> log) { ... uses repo, log ... }
public class OrderService
{
    [CompilerGenerated] private IOrderRepository <repo>P;      // mutable, not readonly
    [CompilerGenerated] private ILogger<OrderService> <log>P;

    public OrderService(IOrderRepository repo, ILogger<OrderService> log)
    {
        <repo>P = repo;
        <log>P = log;
        base..ctor();
    }

    public Order Get(int id) { <log>P.LogDebug(...); return <repo>P.Find(id); }
}
```

When a parameter is used only in an initializer (`private readonly IFileSystem _fs = fs;`), no `<fs>P` field exists;
the value goes straight into `_fs`.

## 6. Performance and allocation

Same as hand-written fields. Watch for double storage (CS9124): a captured parameter plus a field initialized from it doubles the memory per instance.

## 7. Pitfalls

- **Mutable capture.** `repo = null;` inside a method compiles and changes the captured state. Use explicit `readonly` fields when this matters.
- **Double storage** if you both capture a parameter and assign it to a field/property.
- Names: parameters are lower-case and look like locals; in long classes readers may not realize they are state.
- Parameters shadowed by members with the same name (`Id` vs `id`) can cause confusion; the compiler prefers the member in some contexts.
- **Godot:** do not add a primary constructor to script classes deriving from `GodotObject` (`Node`, `Resource`, ...).
  The engine instantiates scripts through a **parameterless** constructor, which a primary-constructor class does not have.

## 8. Concerns and limitations

- Public API: changing a public class to a primary constructor is source-compatible only if the constructor shape stays the same.
- DI containers work unchanged (they see a normal constructor).
- Serialization: serializers bind to the primary constructor like any other constructor; but no properties are generated.

## 9. When to use: clean-code guidance

### 9.1 Use when
- Services and handlers with injected dependencies.
- Small types whose constructor only initializes fields or properties.

### 9.2 Avoid when
- The type needs constructor logic beyond initializers.
- Engine-instantiated classes (Godot nodes, resources, editor tools).
- Large classes where hidden captured state hurts readability.

### 9.3 Good versus bad

```csharp
// Good
public sealed class DamageCalculator(IRandom rng, BalanceTable table)
{
    public int Roll(int baseDmg) => (int)(baseDmg * table.Multiplier * rng.Range(0.9f, 1.1f));
}

// Bad: Godot script with a primary constructor (engine cannot create it)
public partial class Enemy(int hp) : CharacterBody2D { }
```

## 10. Related features

- [../10/record-structs.md](../10/record-structs.md): positional records generate properties.
- [../11/required-members.md](../11/required-members.md): initializer-based alternative.
- [../14/partial-events-and-constructors.md](../14/partial-events-and-constructors.md)

## 11. References

- [Primary constructors](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/instance-constructors#primary-constructors)
- [Tutorial: Explore primary constructors](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/tutorials/primary-constructors)
- [Proposal: Primary constructors](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-12.0/primary-constructors)
- [IDE0290](https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/style-rules/ide0290)
