---
id: cs14-user-defined-compound-assignment
csharp: 14
tfm: net10.0
status: stable
tags: [operator, compound-assignment, increment, in-place, allocation]
source:
  - https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-14
  - https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-14.0/user-defined-compound-assignment
---
# User-defined compound assignment and instance `++` / `--` (C# 14)

Types can define `+=`, `-=`, `*=`, ... and `++` / `--` as **instance** operators that mutate in place.
Before, `x += y` always meant `x = x + y`, which for reference types allocates a new object.

## Use when
- Mutable reference types where `x = x + y` would allocate: accumulators, buffers, big vectors or matrices,
  builders, stat containers.

## Syntax
```csharp
class C1
{
    public int Value;
    public void operator +=(int x) { Value += x; }   // compound assignment
    public void operator ++() { Value++; }           // instance increment
}
```

## Rules
- Compound assignment operators: no `static`, exactly one parameter, return `void`.
- Instance `++` / `--`: no `static`, no parameters, return `void`.
- A `checked` variant requires the regular variant too.
- Allowed modifiers include `override`, `new`, `readonly` (structs).
- Resolution: for `x op= y` with `x` a variable, an applicable instance `op=` is preferred;
  otherwise the static binary operator is used.
- The compiler does **not** check that `+=` and `+` are consistent. Keep them consistent yourself.

## Pitfalls
- Postfix `x++` whose result is used falls back to the static operator (the old value must be produced).
- Mutating in place changes every reference to the same object; do not do this on shared immutable-looking types.

## Example (game)
```csharp
public sealed class DamageAccumulator
{
    public float Physical, Fire, Poison;
    public void operator +=(DamageAccumulator other)
    {
        Physical += other.Physical; Fire += other.Fire; Poison += other.Poison;
    }
}

total += hit;   // no new DamageAccumulator per hit
```
