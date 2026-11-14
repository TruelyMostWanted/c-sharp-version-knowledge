---
id: cs13-implicit-index-in-object-initializers
csharp: 13
tfm: net9.0
status: stable
tags: [index, hat-operator, object-initializer]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-13#implicit-index-access
---
# Implicit "from the end" index `^` in object initializers (C# 13)

> **Abstract.** Indexer assignments inside an object initializer may use the from-end index operator `^`:
> `buffer = { [^1] = 0, [^2] = 1 }`. Before C# 13, `^` was rejected in initializers because the object's length is
> not known while it is being initialized.

| | |
|---|---|
| Introduced | C# 13 · .NET 9 (`net9.0`) |
| Status | Stable |
| Kind | Syntax |
| Runtime support needed | No |
| Audience | Everyday code |
| Modernize category | n/a |

## 1. Motivation

`^n` (C# 8) is shorthand for `Length - n`. In object initializers it was not allowed, so filling the last elements of a
nested collection required manual index arithmetic or code after the initializer.

## 2. The feature explained

### 2.1 Syntax

```csharp
var x = new Holder
{
    Items = { [^1] = 10, [^2] = 20 },
};
```

### 2.2 Semantics and rules

1. Applies to indexer initializers (`[index] = value`) inside object initializers, including nested member initializers.
2. The target must support `Index`-based indexing (directly or via `Length`/`Count` plus `int` indexer).
3. The collection must already exist (created by the containing type); the initializer does not create it.

## 3. Code examples

### 3.1 Minimal example

```csharp
public class TimerRemaining
{
    public int[] Buffer { get; set; } = new int[10];
}

var countdown = new TimerRemaining
{
    Buffer =
    {
        [^1] = 0,
        [^2] = 1,
        [^3] = 2,
    }
};
```

### 3.2 Extended examples

Setting sentinels at both ends:

```csharp
var lane = new Lane
{
    Cells =
    {
        [0]  = CellType.Spawn,
        [^1] = CellType.Goal,
    }
};

public sealed class Lane
{
    public CellType[] Cells { get; } = new CellType[16];
}
```

With a `List<T>` that was pre-filled by the constructor:

```csharp
public sealed class Ring
{
    public List<int> Slots { get; } = new(new int[8]);   // 8 elements
}

var r = new Ring { Slots = { [^1] = 99 } };
```

## 4. Before this feature

```csharp
var countdown = new TimerRemaining();
countdown.Buffer[countdown.Buffer.Length - 1] = 0;
countdown.Buffer[countdown.Buffer.Length - 2] = 1;
```

## 5. Under the hood

```csharp
var tmp = new TimerRemaining();
int[] b = tmp.Buffer;
b[b.Length - 1] = 0;
b[b.Length - 2] = 1;
b[b.Length - 3] = 2;
var countdown = tmp;
```

The collection is read once; the length is evaluated per element in order.

## 6. Performance and allocation

None beyond the assignments.

## 7. Pitfalls

- If the collection is empty or shorter than the index, `IndexOutOfRangeException` / `ArgumentOutOfRangeException` at runtime.
- The nested initializer does **not** replace the collection; it indexes into the existing one.

## 8. Concerns and limitations

- Niche; useful mainly for fixed-size buffers exposed by properties.

## 9. When to use: clean-code guidance

### 9.1 Use when
- Initializing the last elements of a pre-sized collection declaratively.

### 9.2 Avoid when
- The collection size is not obvious from the type; readers cannot see whether `^3` is valid.

### 9.3 Good versus bad

```csharp
// Good: obvious fixed size
var board = new Board { Rows = { [0] = TopRow, [^1] = BottomRow } };

// Bad: List<T> that starts empty; ^1 throws
var list = new Holder { Items = { [^1] = 5 } };   // Items = new List<int>() -> exception
```

## 10. Related features

- [../12/collection-expressions.md](../12/collection-expressions.md)

## 11. References

- [What's new in C# 13: implicit index access](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-13#implicit-index-access)
- [Object and collection initializers](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/object-and-collection-initializers)
