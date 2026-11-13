---
id: cs15-closed-hierarchies
csharp: 15
tfm: net11.0
status: stable
tags: [closed, sealed-hierarchy, pattern-matching, exhaustiveness, state-machine]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-15
---
# Closed hierarchies (C# 15)

The contextual modifier `closed` on a class means it can be derived from **only inside its declaring
assembly**. The compiler therefore knows every direct descendant, and a `switch` that handles each
of them is exhaustive without a default arm.

## Use when
- State machines (player, AI, building states).
- Domain events, commands, replay or undo logs.
- Any hierarchy you own where adding a variant should flag every unhandled `switch`.
- Use a `union` instead for unrelated or foreign types (see [union-types.md](union-types.md)).

## Syntax
```csharp
public closed record class GateState;
public record class Closed : GateState;
public record class Open(float Percent) : GateState;

string Describe(GateState state) => state switch
{
    Closed => "closed",
    Open(var percent) => $"{percent}% open",
    // no warning: every direct descendant is handled
};
```

## Rules
- A `closed` class is implicitly `abstract`.
- Cannot be combined with `sealed`, `static` or an explicit `abstract`.
- **Not transitive**: a non-closed descendant can still be derived from in other assemblies.
  Mark leaves `sealed` or intermediate levels `closed` to keep exhaustiveness further down.

## Pitfalls
- Do not add a discard arm `_ =>`; it hides the warning you want when a new variant is added.
- Each state or event is a heap object. Fine for few objects with rare changes; for thousands of
  entities per frame use enums or struct data instead.

## Example (game)
```csharp
public closed record PlayerState;
public sealed record Idle : PlayerState;
public sealed record Running(float Speed) : PlayerState;
public sealed record Airborne(float VerticalVelocity, bool DoubleJumpUsed) : PlayerState;
public sealed record Dead(float RespawnTimer) : PlayerState;

_state = _state switch
{
    Idle       => Input.IsActionJustPressed("jump") ? new Airborne(JumpForce, false) : _state,
    Running r  => UpdateRunning(r, delta),
    Airborne a => UpdateAir(a, delta),
    Dead d     => d.RespawnTimer <= 0 ? Respawn() : d with { RespawnTimer = d.RespawnTimer - (float)delta },
};
```
