---
id: cs11-generic-math
csharp: 11
tfm: net7.0
status: stable
needs-runtime: true (.NET 7 interfaces INumber<T> etc.)
tags: [generic-math, static-abstract, interface, INumber, operators, >>>]
source: https://learn.microsoft.com/en-us/dotnet/standard/generics/math
modernizes: one overload per numeric type (int/float/double...), dynamic, code generation
---
# Generic math support (C# 11 / .NET 7)

Interfaces can declare `static abstract` and `static virtual` members, including operators.
.NET 7 uses this for the generic math interfaces (`INumber<T>`, `IFloatingPoint<T>`, `IAdditionOperators<,,>`, ...).

```csharp
static T Sum<T>(ReadOnlySpan<T> values) where T : INumber<T>
{
    T total = T.Zero;
    foreach (var v in values) total += v;
    return total;
}

int   a = Sum<int>([1, 2, 3]);
float b = Sum<float>([0.5f, 1.5f]);
```

Own interfaces with static abstract members:
```csharp
public interface IStatBlock<TSelf> where TSelf : IStatBlock<TSelf>
{
    static abstract TSelf Default { get; }
    static abstract TSelf operator +(TSelf left, TSelf right);
}
```

## Related C# 11 additions (part of this feature set)
- `checked` user-defined operators (`public static T operator checked +(T a, T b)`).
- Unsigned right shift `>>>`.
- Relaxed shift operators: the right operand need not be `int`.

## Pitfalls
- Static abstract members are callable only through a type parameter (`T.Zero`), not through the interface itself.
- Godot's own `Vector2/3`, `Color` etc. do not implement these interfaces; use generic math for scalar code.
