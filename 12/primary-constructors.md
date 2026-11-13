---
id: cs12-primary-constructors
csharp: 12
tfm: net8.0
status: stable
tags: [constructor, primary-constructor, dependency-injection]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-12
---
# Primary constructors for classes and structs (C# 12)

Any `class` or `struct` (not only records) can declare parameters after its name. They are in scope in the whole body.

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

## Rules
- Every explicitly declared constructor must call the primary constructor via `this(...)`.
- A class with a primary constructor gets **no** implicit parameterless constructor.
- In a struct, the implicit parameterless constructor zero-initializes everything, including primary constructor parameters.
- Public properties are generated only for `record` types, not for classes or structs.

## Pitfalls
- Parameters are captured as mutable hidden fields, not `readonly`. Assign to a `readonly` field if immutability matters.
- Capturing a parameter **and** assigning it to a field or property stores it twice (compiler warning).
- **Godot:** do not add a primary constructor to script classes deriving from `GodotObject` (`Node`, `Resource`, ...).
  The engine instantiates scripts through a parameterless constructor. Use primary constructors for plain C# services and data.
