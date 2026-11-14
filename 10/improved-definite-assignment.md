---
id: cs10-improved-definite-assignment
csharp: 10
tfm: net6.0
status: stable
tags: [definite-assignment, nullable, warnings]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-10#improved-definite-assignment
modernizes: dummy initializations and ! added to silence false warnings
---
# Improved definite assignment and null-state analysis (C# 10)

> **Abstract.** The compiler's flow analysis became more precise for `?.`, `??`, `is` and comparisons with
> constant `true` / `false`. Code that was correct but rejected ("use of unassigned local variable") or warned
> about (possible null) now compiles cleanly. There is no new syntax; the gain is that workarounds can be removed.

| | |
|---|---|
| Introduced | C# 10 · .NET 6 (`net6.0`) |
| Status | Stable |
| Kind | Semantics (flow analysis) |
| Runtime support needed | No (compile-time only) |
| Audience | Everyday code |
| Modernize category | CASE (remove workarounds when touching code) |

## 1. Motivation

Definite assignment requires that a local is assigned on every path before it is read. Before C# 10 the
compiler did not "see through" some common shapes, for example a `TryGetValue` reached through `?.` and
compared with `true`. Developers added `= default` or `!` just to satisfy the compiler, which hides real bugs.

## 2. The feature explained

### 2.1 Syntax

No new syntax.

### 2.2 Semantics and rules

The analysis now tracks assignment and null state through:

1. `c?.M(out x) == true` / `!= false` / `is true`: in the true branch, `x` is definitely assigned and `c` is not null.
2. `c?.M(out x) ?? false`-style coalescing with constants.
3. Conditional expressions `cond ? M(out x) : false` compared with constants.
4. `is` patterns combined with `?.` (`obj?.Prop is { } v`).

## 3. Code examples

### 3.1 Minimal example

```csharp
if (dict?.TryGetValue(key, out var value) == true)
{
    Use(value);   // C# 10: value is definitely assigned here
}
```

### 3.2 Extended examples

```csharp
// Null state flows through ?. and == true
if (player?.Weapon?.TryFire(out Projectile p) == true)
{
    _projectiles.Add(p);   // player and player.Weapon known non-null here, p assigned
}

// Conditional expression compared with a constant
if ((cache is null ? false : cache.TryGet(id, out var item)) == true)
    Render(item);

// Coalescing to a constant
if (_lookup?.TryGetValue(name, out var entry) ?? false)
    Use(entry);
```

## 4. Before this feature

```csharp
Projectile p = null!;                                   // dummy init + null-forgiving
if (player != null && player.Weapon != null && player.Weapon.TryFire(out p))
    _projectiles.Add(p);
```

## 5. Under the hood

No change to generated IL. Only the compiler's static analysis (definite assignment and nullable flow state) changed.

## 6. Performance and allocation

None.

## 7. Pitfalls

- Removing `= default` initializers is safe only where the compiler is now satisfied **and** the value really is
  assigned on every path you care about. Let the compiler decide; do not remove initializers by hand on a guess.
- `!` (null-forgiving) removals: rebuild with nullable warnings as errors to verify.

## 8. Concerns and limitations

- None. Code that compiled before still compiles.

## 9. When to use: clean-code guidance

### 9.1 Use when
- Touching code that contains `= default`, `= null!` or `!` added only to silence the compiler.

### 9.2 Avoid when
- Not applicable; this is automatic.

### 9.3 Good versus bad

```csharp
// Good (C# 10)
if (map?.TryGetValue(k, out var v) == true) Use(v);

// Bad: hides a real "unassigned" bug if the code later changes
var v2 = default(Thing)!;
if (map != null && map.TryGetValue(k, out v2)) Use(v2);
```

## 10. Related features

- [extended-property-patterns.md](extended-property-patterns.md): patterns feed the same flow analysis.

## 11. References

- [What's new in C# 10: improved definite assignment](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-10#improved-definite-assignment)
- [Proposal: Improved definite assignment](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-10.0/improved-definite-assignment)
- [Nullable reference types](https://learn.microsoft.com/en-us/dotnet/csharp/nullable-references)
