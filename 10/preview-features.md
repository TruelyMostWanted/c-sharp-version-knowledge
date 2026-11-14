---
id: cs10-preview-features
csharp: 10
tfm: net6.0
status: preview (stable in C# 11)
tags: [preview, generic-attributes, static-abstract]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-10
---
# C# 10 preview features (pointer chapter)

> **Abstract.** C# 10 shipped two features only as **preview**: generic attributes and static abstract members in
> interfaces (the basis of generic math). Both became stable in C# 11 (.NET 7). On a `net6.0` project without
> `<LangVersion>preview</LangVersion>` they are not available, and even with preview they should not be used.

| | |
|---|---|
| Introduced | C# 10 (preview) · stable in C# 11 · .NET 7 |
| Status | Preview in C# 10 |
| Kind | Pointer chapter |
| Runtime support needed | Static abstract members need runtime support (.NET 6 preview, .NET 7 stable) |
| Audience | Everyone on `net6.0` |
| Modernize category | NEVER-AUTO on C# 10 |

## 1. What this means for a C# 10 project

| Feature | C# 10 | Full chapter |
|---|---|---|
| Generic attributes `[Attr<T>]` | preview only | [../11/generic-attributes.md](../11/generic-attributes.md) |
| `static abstract` / `static virtual` interface members, generic math | preview only (`System.Runtime.Experimental` package) | [../11/generic-math.md](../11/generic-math.md) |

## 2. Opting in (and why you usually should not)

```xml
<LangVersion>preview</LangVersion>
<EnablePreviewFeatures>true</EnablePreviewFeatures>   <!-- .NET 6 runtime preview APIs -->
```

- Preview semantics changed before C# 11 shipped; code may not compile with the C# 11 compiler unchanged.
- .NET 6 generic math required the experimental package and preview runtime features, unsupported for production.
- `net6.0` is out of support. The right fix is to raise `<TargetFramework>`.

## 3. Code without the feature (what to write instead)

Generic attribute replacement:

```csharp
[AttributeUsage(AttributeTargets.Class)]
public sealed class ComponentAttribute : Attribute
{
    public ComponentAttribute(Type type) => Type = type;
    public Type Type { get; }
}

[Component(typeof(HealthComponent))]
public partial class Enemy { }
```

Generic math replacement: overloads per numeric type, or a strategy interface:

```csharp
public interface IAdder<T> { T Zero { get; } T Add(T a, T b); }

public readonly struct IntAdder : IAdder<int>
{
    public int Zero => 0;
    public int Add(int a, int b) => a + b;
}

static T Sum<T, TAdder>(ReadOnlySpan<T> values, TAdder adder) where TAdder : struct, IAdder<T>
{
    T total = adder.Zero;
    foreach (var v in values) total = adder.Add(total, v);
    return total;
}
```

(Using a `struct` constraint lets the JIT specialize and inline the adder, avoiding interface dispatch.)

## 4. Migration once the project reaches C# 11

- `[Component(typeof(X))]` → `[Component<X>]` (CASE: public API change).
- Strategy structs / overload families → `where T : INumber<T>` (CASE).

## 5. References

- [What's new in C# 10](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-10)
- [Preview features in .NET 6: generic math (.NET Blog)](https://devblogs.microsoft.com/dotnet/preview-features-in-net-6-generic-math/)
