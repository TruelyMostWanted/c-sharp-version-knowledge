---
id: cs15-closed-hierarchies
csharp: 15
tfm: net11.0
status: stable
tags: [closed, sealed-hierarchy, pattern-matching, exhaustiveness, state-machine]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-15
modernizes: abstract base plus switch with a `_ => throw` default arm
---
# Closed hierarchies (C# 15)

> **Abstract.** The contextual modifier `closed` on a class means it can be derived from **only inside its declaring
> assembly**. The compiler therefore knows every direct descendant, and a `switch` that handles each of them is
> **exhaustive without a default arm**. Adding a new variant then produces warnings at every `switch` that does not
> handle it, which is exactly what you want for state machines, events and commands.

| | |
|---|---|
| Introduced | C# 15 · .NET 11 (`net11.0`) |
| Status | Stable |
| Kind | Language (type system + pattern exhaustiveness) |
| Runtime support needed | No new runtime type for the language rule (check the docs for metadata details) |
| Audience | Everyday code with domain hierarchies |
| Modernize category | CASE |

## 1. Motivation

C# pattern matching could not know that an abstract class has a fixed set of subclasses; any assembly could add one.
`switch` expressions over such hierarchies needed a discard arm `_ => throw new UnreachableException()`, which also
silenced the compiler when someone added a new subclass and forgot to handle it. Functional languages solve this with
closed sum types; C# 15 offers two forms: `closed` class hierarchies and [unions](union-types.md).

## 2. The feature explained

### 2.1 Syntax

```csharp
public closed class Shape;                  // or closed record / closed record class
public sealed class Circle(float r) : Shape;
public sealed class Square(float side) : Shape;
```

### 2.2 Semantics and rules

1. A `closed` class can only be derived from within the same assembly.
2. A `closed` class is implicitly **abstract** (cannot be instantiated).
3. `closed` cannot be combined with `sealed`, `static` or an explicit `abstract`.
4. Exhaustiveness: a `switch` that handles every **direct** descendant (or every leaf of nested closed levels) needs no default arm.
5. **Not transitive:** a non-closed, non-sealed descendant can still be derived from in other assemblies. Mark leaves `sealed`
   or intermediate levels `closed` to keep exhaustiveness further down.
6. Works for `class` and `record class` hierarchies.
7. `null` is still a possible value of a reference-type variable; with nullable enabled the compiler may ask you to handle it.

## 3. Code examples

### 3.1 Minimal example

```csharp
public closed record class GateState;
public sealed record class Closed : GateState;
public sealed record class Open(float Percent) : GateState;

string Describe(GateState state) => state switch
{
    Closed => "closed",
    Open(var percent) => $"{percent}% open",
    // no warning: every direct descendant is handled
};
```

### 3.2 Extended examples

Player state machine:

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

Commands with an undo log:

```csharp
public closed record EditorCommand;
public sealed record PlaceTile(Vector2I Cell, int TileId) : EditorCommand;
public sealed record EraseTile(Vector2I Cell, int PreviousTileId) : EditorCommand;
public sealed record MoveSelection(Rect2I From, Vector2I Offset) : EditorCommand;

EditorCommand Invert(EditorCommand c) => c switch
{
    PlaceTile p     => new EraseTile(p.Cell, p.TileId),
    EraseTile e     => new PlaceTile(e.Cell, e.PreviousTileId),
    MoveSelection m => new MoveSelection(m.From with { Position = m.From.Position + m.Offset }, -m.Offset),
};
```

Nested closed levels:

```csharp
public closed record Message;
public closed record Request : Message;
public sealed record Ping : Request;
public sealed record Query(string Text) : Request;
public sealed record Response(string Body) : Message;

int Priority(Message m) => m switch
{
    Ping     => 0,
    Query    => 1,
    Response => 2,
};
```

Shared members on the base (an advantage over unions):

```csharp
public closed record GameEvent(int Tick)
{
    public bool IsLate(int now) => now - Tick > 5;
}
public sealed record UnitSpawned(int Tick, int UnitId) : GameEvent(Tick);
public sealed record UnitDied(int Tick, int UnitId) : GameEvent(Tick);
```

## 4. Before this feature

```csharp
public abstract record PlayerState;
public sealed record Idle : PlayerState;
...

_state = _state switch
{
    Idle => ...,
    Running r => ...,
    Airborne a => ...,
    Dead d => ...,
    _ => throw new UnreachableException(),   // hides missing cases when a new state is added
};
```

Or the visitor pattern (`Accept(IVisitor)`) to get compile-time completeness at the cost of boilerplate.

## 5. Under the hood

A `closed` class is emitted as an **abstract** class. The C# compiler records the closedness in metadata and refuses
derivation from other assemblies. A `switch` over a closed hierarchy lowers to the usual type tests (`isinst`) in
sequence; the compiler simply omits the "no arm matched" path it would otherwise need (it still emits a throw of
`SwitchExpressionException` for unexpected values such as `null`).

## 6. Performance and allocation

- Same as ordinary class hierarchies: each state/event is a heap object.
- Type switches cost a few type checks; ordering arms by frequency can help in hot code.
- For thousands of entities per frame, prefer enums or struct data (SoA/AoS); use closed hierarchies for few, rarely changing objects.

## 7. Pitfalls

- Do not add a discard arm `_ =>`; it hides the warning you want when a new variant is added.
- Non-sealed leaves weaken the guarantee (other assemblies can derive from them). Seal leaves.
- `closed` is a contextual keyword: an existing type named `closed` conflicts (see warning wave 7).

## 8. Concerns and limitations

- API evolution: adding a new case is a **breaking change** for consumers' exhaustive switches (they get warnings). That is the point, but plan public hierarchies accordingly.
- Serialization: polymorphic JSON still needs `[JsonDerivedType]` or a type discriminator.
- Godot: records and closed hierarchies are not Variant-compatible; keep them in the C# logic layer, not in `[Export]` or signals.

## 9. When to use: clean-code guidance

### 9.1 Use when
- State machines, domain events, commands, AST nodes, replay or undo logs.
- You own all variants and want shared base members.

### 9.2 Avoid when
- Variants are unrelated or foreign types: use a [union](union-types.md).
- Third parties must be able to add variants (plugins): keep the hierarchy open and use virtual methods.

### 9.3 Good versus bad

```csharp
// Good
public closed record LoadState;
public sealed record Loading(float Progress) : LoadState;
public sealed record Loaded(Scene Scene) : LoadState;
public sealed record Failed(string Error) : LoadState;

// Bad: closed but with a discard arm, losing the benefit
var text = state switch { Loading l => "...", _ => "done" };
```

## 10. Related features

- [union-types.md](union-types.md)
- [../10/record-sealed-tostring.md](../10/record-sealed-tostring.md)
- [../10/extended-property-patterns.md](../10/extended-property-patterns.md)

## 11. References

- [What's new in C# 15](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-15)
- [Pattern matching](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/functional/pattern-matching)
