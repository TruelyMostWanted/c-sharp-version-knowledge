---
id: cs14-null-conditional-assignment
csharp: 14
tfm: net10.0
status: stable
tags: [null-conditional, assignment, ?.]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-14
---
# Null-conditional assignment (C# 14)

`?.` and `?[]` may appear on the **left** side of an assignment or compound assignment.

## Syntax
```csharp
customer?.Order = GetCurrentOrder();
counter?.Value += 1;
lookup?["key"] = value;
```

## Rules
- The right side is evaluated **only** if the left side is not null (`GetCurrentOrder()` is not called when `customer` is null).
- Compound assignments (`+=`, `-=`, ...) are supported.
- `++` and `--` are **not** allowed.

## Pitfalls
- Silent no-op when null. If null is a bug, keep an explicit check or throw.

## Godot
- A freed Godot object is **not** `null` on the C# side. `?.` does not protect against freed nodes;
  use `GodotObject.IsInstanceValid(node)` where nodes can be freed.
```csharp
_hud?.ScoreLabel.Text = score.ToString();   // only guards against null, not against QueueFree()
```
