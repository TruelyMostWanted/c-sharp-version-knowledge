---
id: cs14-user-defined-compound-assignment
csharp: 14
tfm: net10.0
status: stable
tags: [operator, compound-assignment, increment, in-place, allocation]
source:
  - https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/operator-overloading
  - https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-14.0/user-defined-compound-assignment
modernizes: x = x + y allocating on mutable reference types
---
# User-defined compound assignment and instance `++` / `--` (C# 14)

> **Abstract.** Types can declare `+=`, `-=`, `*=`, `/=`, `%=`, `&=`, `|=`, `^=`, `<<=`, `>>=`, `>>>=` and `++` / `--`
> as **instance operators that mutate in place**. Before C# 14, `x += y` always meant `x = x + y`, which for
> reference types allocates a new object every time. Now accumulators, buffers and large numeric types can update
> themselves without garbage.

| | |
|---|---|
| Introduced | C# 14 · .NET 10 (`net10.0`) |
| Status | Stable |
| Kind | Language (operator overloading) |
| Runtime support needed | No |
| Audience | Library authors, performance code |
| Modernize category | CASE |

## 1. Motivation

For a mutable reference type such as a big matrix, a vector of 1,000 floats or a stat container,
`total += hit` had to call `operator +(total, hit)`, which must return a **new** object. In a loop this produced one
allocation per iteration, even though the caller wanted to mutate `total`.

## 2. The feature explained

### 2.1 Syntax

```csharp
public sealed class Accumulator
{
    public int Value;
    public void operator +=(int x) { Value += x; }   // compound assignment (instance)
    public void operator ++() { Value++; }           // instance increment
}
```

### 2.2 Semantics and rules

1. Compound assignment operators: **instance** members, no `static`, exactly one parameter, return `void`.
2. Instance `++` / `--`: no `static`, no parameters, return `void`.
3. A `checked` variant (`operator checked +=`) requires the regular variant too.
4. Allowed modifiers include `virtual`, `abstract`, `override`, `sealed`, `new`, and `readonly` on structs.
5. Resolution for `x op= y` where `x` is a variable: an applicable instance `op=` is preferred; otherwise the static binary operator (`x = x op y`) is used.
6. Resolution for `++x` / `x++`: the instance operator is used when the result is not needed or for prefix; postfix whose value is used falls back to the static `++` (the old value must be produced).
7. The compiler does **not** check consistency between `+=` and `+`; keep them consistent.

## 3. Code examples

### 3.1 Minimal example

```csharp
var acc = new Accumulator();
acc += 5;      // calls acc.op_AdditionAssignment(5), no new object
acc++;         // calls the instance ++
```

### 3.2 Extended examples

Damage accumulation without per-hit allocation:

```csharp
public sealed class DamageAccumulator
{
    public float Physical, Fire, Poison;

    public void operator +=(DamageAccumulator other)
    {
        Physical += other.Physical;
        Fire     += other.Fire;
        Poison   += other.Poison;
    }

    public static DamageAccumulator operator +(DamageAccumulator a, DamageAccumulator b)
    {
        var r = new DamageAccumulator { Physical = a.Physical, Fire = a.Fire, Poison = a.Poison };
        r += b;
        return r;
    }
}

foreach (var hit in hits) total += hit;   // no new DamageAccumulator per hit
```

Large vector type (`TensorPrimitives` comes from the `System.Numerics.Tensors` package):

```csharp
public sealed class VectorN(int length)
{
    private readonly float[] _data = new float[length];
    public ref float this[int i] => ref _data[i];

    public void operator +=(VectorN other)
    {
        ArgumentOutOfRangeException.ThrowIfNotEqual(other._data.Length, _data.Length);
        TensorPrimitives.Add(_data, other._data, _data);   // SIMD, in place
    }

    public void operator *=(float scale) => TensorPrimitives.Multiply(_data, scale, _data);
}
```

## 4. Before this feature

```csharp
public static DamageAccumulator operator +(DamageAccumulator a, DamageAccumulator b) => new() { ... };

total += hit;          // = total + hit -> allocation per call
total.Add(hit);        // or a named method, losing operator syntax
```

## 5. Under the hood

The operator is an instance method with a special name:

```csharp
public void op_AdditionAssignment(DamageAccumulator other) { ... }
```

`total += hit;` compiles to `total.op_AdditionAssignment(hit);` (with a null check for reference types, as for any instance call).
Increment operators use corresponding special names (see the proposal). When no instance operator applies,
the compiler emits the classic `total = DamageAccumulator.op_Addition(total, hit);`.

## 6. Performance and allocation

- Eliminates one allocation per compound assignment for reference types.
- For structs, in-place operators avoid copying large structs back and forth.

## 7. Pitfalls

- **Aliasing:** in-place mutation changes every reference to the same object. `var a = b; a += c;` now also changes `b`. Do not add in-place operators to types that look immutable.
- `x += y` with `x` being a property: the compiler calls the getter and then the instance operator on the returned object; for a struct property it operates on a copy (likely a bug).
- Postfix `x++` whose value is used falls back to the static operator; if none exists, a compile error.
- Inconsistent `+` and `+=` implementations produce different results depending on syntax.

## 8. Concerns and limitations

- API design: adding `+=` to a public type changes the semantics of existing `x += y` calls in consumers when they recompile (from "new object" to "mutate").
- Godot: Godot's math types are immutable structs with static operators; this feature is for your own mutable types.

## 9. When to use: clean-code guidance

### 9.1 Use when
- Mutable reference types used as accumulators, buffers, builders or large numeric containers.

### 9.2 Avoid when
- Value-like or immutable types (records, small structs): keep static operators only.

### 9.3 Good versus bad

```csharp
// Good: clearly mutable accumulator
public sealed class StatTotals { public void operator +=(StatBlock s) { ... } }

// Bad: record that looks immutable but mutates in place
public sealed record Money(decimal Amount) { public decimal Value = Amount; public void operator +=(Money m) => Value += m.Value; }
```

## 10. Related features

- [../11/generic-math.md](../11/generic-math.md)
- [extension-members.md](extension-members.md): operators for types you don't own.

## 11. References

- [Operator overloading](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/operator-overloading)
- [Proposal: User-defined compound assignment](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-14.0/user-defined-compound-assignment)
