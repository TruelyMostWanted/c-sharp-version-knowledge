---
id: cs15-labeled-break-continue
csharp: 15
tfm: net11.0
status: stable
tags: [break, continue, label, nested-loops, goto]
analyzer: IDE0410
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-15
modernizes: bool flags or goto to leave nested loops
---
# Labeled `break` and `continue` (C# 15)

> **Abstract.** `break` and `continue` can name a **label on an enclosing loop or `switch`**. `break outer;` leaves the
> labeled statement; `continue outer;` starts its next iteration. This replaces boolean flags checked at every level
> and `goto` past loops, and analyzer IDE0410 suggests the rewrite.

| | |
|---|---|
| Introduced | C# 15 · .NET 11 (`net11.0`) |
| Status | Stable |
| Kind | Syntax |
| Runtime support needed | No |
| Audience | Everyday code |
| Modernize category | AUTO (IDE0410) |

## 1. Motivation

Leaving or skipping an outer loop from an inner loop required either a flag:

```csharp
bool found = false;
for (...) { for (...) { if (cond) { found = true; break; } } if (found) break; }
```

or `goto`, which many style guides forbid. Java, JavaScript, Kotlin, Rust and Go have had labeled loops for years.

## 2. The feature explained

### 2.1 Syntax

```csharp
outer: for (...)
{
    for (...)
    {
        if (a) continue outer;
        if (b) break outer;
    }
}
```

### 2.2 Semantics and rules

1. The label must be placed **directly** on a loop (`for`, `foreach`, `while`, `do`) or a `switch` statement.
2. `break label;` exits the labeled statement; `continue label;` jumps to the next iteration of the labeled loop (`continue` cannot target a `switch`).
3. The label must enclose the `break`/`continue`; jumping out of lambdas or local functions is not allowed.
4. `finally` blocks between the jump and the target run, as with any jump.
5. Unlabeled `break` / `continue` keep their old meaning (innermost statement).

## 3. Code examples

### 3.1 Minimal example

```csharp
outer: for (int row = 0; row < grid.Height; row++)
{
    for (int column = 0; column < grid.Width; column++)
    {
        if (grid[row, column].IsBlocked) continue outer;
        if (grid[row, column].IsGoal)    break outer;
    }
}
```

### 3.2 Extended examples

First free build spot:

```csharp
Vector2I? result = null;
rows: for (int y = 0; y <= MapHeight - size.Y; y++)
{
    candidates: for (int x = 0; x <= MapWidth - size.X; x++)
    {
        for (int dy = 0; dy < size.Y; dy++)
        for (int dx = 0; dx < size.X; dx++)
            if (_blocked[(y + dy) * MapWidth + (x + dx)])
                continue candidates;

        result = new Vector2I(x, y);
        break rows;
    }
}
```

Collision pairs with early exit:

```csharp
pairs: foreach (var a in bodies)
{
    foreach (var b in bodies)
    {
        if (ReferenceEquals(a, b)) continue;
        if (a.Overlaps(b) && a.IsPlayer && b.IsLava)
        {
            KillPlayer();
            break pairs;
        }
    }
}
```

Leaving a loop from inside a `switch`:

```csharp
loop: while (reader.Read())
{
    switch (reader.TokenType)
    {
        case JsonTokenType.EndObject:
            break loop;                 // before C# 15, `break` here only left the switch
        case JsonTokenType.PropertyName:
            ReadProperty(ref reader);
            break;
    }
}
```

## 4. Before this feature

```csharp
Vector2I? result = null;
for (int y = 0; y <= MapHeight - size.Y && result is null; y++)
{
    for (int x = 0; x <= MapWidth - size.X; x++)
    {
        bool blocked = false;
        for (int dy = 0; dy < size.Y && !blocked; dy++)
        for (int dx = 0; dx < size.X && !blocked; dx++)
            blocked = _blocked[(y + dy) * MapWidth + (x + dx)];

        if (blocked) continue;
        result = new Vector2I(x, y);
        break;
    }
}

// or goto
for (...) { for (...) { if (found) goto done; } }
done:;
```

## 5. Under the hood

Labeled jumps compile to unconditional branches (`br` / `leave` when crossing `try` regions) to the end of the labeled
statement (`break`) or to its continue point (`continue`), exactly what a hand-written `goto` would produce.

## 6. Performance and allocation

None; often slightly faster than flag-based code (fewer checks per iteration).

## 7. Pitfalls

- Deeply nested loops with several labels are still hard to read. If the loop can be extracted into its own method, `return` is often clearer.
- The label name is a code smell detector: if you cannot name it well (`rows`, `candidates`, `pairs`), reconsider the structure.

## 8. Concerns and limitations

- None for compatibility.
- Godot: useful in grid, tile and pathfinding code that runs per frame (no allocations, no flags).

## 9. When to use: clean-code guidance

### 9.1 Use when
- Nested loops where an inner condition must skip or end an outer loop (grid search, placement checks, collision pairs, inventory fit).

### 9.2 Avoid when
- A helper method with `return` expresses the intent better (e.g. `TryFindFreeSpot(out pos)`).

### 9.3 Good versus bad

```csharp
// Good
search: foreach (var slot in inventory.Slots)
    foreach (var item in incoming)
        if (slot.TryAdd(item)) continue search;

// Bad: labels on every level, jumps in all directions
a: for (...) { b: for (...) { c: for (...) { if (x) continue a; if (y) break b; if (z) continue c; } } }
```

## 10. Related features

- [closed-hierarchies.md](closed-hierarchies.md): exhaustive `switch` statements.

## 11. References

- [What's new in C# 15](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-15)
- [Jump statements](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/statements/jump-statements)
- [IDE0410](https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/style-rules/ide0410)
