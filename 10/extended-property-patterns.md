---
id: cs10-extended-property-patterns
csharp: 10
tfm: net6.0
status: stable
tags: [pattern-matching, property-pattern]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/patterns#property-pattern
modernizes: nested property patterns { A: { B: x } }
---
# Extended property patterns (C# 10)

Nested members can be addressed with a dotted path inside a property pattern.

```csharp
if (order is { Customer.Address.City: "Paris" }) { }

bool canBuild = player is { Inventory.Wood: >= 10, Inventory.Stone: >= 5 };
```
Before C# 10: `{ Customer: { Address: { City: "Paris" } } }`.
