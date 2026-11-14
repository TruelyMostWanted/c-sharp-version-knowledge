---
id: cs10-struct-improvements
csharp: 10
tfm: net6.0
status: stable
tags: [struct, constructor, initializer, with]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/struct
---
# Improvements of structure types (C# 10)

> **Abstract.** Structs may declare an explicit **parameterless constructor** and **field / property initializers**,
> and the `with` expression now works on **any struct** and on anonymous types. The crucial caveat:
> `default(T)` and array elements never run constructors or initializers, so a struct's "real" default is still all zeros.

| | |
|---|---|
| Introduced | C# 10 · .NET 6 (`net6.0`) |
| Status | Stable |
| Kind | Syntax + semantics |
| Runtime support needed | No |
| Audience | Everyday code |
| Modernize category | n/a (use in new code where helpful) |

## 1. Motivation

Before C# 10 a struct could not express "a new instance starts with `Interval = 1.5f`": initializers were forbidden
and `new S()` always meant zero-initialization. Non-destructive mutation with `with` existed only for records.

## 2. The feature explained

### 2.1 Syntax

```csharp
public struct S
{
    public int A = 1;                       // field initializer
    public float B { get; set; } = 2f;      // property initializer
    public S() { }                          // explicit parameterless constructor
}

var t = s with { B = 3f };                  // with on any struct
var u = anon with { Name = "x" };           // with on anonymous types
```

### 2.2 Semantics and rules

1. Declare the parameterless constructor `public`; that is the form the documentation uses and the one that
   works with `new()` generic constraints.
2. A struct that declares field or property initializers must also declare an explicit constructor (error CS8983);
   the initializers run as part of that constructor.
3. Field initializers run only as part of a constructor call (`new S()` with a declared parameterless constructor, or any declared constructor).
4. `default(S)`, `new S[n]` and uninitialized fields of type `S` produce **all-zero** values; no constructor runs.
5. `with` copies the struct and assigns the listed members; members must be settable (fields, `set` or `init` properties).

## 3. Code examples

### 3.1 Minimal example

```csharp
public struct Spawner
{
    public float Interval { get; set; } = 1.5f;
    public int Max = 10;
    public Spawner() { }   // needed so the initializers run for `new Spawner()`
}

var fast = new Spawner() with { Interval = 0.5f };   // Interval 0.5, Max 10
var zero = default(Spawner);                         // Interval 0, Max 0 (!)
```

### 3.2 Extended examples

`with` on a plain (non-record) struct:

```csharp
public struct Transform2D
{
    public Vector2 Position;
    public float Rotation;
    public Vector2 Scale;
}

var moved = transform with { Position = transform.Position + offset };
```

`with` on anonymous types:

```csharp
var settings = new { Volume = 0.8f, Muted = false };
var muted = settings with { Muted = true };
```

Constructor validation plus defaults:

```csharp
public readonly struct Cooldown
{
    public float Duration { get; } = 1f;
    public Cooldown() { }
    public Cooldown(float duration)
        => Duration = duration > 0 ? duration : throw new ArgumentOutOfRangeException(nameof(duration));
}
```

## 4. Before this feature

```csharp
public struct Spawner
{
    public float Interval;
    public int Max;

    public static Spawner Create() => new Spawner { Interval = 1.5f, Max = 10 };   // factory instead of ctor
}

var fast = Spawner.Create();
fast.Interval = 0.5f;            // instead of `with`
```

## 5. Under the hood

Field initializers are inlined at the start of every constructor that does not chain to `this(...)`:

```csharp
public Spawner()
{
    this.<Interval>k__BackingField = 1.5f;
    this.Max = 10;
}
```

`new Spawner()` with a declared parameterless constructor emits a constructor call (`newobj` / `call .ctor`);
without one it emits `initobj` (zeroing). `default(Spawner)` and array allocation always zero memory and never call `.ctor`.

`s with { B = 3f }` lowers to:

```csharp
Spawner tmp = s;    // copy
tmp.B = 3f;
```

## 6. Performance and allocation

- No allocations. A declared parameterless constructor makes `new S()` run code instead of a simple zeroing; negligible unless in tight loops.
- Generic code `new T()` with `where T : new()` calls the declared constructor via `Activator.CreateInstance<T>()`, which is slower than `default`.

## 7. Pitfalls

- **Arrays and `default` skip initializers.** `new Spawner[100]` gives `Interval = 0` everywhere.
  Never rely on initializers for structs stored in arrays, pools or SoA/AoS buffers; treat zero as a valid state.
- `with` on a large struct copies the whole struct.
- `with` on a mutable struct stored in a `List<T>` returns a copy; you must write it back (`list[i] = list[i] with { ... }`).

## 8. Concerns and limitations

- Readability: two different "defaults" (`new S()` vs `default(S)`) can confuse readers; document which one is valid.
- Godot: Godot's own structs (`Vector2`, `Color`) work with `with` (`pos with { Y = 0 }`).

## 9. When to use: clean-code guidance

### 9.1 Use when
- A struct has meaningful non-zero defaults **and** is mostly created with `new()`.
- Non-destructive updates of small structs: `with`.

### 9.2 Avoid when
- The struct lives mostly in arrays or is created via `default`: design it so that zero is the correct default instead.

### 9.3 Good versus bad

```csharp
// Good: zero is a valid default, `with` for updates
public struct Health { public int Current; public int Max; }
var h2 = h with { Current = Math.Min(h.Current + 10, h.Max) };

// Bad: correctness depends on initializers, but instances come from an array
public struct Tile { public int Hp = 100; public Tile() { } }
var tiles = new Tile[1024];   // all Hp == 0
```

## 10. Related features

- [record-structs.md](record-structs.md)
- [../11/auto-default-structs.md](../11/auto-default-structs.md): constructors need not assign every field.
- [../12/primary-constructors.md](../12/primary-constructors.md): primary constructors on structs.

## 11. References

- [Structure types](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/struct)
- [Proposal: Parameterless struct constructors](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-10.0/parameterless-struct-constructors)
- [with expression](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/with-expression)
