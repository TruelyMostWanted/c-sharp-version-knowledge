---
id: cs11-auto-default-structs
csharp: 11
tfm: net7.0
status: stable
tags: [struct, constructor, definite-assignment]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/struct#struct-initialization-and-default-values
modernizes: struct constructors assigning default values to every field
---
# Auto-default structs (C# 11)

> **Abstract.** A struct constructor no longer has to assign every field before it returns. The compiler
> zero-initializes any field the constructor does not definitely assign. Boilerplate such as `Velocity = default;`
> disappears, and `this` can be used before all fields are set.

| | |
|---|---|
| Introduced | C# 11 · .NET 7 (`net7.0`) |
| Status | Stable |
| Kind | Semantics (definite assignment) |
| Runtime support needed | No |
| Audience | Everyday code |
| Modernize category | PREFER (drop redundant assignments) |

## 1. Motivation

C# required every field of a struct to be definitely assigned in each constructor (error CS0171), and `this`
could not be used before that point (CS0188). Developers wrote `: this()` chains or long lists of `X = default;`
lines that carried no information.

## 2. The feature explained

### 2.1 Syntax

No new syntax.

### 2.2 Semantics and rules

1. Fields (and auto-property backing fields) not definitely assigned by the end of the constructor are set to `default`.
2. Calling methods on `this` or reading fields before all are assigned is allowed; unassigned fields read as zero.
3. The compiler emits the zeroing only where needed (fields not assigned on every path).
4. Optional informational warnings (CS9020 / CS9021 family, off by default) report where auto-defaulting happens, for code that wants explicit initialization.

## 3. Code examples

### 3.1 Minimal example

```csharp
public struct Particle
{
    public Vector2 Position;
    public Vector2 Velocity;
    public float Life;

    public Particle(Vector2 position) { Position = position; }   // Velocity and Life default to 0
}
```

### 3.2 Extended examples

Calling a helper on `this` before all fields are assigned:

```csharp
public struct Ring
{
    public int Head;
    public int Count;
    public int Capacity;

    public Ring(int capacity)
    {
        Capacity = capacity;
        Reset();                 // allowed in C# 11: Head and Count are already zero
    }

    public void Reset() { Head = 0; Count = 0; }
}
```

Conditional assignment:

```csharp
public readonly struct Hit
{
    public readonly float Damage;
    public readonly bool Critical;

    public Hit(float damage, float critChance, Random rng)
    {
        Damage = damage;
        if (rng.NextSingle() < critChance) { Damage *= 2; Critical = true; }
        // Critical is auto-defaulted on the non-crit path
    }
}
```

## 4. Before this feature

```csharp
public Particle(Vector2 position)
{
    Position = position;
    Velocity = default;
    Life = 0f;
}

// or
public Particle(Vector2 position) : this() { Position = position; }
```

## 5. Under the hood

The compiler inserts zero assignments at the start of the constructor for fields not definitely assigned:

```csharp
public Particle(Vector2 position)
{
    this.Velocity = default;   // inserted
    this.Life = 0f;            // inserted
    this.Position = position;
}
```

If no field is definitely assigned it may instead emit a single `initobj` on `this` (whole struct zeroed).

## 6. Performance and allocation

Equivalent to the hand-written zeroing. Constructors of large structs that assign only a few fields still zero the rest, as before.

## 7. Pitfalls

- A field you **forgot** to assign is no longer a compile error; it silently becomes zero. For structs whose
  invariants require every field, enable the informational warnings or keep explicit assignments.

## 8. Concerns and limitations

- None for compatibility: code that compiled before compiles identically.

## 9. When to use: clean-code guidance

### 9.1 Use when
- Removing `= default` lines and `: this()` chains that only existed to satisfy the old rule.

### 9.2 Avoid when
- The explicit assignment documents intent (a meaningful initial value that happens to be zero, such as `Level = 0` in a domain rule): keep it.

### 9.3 Good versus bad

```csharp
// Good
public Bullet(Vector2 pos, Vector2 vel) { Position = pos; Velocity = vel; }

// Bad (on C# 11+): noise
public Bullet(Vector2 pos, Vector2 vel) : this() { Position = pos; Velocity = vel; Age = 0; Bounces = 0; Flags = default; }
```

## 10. Related features

- [../10/struct-improvements.md](../10/struct-improvements.md)
- [../12/primary-constructors.md](../12/primary-constructors.md)

## 11. References

- [Struct initialization and default values](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/struct#struct-initialization-and-default-values)
- [Proposal: Auto-default structs](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-11.0/auto-default-structs)
