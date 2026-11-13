---
id: cs15-extension-indexers
csharp: 15
tfm: net11.0
status: stable
tags: [extension, indexer, extension-members]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-15
related: ../14/extension-members.md
---
# Extension indexers (C# 15)

Indexers can be declared in an `extension` block (C# 14 extension members). The receiver can then
be indexed as if the indexer were declared on its type.

## Use when
- A type you don't own has a natural "get by key or position" method (`GetPixel(x, y)`, `GetCellTileData(cell)`).

## Syntax
```csharp
public static class SequenceIndexer
{
    extension(IEnumerable<int> sequence)          // named receiver is required
    {
        public int this[int index] => sequence.ElementAt(index);
    }
}

int third = Enumerable.Range(1, 10)[2];
```

## Rules
- The extension block must have a **named receiver parameter**, because indexers are instance members.
- The official example is get-only; check the spec before declaring setters.

## Pitfalls
- An indexer looks O(1). Do not hide expensive work behind it: `ElementAt` on `IEnumerable<T>` is
  usually O(n), so a loop becomes O(n^2).
- Real indexers on the type win over extension indexers.

## Example (Godot)
```csharp
public static class GodotIndexers
{
    extension(Image image)
    {
        public Color this[int x, int y] => image.GetPixel(x, y);
    }
    extension(TileMapLayer layer)
    {
        public TileData? this[Vector2I cell] => layer.GetCellTileData(cell);
    }
}

float h = heightmap[x, y].R * MaxHeight;
```
