---
id: cs11-auto-default-structs
csharp: 11
tfm: net7.0
status: stable
tags: [struct, constructor, definite-assignment]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-11
---
# Auto-default structs (C# 11)

A struct constructor no longer has to assign every field: the compiler initializes unassigned fields to their default value.

```csharp
public struct Particle
{
    public Vector2 Position;
    public Vector2 Velocity;
    public float Life;

    public Particle(Vector2 position) { Position = position; }   // Velocity and Life default to 0
}
```
