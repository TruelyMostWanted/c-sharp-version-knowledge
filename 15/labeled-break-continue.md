---
id: cs15-labeled-break-continue
csharp: 15
tfm: net11.0
status: stable
tags: [break, continue, label, nested-loops, goto]
analyzer: IDE0410
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-15
---
# Labeled `break` and `continue` (C# 15)

`break` and `continue` can name a label on an enclosing loop or `switch`.
A labeled `break` exits that statement; a labeled `continue` starts its next iteration.

## Use when
- Nested loops where an inner condition must skip or end an outer loop
  (grid search, placement checks, collision pairs, inventory fit).
- Replaces bool flags checked at each level and `goto` past loops. Analyzer **IDE0410** flags these patterns.

## Syntax
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

## Rules
- Put the label directly on the loop or `switch` it identifies.
- Without a label, `break` and `continue` keep their old meaning (innermost statement).

## Pitfalls
- If the loop can be extracted into its own method, `return` is often clearer.

## Example (game: first free build spot)
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
