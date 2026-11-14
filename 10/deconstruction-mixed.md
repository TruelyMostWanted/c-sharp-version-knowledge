---
id: cs10-deconstruction-mixed
csharp: 10
tfm: net6.0
status: stable
tags: [deconstruction, tuple]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-10#assignment-and-declaration-in-same-deconstruction
---
# Assignment and declaration in the same deconstruction (C# 10)

> **Abstract.** A deconstruction may now **mix** new variable declarations and assignments to existing
> variables: `(var x, y) = GetPosition();`. Before C# 10 a deconstruction had to be either all declarations
> or all assignments, which forced temporary variables in loops and accumulators.

| | |
|---|---|
| Introduced | C# 10 · .NET 6 (`net6.0`) |
| Status | Stable |
| Kind | Syntax |
| Runtime support needed | No (compile-time only) |
| Audience | Everyday code |
| Modernize category | PREFER (where temporaries existed only to work around the rule) |

## 1. Motivation

Deconstruction (C# 7) splits a tuple or any type with a `Deconstruct` method into variables. Typical loops
need to **update** one variable and **introduce** another in the same step, which used to require a temp:

```csharp
int y = 0;
var (x, tmpY) = GetPosition();
y = tmpY;
```

## 2. The feature explained

### 2.1 Syntax

```csharp
(var x, y) = expression;          // x declared, y assigned
(int count, total) = expression;  // explicit type also possible
```

### 2.2 Semantics and rules

1. Each element is either a declaration (`var a`, `int a`) or an assignable expression (local, field, property, indexer, `_` discard).
2. Nesting works: `(var a, (b, var c)) = ...`.
3. Evaluation order is unchanged: left-side target expressions (e.g. `arr[i]`) are evaluated first, then the right side, then assignments left to right.
4. The outer `var (a, b)` form still means "declare all".

## 3. Code examples

### 3.1 Minimal example

```csharp
int y = 0;
(var x, y) = GetPosition();   // declares x, assigns existing y
```

### 3.2 Extended examples

Iterative algorithms (Euclid, Fibonacci) where one value carries over:

```csharp
long a = 0, b = 1;
for (int i = 0; i < n; i++)
    (a, b) = (b, a + b);         // all-assign, valid since C# 7

int remainder;
(var quotient, remainder) = Math.DivRem(total, size);   // mixed, C# 10
```

Updating a field while declaring a local:

```csharp
private Vector2 _lastPosition;

public void Step()
{
    (var delta, _lastPosition) = Integrate(_lastPosition);
    Log(delta);
}
```

Mixing with a discard:

```csharp
(var name, _, Score) = ParseLine(line);   // Score is a property
```

## 4. Before this feature

```csharp
var (quotient, tmp) = Math.DivRem(total, size);
remainder = tmp;
```

## 5. Under the hood

For a tuple on the right side, the compiler evaluates it into a temporary and assigns elements:

```csharp
// (var x, y) = GetPosition();
(int, int) tmp = GetPosition();
int x = tmp.Item1;
y = tmp.Item2;
```

For a non-tuple type it calls `Deconstruct(out var t1, out var t2)` and assigns from the temporaries.
No extra cost compared with the pre-C# 10 workaround.

## 6. Performance and allocation

None beyond normal deconstruction (value tuples are structs; no allocation).

## 7. Pitfalls

- Readability: `(var a, b) = ...` is easy to misread as declaring both. Use it where the mix is the point.
- Assigning to a property inside a deconstruction invokes the setter; side effects happen in left-to-right order after the right side is evaluated.

## 8. Concerns and limitations

- Purely syntactic; no compatibility concerns.

## 9. When to use: clean-code guidance

### 9.1 Use when
- A loop or accumulator updates existing state and introduces a new local at the same time.

### 9.2 Avoid when
- Many elements with mixed roles: split into two statements for clarity.

### 9.3 Good versus bad

```csharp
// Good
(var head, rest) = Split(rest);

// Bad: hard to scan which variables are new
(var a, b, var c, d, _, var e) = Parse(line);
```

## 10. Related features

- [record-structs.md](record-structs.md): positional records generate `Deconstruct`.

## 11. References

- [What's new in C# 10](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-10#assignment-and-declaration-in-same-deconstruction)
- [Deconstructing tuples and other types](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/functional/deconstruct)
