---
id: cs10-extended-property-patterns
csharp: 10
tfm: net6.0
status: stable
analyzer: IDE0170
tags: [pattern-matching, property-pattern]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/patterns#property-pattern
modernizes: nested property patterns { A: { B: x } }
---
# Extended property patterns (C# 10)

> **Abstract.** Inside a property pattern, nested members can be addressed with a dotted path:
> `{ Customer.Address.City: "Paris" }` instead of `{ Customer: { Address: { City: "Paris" } } }`.
> Semantics are identical (including the implicit null checks); only the noise is gone.

| | |
|---|---|
| Introduced | C# 10 · .NET 6 (`net6.0`) |
| Status | Stable |
| Kind | Syntax (pattern matching) |
| Runtime support needed | No (compile-time only) |
| Audience | Everyday code |
| Modernize category | AUTO (IDE0170) |

## 1. Motivation

Property patterns (C# 8) made it possible to test object shapes declaratively, but nested members required
nested braces, which became unreadable after two levels:

```csharp
if (order is { Customer: { Address: { City: "Paris" } } }) { }
```

## 2. The feature explained

### 2.1 Syntax

```csharp
expr is { A.B.C: pattern }
expr is { A.B: pattern1, A.D: pattern2 }
```

### 2.2 Semantics and rules

1. `{ A.B: p }` is exactly equivalent to `{ A: { B: p } }`.
2. Each intermediate member access includes an implicit **null check** for reference types and nullable values; if any link is null, the pattern fails (no `NullReferenceException`).
3. Any pattern can follow the colon: constant, relational (`>= 10`), type, `var`, list patterns (C# 11), nested property patterns.
4. Works in `is`, `switch` expressions and `switch` statements.

## 3. Code examples

### 3.1 Minimal example

```csharp
if (order is { Customer.Address.City: "Paris" }) { }
```

### 3.2 Extended examples

Several paths into the same object:

```csharp
bool canBuild = player is { Inventory.Wood: >= 10, Inventory.Stone: >= 5, Stats.Level: > 2 };
```

Switch expression with extended patterns:

```csharp
decimal Shipping(Order o) => o switch
{
    { Customer.IsPremium: true }                    => 0m,
    { Customer.Address.Country: "DE", Total: > 50 } => 0m,
    { Customer.Address.Country: "DE" }              => 4.90m,
    _                                               => 14.90m,
};
```

Capturing an inner value with `var`:

```csharp
if (evt is { Source.Owner.Team: var team } && team != myTeam) Alert(team);
```

Combined with a type pattern:

```csharp
if (collider is CharacterBody2D { Velocity.Y: > 0 } body) Land(body);
```

## 4. Before this feature

```csharp
bool canBuild = player is { Inventory: { Wood: >= 10, Stone: >= 5 } };

// or imperative
bool canBuild2 = player?.Inventory != null
              && player.Inventory.Wood >= 10
              && player.Inventory.Stone >= 5;
```

## 5. Under the hood

Lowered to a chain of member reads and null checks, each member read once per path:

```csharp
// order is { Customer.Address.City: "Paris" }
bool result = false;
if (order != null)
{
    Customer c = order.Customer;
    if (c != null)
    {
        Address a = c.Address;
        if (a != null)
            result = a.City == "Paris";    // string equality via string.op_Equality
    }
}
```

In `switch` expressions the compiler builds a decision DAG and reuses already-read members across arms.

## 6. Performance and allocation

No allocation. Same as the hand-written null checks; property getters are called (avoid patterns over expensive getters).

## 7. Pitfalls

- Property getters with side effects run during matching; the number of calls is not something to rely on.
- Mixing both styles (`{ A.B: 1, A: { C: 2 } }`) is legal but confusing.

## 8. Concerns and limitations

- None for compatibility. Readability can suffer if paths get long; consider a helper property.
- Godot: patterns on engine objects call into native getters; fine for occasional checks, avoid in tight per-frame loops over many nodes.

## 9. When to use: clean-code guidance

### 9.1 Use when
- Checking one or two nested values declaratively, especially in `switch` expressions.

### 9.2 Avoid when
- The path crosses many objects (Law of Demeter): introduce a property such as `order.ShipsToParis`.

### 9.3 Good versus bad

```csharp
// Good
if (unit is { Weapon.Range: > 5 }) EngageFromDistance(unit);

// Bad: deep path couples the caller to the whole object graph
if (game is { World.Map.Regions.Current.Weather.Wind.Speed: > 20 }) ...
```

## 10. Related features

- [../11/list-patterns.md](../11/list-patterns.md): patterns over sequences.
- [improved-definite-assignment.md](improved-definite-assignment.md): flow analysis after patterns.

## 11. References

- [Property pattern](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/patterns#property-pattern)
- [Proposal: Extended property patterns](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-10.0/extended-property-patterns)
- [IDE0170](https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/style-rules/ide0170)
