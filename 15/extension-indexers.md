---
id: cs15-extension-indexers
csharp: 15
tfm: net11.0
status: stable
tags: [extension, indexer, extension-members]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-15
related: ../14/extension-members.md
modernizes: GetPixel(x, y)-style accessors on foreign types
---
# Extension indexers (C# 15)

> **Abstract.** Indexers can be declared in an `extension` block (C# 14 extension members). The receiver can then be
> indexed as if the indexer were declared on its type: `image[x, y]` instead of `image.GetPixel(x, y)`.
> An indexer suggests cheap access, so only wrap operations that really are O(1) or close to it.

| | |
|---|---|
| Introduced | C# 15 · .NET 11 (`net11.0`) |
| Status | Stable |
| Kind | Syntax (extension members) |
| Runtime support needed | No |
| Audience | Everyday code |
| Modernize category | CASE |

## 1. Motivation

C# 14 extension blocks covered methods, properties and operators but not indexers. Types you do not own that have a
natural "get by key or position" accessor (`GetPixel`, `GetCellTileData`, `GetValue(row, column)`) could not be indexed.

## 2. The feature explained

### 2.1 Syntax

```csharp
public static class Ext
{
    extension(Receiver r)            // named receiver is required
    {
        public T this[int i] => ...;
        public T this[int x, int y] => ...;
    }
}
```

### 2.2 Semantics and rules

1. The extension block must have a **named receiver parameter**, because indexers are instance members.
2. Multiple parameters and overloads by parameter types are allowed.
3. Real indexers on the type always win over extension indexers.
4. The official example is get-only; check the specification before declaring setters, especially for value-type receivers (a setter on a by-value struct receiver would mutate a copy).

## 3. Code examples

### 3.1 Minimal example

```csharp
public static class SequenceIndexer
{
    extension(IEnumerable<int> sequence)
    {
        public int this[int index] => sequence.ElementAt(index);
    }
}

int third = Enumerable.Range(1, 10)[2];   // 3
```

### 3.2 Extended examples

Godot image and tile access:

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
bool walkable = ground[cell]?.GetCustomData("walkable").AsBool() ?? false;
```

Configuration lookups with a key:

```csharp
public static class ConfigIndexers
{
    extension(IConfiguration config)
    {
        public int this[string key, int fallback] => int.TryParse(config[key], out var v) ? v : fallback;
    }
}

int maxPlayers = configuration["Game:MaxPlayers", 8];
```

## 4. Before this feature

```csharp
Color c = image.GetPixel(x, y);
TileData? t = layer.GetCellTileData(cell);
int maxPlayers = configuration.GetValue("Game:MaxPlayers", 8);
```

## 5. Under the hood

Like other extension members, the indexer compiles to static accessor methods in the containing static class with the receiver as first parameter
(roughly `get_Item(Image image, int x, int y)`), plus compiler-generated metadata describing the extension block.
`image[x, y]` becomes a static call.

## 6. Performance and allocation

Same as a static method call. The cost is whatever the indexer body does.

## 7. Pitfalls

- **An indexer looks O(1).** Do not hide expensive work behind it: `ElementAt` on `IEnumerable<T>` is usually O(n), so a loop becomes O(n²).
- Engine calls behind indexers (`Image.GetPixel`) cross the native boundary per access; fine for occasional reads, slow for full-image loops (use `GetData()` once).
- A future real indexer on the type silently replaces the extension.

## 8. Concerns and limitations

- Discoverability: readers may not know an indexer is an extension. Keep extension indexers in clearly named classes.
- Godot: compile-time only, invisible to GDScript.

## 9. When to use: clean-code guidance

### 9.1 Use when
- A foreign type has a natural, cheap "get by key or position" method.

### 9.2 Avoid when
- The access is expensive, allocates or has side effects.

### 9.3 Good versus bad

```csharp
// Good
extension(TileMapLayer layer) { public int this[Vector2I c] => layer.GetCellSourceId(c); }

// Bad: O(n) indexer used in loops
extension(LinkedList<int> list) { public int this[int i] => list.ElementAt(i); }
```

## 10. Related features

- [../14/extension-members.md](../14/extension-members.md)
- [../13/implicit-index-in-object-initializers.md](../13/implicit-index-in-object-initializers.md)

## 11. References

- [What's new in C# 15](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-15)
- [Extension members](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/extension-methods)
