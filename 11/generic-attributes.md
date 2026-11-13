---
id: cs11-generic-attributes
csharp: 11
tfm: net7.0
status: stable (preview in C# 10)
tags: [attribute, generics]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-11
modernizes: attributes taking a Type parameter ([X(typeof(Foo))])
---
# Generic attributes (C# 11)

```csharp
public class ComponentAttribute<T> : Attribute where T : class { }

[Component<HealthComponent>]
public partial class Enemy { }
```
Before: `public ComponentAttribute(Type type)` used as `[Component(typeof(HealthComponent))]`, with no compile-time check.

## Rules
- Type arguments must be fully known at compile time: no `dynamic`, no nullable reference annotations (`string?`),
  no tuple element names. Open type parameters of an enclosing generic type are not allowed.
