---
id: cs14-null-conditional-assignment
csharp: 14
tfm: net10.0
status: stable
tags: [null-conditional, assignment, "?."]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/member-access-operators#null-conditional-operators--and-
modernizes: if (x is not null) x.P = v;
---
# Null-conditional assignment (C# 14)

> **Abstract.** The null-conditional operators `?.` and `?[]` may appear on the **left** side of an assignment or
> compound assignment: `customer?.Order = GetCurrentOrder();`. If the receiver is `null`, nothing happens and the
> right-hand side is **not evaluated**. `++` and `--` are not supported.

| | |
|---|---|
| Introduced | C# 14 · .NET 10 (`net10.0`) |
| Status | Stable |
| Kind | Syntax + semantics |
| Runtime support needed | No |
| Audience | Everyday code |
| Modernize category | PREFER (keep explicit checks where null is a bug) |

## 1. Motivation

`?.` could read (`x?.P`) and call (`x?.M()`), but not write. Assignments to optional objects required an `if`:

```csharp
if (customer is not null) customer.Order = GetCurrentOrder();
```

## 2. The feature explained

### 2.1 Syntax

```csharp
a?.B = value;
a?.B += value;
a?[i] = value;
a?.B?.C = value;
```

### 2.2 Semantics and rules

1. The receiver is evaluated once; if it is `null`, the assignment is skipped **including the right-hand side**.
2. Compound assignments (`+=`, `-=`, `*=`, `??=`, ...) are supported.
3. `++` / `--` (prefix or postfix) are **not** allowed on null-conditional targets.
4. Use it as a statement; relying on the value of the assignment expression hurts readability.

## 3. Code examples

### 3.1 Minimal example

```csharp
customer?.Order = GetCurrentOrder();   // GetCurrentOrder() is not called when customer is null
counter?.Value += 1;
lookup?["key"] = value;
```

### 3.2 Extended examples

Optional UI elements:

```csharp
_hud?.ScoreLabel.Text = score.ToString();
_statusBar?.Message = "Saved";
```

Chained optional targets:

```csharp
selection?.Target?.Highlight = true;
```

Event-like accumulators:

```csharp
_stats?.DamageDealt += hit.Amount;
_cache?[key] ??= Compute(key);
```

## 4. Before this feature

```csharp
if (customer != null) customer.Order = GetCurrentOrder();

var hud = _hud;
if (hud is not null) hud.ScoreLabel.Text = score.ToString();
```

## 5. Under the hood

```csharp
// customer?.Order = GetCurrentOrder();
var tmp = customer;
if (tmp is not null)
{
    tmp.Order = GetCurrentOrder();
}
```

The receiver is copied to a temporary so it is read once (thread-safe snapshot like other `?.` usages).

## 6. Performance and allocation

None beyond the null check.

## 7. Pitfalls

- **Silent no-op when null.** If null indicates a bug, keep an explicit check or throw; otherwise errors disappear.
- Side effects on the right side do not happen when the receiver is null (`x?.Log = NextId++` does not increment).
- Value-type receivers: `?.` requires a nullable or reference receiver.

## 8. Concerns and limitations

- Godot: a freed Godot object is **not** `null` on the C# side. `?.` does not protect against freed nodes;
  use `GodotObject.IsInstanceValid(node)` where nodes can be freed.

```csharp
_hud?.ScoreLabel.Text = score.ToString();   // only guards against null, not against QueueFree()

if (IsInstanceValid(_hud)) _hud.ScoreLabel.Text = score.ToString();   // safe for freed nodes
```

## 9. When to use: clean-code guidance

### 9.1 Use when
- The target is legitimately optional (not yet created, optional component, feature disabled).

### 9.2 Avoid when
- Null means a programming error: fail loudly with `ArgumentNullException.ThrowIfNull` or a guard.
- Godot nodes that can be freed.

### 9.3 Good versus bad

```csharp
// Good: optional debug overlay
_debugOverlay?.Text = $"FPS {Engine.GetFramesPerSecond()}";

// Bad: required dependency silently ignored
_saveService?.LastSlot = slot;   // if _saveService is null, the save slot is lost without a trace
```

## 10. Related features

- [../10/improved-definite-assignment.md](../10/improved-definite-assignment.md)
- [field-keyword.md](field-keyword.md)

## 11. References

- [Null-conditional operators](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/member-access-operators#null-conditional-operators--and-)
- [Proposal: Null-conditional assignment](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-14.0/null-conditional-assignment)
